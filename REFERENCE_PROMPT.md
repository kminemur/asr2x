# Reference Prompt For `app.py`

以下を満たす `app.py` を Python で実装してください。

## 目的

Windows で動作する CLI 専用の音声文字起こしアプリを作る。  
`onnx-community/whisper-large-v3-ONNX` を初回実行時にダウンロードし、OpenVINO IR へ変換してから使う。  
モデルは録音前に先にロードできるようにし、マイク入力のリアルタイム認識にも対応させる。

## 使用技術

- 言語: Python
- UI: なし
- 音声入力: `sounddevice`
- 数値処理: `numpy`
- モデル取得: `huggingface_hub.snapshot_download`
- OpenVINO デバイス確認: `openvino`
- ONNX -> OpenVINO IR 変換と推論: `optimum.intel.openvino.OVModelForSpeechSeq2Seq`
- 前処理 / 後処理: `transformers.AutoConfig`, `transformers.AutoProcessor`

## モデルと保存先

- `MODEL_ID = "onnx-community/whisper-large-v3-ONNX"`
- `BASE_DIR = Path(__file__).resolve().parent`
- `MODEL_DIR = BASE_DIR / "models" / "whisper-large-v3-onnx-ov"`
- `SOURCE_DIR = MODEL_DIR / "source"`
- `CONVERSION_MARKER = ".optimum_onnx_exported"`
- 変換済み OpenVINO IR は `MODEL_DIR` に保存する
- 元の ONNX と tokenizer / config は `SOURCE_DIR` に保存する

## 基本定数

- `SAMPLE_RATE = 16000`
- `CHANNELS = 1`
- `MODEL_FILENAMES = {"encoder": "openvino_encoder_model.xml", "decoder": "openvino_decoder_model.xml", "decoder_with_past": "openvino_decoder_with_past_model.xml"}`

## ダウンロード対象

`snapshot_download(..., allow_patterns=SOURCE_PATTERNS)` を使い、少なくとも以下を取得する。

- `config.json`
- `generation_config.json`
- `preprocessor_config.json`
- `special_tokens_map.json`
- `tokenizer.json`
- `tokenizer_config.json`
- `added_tokens.json`
- `merges.txt`
- `vocab.json`
- `onnx/encoder_model.onnx`
- `onnx/encoder_model.onnx_data`
- `onnx/decoder_model.onnx`
- `onnx/decoder_model.onnx_data`
- `onnx/decoder_with_past_model.onnx`
- `onnx/decoder_with_past_model.onnx_data`

## OpenVINO IR 変換

- `AutoConfig.from_pretrained(str(SOURCE_DIR))` で config を読む
- `OVModelForSpeechSeq2Seq.from_pretrained(..., from_onnx=True, compile=False)` を使って ONNX から読み込む
- ファイル名は以下を明示する
  - `encoder_file_name="onnx/encoder_model.onnx"`
  - `decoder_file_name="onnx/decoder_model.onnx"`
  - `decoder_with_past_file_name="onnx/decoder_with_past_model.onnx"`
- 変換後は `model.save_pretrained(MODEL_DIR)` を呼ぶ
- 変換完了の印として `MODEL_DIR / ".optimum_onnx_exported"` に `MODEL_ID` を書く
- `config.json`、`generation_config.json`、`preprocessor_config.json`、tokenizer 関連ファイルは `SOURCE_DIR` から `MODEL_DIR` にコピーする

## デバイス選択

- `openvino.Core().available_devices` で OpenVINO デバイスを確認する
- デフォルト優先順は `GPU -> CPU`
- `--device` には `auto`, `GPU`, `CPU` を受け付ける
- `preferred_device` が指定され、かつ利用可能ならそのデバイスを先頭にする
- 指定デバイスが利用不可なら例外を出す

## 録音仕様

- `sounddevice.InputStream` を使う
- `dtype="float32"`
- `samplerate=16000`
- `channels=1`
- コールバックで受け取った `indata.copy()` を内部バッファへ積む
- `Recorder.start()` は録音開始だけ行う
- `Recorder.stop()` は stream を止めて閉じ、内部バッファを `np.ndarray[float32]` にして返す
- `Recorder.pop_audio()` は途中バッファを取り出して空にする

## WAV 読み込み

- `wave.open()` で読む
- 16kHz 以外はエラー
- 1 byte / 2 byte / 4 byte PCM をサポートする
- ステレオ以上は平均してモノラル化する
- 戻り値は contiguous な `float32` 配列にする

## 推論仕様

- 推論時は `OVModelForSpeechSeq2Seq.from_pretrained(str(MODEL_DIR), compile=False)` でロードする
- `model.to(device)` の後に `model.compile()` を呼ぶ
- `AutoProcessor.from_pretrained(str(MODEL_DIR))` を使う
- 推論時は `processor(audio=audio, sampling_rate=SAMPLE_RATE, return_tensors="pt")` を呼ぶ
- `model.generate(**inputs, task="transcribe", max_new_tokens=256)` を使う
- `processor.batch_decode(generated_ids, skip_special_tokens=True)` でテキスト化する
- テキストがあれば先頭要素を `strip()` して返す

## モデル先読み

- `WhisperService.warmup()` を用意し、内部で `_ensure_model_runtime()` を呼ぶ
- 通常録音モードでも録音開始前に `warmup()` を呼ぶ
- リアルタイムモードでも録音開始前に `warmup()` を呼ぶ

## リアルタイム認識

- `--realtime` で有効化する
- `--realtime-chunk` のデフォルトは `2.0`
- `--realtime-context` のデフォルトは `8.0`
- `run_realtime(...)` を用意する
- 録音開始後、`chunk_seconds` ごとに `Recorder.pop_audio()` で最新チャンクを取得する
- `deque[np.ndarray]` にチャンクを積み、`context_seconds * SAMPLE_RATE` を超えた古い音声は捨てる
- ローリングバッファ全体を結合して毎回再認識する
- 前回の認識結果と今回の認識結果の共通 prefix を `common_prefix_length()` で求める
- 新しい差分だけ標準出力へ出す
- `--output` 指定時は差分ごとに追記保存する
- `Ctrl + C` 時は `Stopped realtime recognition.` を標準エラーへ出す

## 通常モード

- 対話モードでは `Enter` で録音開始、再度 `Enter` で録音停止
- `--duration` 指定時はその秒数だけ録音する
- 録音完了後に全文を 1 回認識して標準出力へ出す
- `--output` 指定時は全文をファイルへ保存する

## CLI オプション

- `--duration`
- `--audio-file`
- `--audio-device`
- `--device`
- `--model-dir`
- `--output`
- `--realtime`
- `--realtime-chunk`
- `--realtime-context`
- `--prepare-model-only`
- `--force-reconvert`
- `--list-openvino-devices`
- `--list-audio-devices`

## 引数制約

- `--audio-file` と `--duration` は同時使用不可
- `--realtime` と `--audio-file` は同時使用不可
- `--realtime-chunk <= 0` はエラー
- `--realtime-context < --realtime-chunk` はエラー
- `--duration <= 0` はエラー

## ステータスとエラー

- 進捗メッセージは標準エラーへ出す
- 使用するメッセージは少なくとも以下を含める
  - `Downloading {MODEL_ID} to {SOURCE_DIR} ...`
  - `Converting ONNX -> OpenVINO IR with Optimum Intel ...`
  - `Loading model on {device} ...`
  - `Recognizing ...`
  - `Preloading model ...`
  - `Realtime recognition ready.`
  - `Listening ... Press Ctrl+C to stop.`
  - `Recording ... Press Enter to stop.`
  - `Recording stopped.`
  - `No audio recorded.`
- `KeyboardInterrupt` で通常モードを止めたら終了コード `130`
- 例外時は `Error: {exc}` を標準エラーへ出して終了コード `1`

## 想定クラス / 関数

- `Recorder`
  - `start()`
  - `stop() -> np.ndarray`
  - `pop_audio() -> np.ndarray`
  - `is_recording`
- `WhisperService`
  - `available_devices() -> list[str]`
  - `ensure_model(force_reconvert: bool = False) -> Path`
  - `warmup() -> None`
  - `transcribe(audio: np.ndarray) -> str`
- `print_status(message: str) -> None`
- `parse_audio_device(device: str | None) -> int | str | None`
- `list_openvino_devices() -> int`
- `list_audio_devices() -> int`
- `record_interactive(recorder: Recorder) -> np.ndarray`
- `record_for_duration(recorder: Recorder, duration: float) -> np.ndarray`
- `load_wav_audio(path: Path) -> np.ndarray`
- `write_output(path: Path, text: str) -> None`
- `common_prefix_length(left: str, right: str) -> int`
- `append_output_line(path: Path, text: str) -> None`
- `run_realtime(...) -> int`
- `build_parser() -> argparse.ArgumentParser`
- `run(args: argparse.Namespace) -> int`
- `main() -> None`

## 実行例

```powershell
python app.py --prepare-model-only
python app.py --prepare-model-only --force-reconvert
python app.py --realtime
python app.py --realtime --duration 10
python app.py --realtime --realtime-chunk 1.5 --realtime-context 6
python app.py
python app.py --duration 5
python app.py --audio-file sample.wav
python app.py --realtime --output result.txt
python app.py --device GPU
python app.py --list-openvino-devices
python app.py --list-audio-devices
```
