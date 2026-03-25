# Reference Prompt For `app.py`

以下を満たす `app.py` を Python で実装してください。

## 目的

Windows で動作する CLI 専用の音声文字起こしアプリを作る。  
マイクから録音した音声を `OpenVINO/distil-whisper-large-v3-int8-ov` をそのまま用いて GPU で推論し、結果を標準出力へ表示する。

## 使用技術

- 言語: Python
- UI: なし
- 音声入力: `sounddevice`
- 数値処理: `numpy`
- モデル取得: `huggingface_hub.snapshot_download`
- 推論: `openvino-genai`
- OpenVINO デバイス確認: `openvino`
- 使用モデル: `OpenVINO/distil-whisper-large-v3-int8-ov`
- モデル保存先: `app.py` と同じディレクトリ配下の `models/distil-whisper-large-v3-int8-ov`
- 推論デバイス: `GPU` 固定

## 基本要件

- CLI のみを実装する
- サンプリングレートは 16kHz
- モノラル録音
- 対話モードでは Enter で録音開始し、再度 Enter で録音停止する
- `--duration` 指定時は固定秒数だけ録音する
- 認識結果は標準出力へ表示する
- `--output` 指定時はテキストファイルにも保存する
- OpenVINO 推論は GPU で実行する
- GPU が使えない場合はエラー終了する
- `--list-openvino-devices` と `--list-audio-devices` を実装する
- モデルが未配置なら自動ダウンロードする
- 進捗メッセージは標準エラー出力へ表示する

## 実装詳細

- `MODEL_ID = "OpenVINO/distil-whisper-large-v3-int8-ov"`
- `BASE_DIR = Path(__file__).resolve().parent`
- `MODEL_DIR = BASE_DIR / "models" / "distil-whisper-large-v3-int8-ov"`
- `SAMPLE_RATE = 16000`
- `CHANNELS = 1`
- `DEVICE = "GPU"`
- `openvino.Core().available_devices` を見て GPU が利用可能か確認する
- モデル未配置時は `snapshot_download(repo_id=MODEL_ID, local_dir=str(MODEL_DIR))` を使って保存する
- 推論時は `WhisperPipeline(str(MODEL_DIR), "GPU")` を使う
- 認識設定は `WhisperGenerationConfig(task="transcribe", max_new_tokens=256)` を使う
- 推論は `pipeline.generate(audio.tolist(), config)` を使う
- 結果は `result.texts[0]` を優先して取得する
- 録音データは `float32` の `numpy.ndarray` として扱う

## 想定クラス / 関数

- `Recorder`
  - `start()`
  - `stop() -> np.ndarray`
  - `is_recording`
- `WhisperService`
  - `available_devices() -> list[str]`
  - `_ensure_model() -> Path`
  - `_ensure_pipeline() -> WhisperPipeline`
  - `transcribe(audio: np.ndarray) -> str`
- `build_parser() -> argparse.ArgumentParser`
- `record_interactive(recorder: Recorder) -> np.ndarray`
- `record_for_duration(recorder: Recorder, duration: float) -> np.ndarray`
- `run(args: argparse.Namespace) -> int`
- `main()`

## ステータス / エラー

- モデルダウンロード中は `Downloading model ...` 相当のメッセージを出す
- モデルロード中は `Loading model on GPU ...` を出す
- 認識中は `Recognizing ...` を出す
- GPU が使えない場合はエラー終了する
- 音声が録れていない場合は `No audio recorded.` を出して終了コード 1
- `Ctrl + C` 時は録音を止めて終了コード 130
- 例外発生時は標準エラー出力へエラーメッセージを出して終了コード 1

## 実行例

```powershell
python app.py
python app.py --duration 5
python app.py --duration 5 --output result.txt
python app.py --list-openvino-devices
python app.py --list-audio-devices
```
