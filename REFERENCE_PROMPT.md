# Reference Prompt For `app.py`

以下を満たす `app.py` を Python で実装してください。

## 目的

Windows で動作する CLI 専用の音声文字起こしアプリを作る。  
`onnx-community/whisper-large-v3-ONNX` を取得し、初回実行時に OpenVINO IR へ変換する。  
モデルを先にロードしてから、マイク入力をリアルタイムで音声認識できるようにする。

## 使用技術

- 言語: Python
- UI: なし
- 音声入力: `sounddevice`
- 数値処理: `numpy`
- モデル取得: `huggingface_hub.snapshot_download`
- OpenVINO 実行基盤: `openvino`
- ONNX -> OpenVINO IR 変換と推論: `optimum-intel[openvino]`
- 前処理 / 後処理: `transformers.AutoConfig`, `transformers.AutoProcessor`
- 使用モデル: `onnx-community/whisper-large-v3-ONNX`
- 変換済みモデル保存先: `models/whisper-large-v3-onnx-ov`
- 元の ONNX 保存先: `models/whisper-large-v3-onnx-ov/source`

## 基本要件

- CLI のみを実装する
- サンプリングレートは 16kHz
- モノラル録音
- 初回実行時に ONNX をダウンロードし、OpenVINO IR に変換する
- モデルは録音前に先にロードする
- `--realtime` でリアルタイム認識を実行する
- リアルタイム認識では短いチャンクごとに音声を取り出し、直近のローリングバッファを再認識する
- 差分テキストだけ標準出力へ逐次表示する
- `--realtime-chunk` で更新間隔を指定できる
- `--realtime-context` で認識に使うローリング文脈長を指定できる
- `--duration` 指定時は通常録音モードでもリアルタイムモードでも固定秒数で終了する
- `--audio-file` 指定時は 16kHz WAV ファイルを読み込んで文字起こしする
- `--output` 指定時はテキストファイルにも保存する
- OpenVINO デバイスは `NPU -> GPU -> CPU` の順で選択する
- `--device` 指定時はそのデバイスを優先する
- `--prepare-model-only` でモデル準備だけ実行して終了できるようにする
- `--force-reconvert` で既存 IR を再生成できるようにする
- `--list-openvino-devices` と `--list-audio-devices` を実装する
- 進捗メッセージは標準エラー出力へ表示する

## 実装詳細

- `MODEL_ID = "onnx-community/whisper-large-v3-ONNX"`
- `BASE_DIR = Path(__file__).resolve().parent`
- `MODEL_DIR = BASE_DIR / "models" / "whisper-large-v3-onnx-ov"`
- `SOURCE_DIR = MODEL_DIR / "source"`
- `SAMPLE_RATE = 16000`
- `CHANNELS = 1`
- `CONVERSION_MARKER = ".optimum_onnx_exported"`
- `openvino.Core().available_devices` を見て利用可能デバイスを確認する
- `snapshot_download(..., allow_patterns=[...])` で必要な ONNX / tokenizer / config だけ取得する
- 取得対象には少なくとも以下を含める:
  - `onnx/encoder_model.onnx`
  - `onnx/decoder_model.onnx`
  - `onnx/decoder_with_past_model.onnx`
  - `config.json`
  - `generation_config.json`
  - `preprocessor_config.json`
  - `tokenizer.json`
  - `tokenizer_config.json`
  - `special_tokens_map.json`
  - `merges.txt`
  - `vocab.json`
- OpenVINO IR 変換は `OVModelForSpeechSeq2Seq.from_pretrained(..., from_onnx=True, compile=False, encoder_file_name="onnx/encoder_model.onnx", decoder_file_name="onnx/decoder_model.onnx", decoder_with_past_file_name="onnx/decoder_with_past_model.onnx")` を使う
- 変換後は `model.save_pretrained(MODEL_DIR)` で IR を保存する
- `config.json`、`generation_config.json`、`preprocessor_config.json`、tokenizer 関連ファイルは `MODEL_DIR` に揃える
- 推論時は `OVModelForSpeechSeq2Seq.from_pretrained(MODEL_DIR, compile=False)` でロードする
- デバイス切り替えは `model.to(device)` の後に `model.compile()` を行う
- モデル先読み用に `warmup()` 相当の処理を用意し、録音開始前にロード完了させる
- 音声前処理とデコードには `AutoProcessor.from_pretrained(MODEL_DIR)` を使う
- 推論は `processor(audio=audio, sampling_rate=16000, return_tensors="pt")` の結果を `model.generate(..., task="transcribe", max_new_tokens=256)` に渡す
- 出力テキストは `processor.batch_decode(..., skip_special_tokens=True)` で取得する
- リアルタイム認識はチャンクを `deque` などに保持し、`--realtime-context` を超えた古い音声は捨てる
- 差分表示は前回テキストとの共通 prefix を比較して、新規部分だけを出す
- 録音データは `float32` の `numpy.ndarray` として扱う

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
- `build_parser() -> argparse.ArgumentParser`
- `record_interactive(recorder: Recorder) -> np.ndarray`
- `record_for_duration(recorder: Recorder, duration: float) -> np.ndarray`
- `load_wav_audio(path: Path) -> np.ndarray`
- `common_prefix_length(left: str, right: str) -> int`
- `run_realtime(...) -> int`
- `run(args: argparse.Namespace) -> int`
- `main()`

## ステータス / エラー

- モデルダウンロード中は `Downloading onnx-community/whisper-large-v3-ONNX ...` 相当のメッセージを出す
- モデル変換中は `Converting ONNX -> OpenVINO IR ...` 相当のメッセージを出す
- モデル先読み中は `Preloading model ...` を出す
- モデルロード中は `Loading model on {device} ...` を出す
- リアルタイム準備完了時は `Realtime recognition ready.` を出す
- リアルタイム待機中は `Listening ... Press Ctrl+C to stop.` を出す
- 認識中は `Recognizing ...` を出す
- 音声が録れていない場合は `No audio recorded.` を出して終了コード 1
- `Ctrl + C` 時は録音を止めて終了コード 130 もしくはリアルタイム停止メッセージを出して正常終了する
- 例外発生時は標準エラー出力へエラーメッセージを出して終了コード 1

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
