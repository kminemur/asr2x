# ASR2X

`onnx-community/whisper-large-v3-ONNX` を初回実行時にダウンロードし、OpenVINO IR に変換して使う CLI 音声認識アプリです。変換後は `openvino_genai.WhisperPipeline` で推論します。

## 特徴

- Hugging Face の `onnx-community/whisper-large-v3-ONNX` を利用
- ONNX から OpenVINO IR (`.xml` / `.bin`) を自動生成
- CLI だけで録音と文字起こしを実行
- 16kHz モノラルのマイク入力に対応
- 16kHz WAV ファイルの文字起こしにも対応
- OpenVINO デバイスは `NPU -> GPU -> CPU` の順で選択
- 初回変換後はローカルの IR を再利用

## セットアップ

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## 使い方

初回にモデルだけ準備:

```powershell
python app.py --prepare-model-only
```

マイクを対話モードで使う:

```powershell
python app.py
```

固定秒数だけ録音:

```powershell
python app.py --duration 5
```

WAV ファイルを文字起こし:

```powershell
python app.py --audio-file sample.wav
```

結果をファイルへ保存:

```powershell
python app.py --duration 5 --output result.txt
```

OpenVINO デバイスを明示:

```powershell
python app.py --device GPU
```

利用可能デバイスの確認:

```powershell
python app.py --list-openvino-devices
python app.py --list-audio-devices
```

再変換:

```powershell
python app.py --prepare-model-only --force-reconvert
```

## 出力先

- 変換済み OpenVINO モデル: `models/whisper-large-v3-onnx-ov`
- ダウンロードした ONNX 元データ: `models/whisper-large-v3-onnx-ov/source`

## 注意

- 初回のダウンロードと変換はかなり時間とディスク容量を使います
- `--audio-file` は 16kHz WAV を前提にしています
- マイク入力には Windows のマイク権限が必要です
