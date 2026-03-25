# ASR2X

`onnx-community/whisper-large-v3-ONNX` を初回実行時に取得し、OpenVINO IR に変換して使う CLI 音声認識アプリです。推論は Optimum Intel の `OVModelForSpeechSeq2Seq` を使います。

## 特徴

- `onnx-community/whisper-large-v3-ONNX` を利用
- 初回実行時に ONNX から OpenVINO IR (`.xml` / `.bin`) を生成
- モデルを先にロードしてから録音を始める
- マイク入力のリアルタイム認識に対応
- 16kHz WAV ファイルの文字起こしにも対応
- OpenVINO デバイスは `NPU -> GPU -> CPU` の順で選択

## セットアップ

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## 使い方

モデルだけ先に準備:

```powershell
python app.py --prepare-model-only
```

リアルタイム認識:

```powershell
python app.py --realtime
```

10 秒だけリアルタイム認識:

```powershell
python app.py --realtime --duration 10
```

リアルタイム認識の更新間隔を変える:

```powershell
python app.py --realtime --realtime-chunk 1.5 --realtime-context 6
```

通常の録音後認識:

```powershell
python app.py
python app.py --duration 5
```

WAV ファイルを文字起こし:

```powershell
python app.py --audio-file sample.wav
```

結果をファイルへ保存:

```powershell
python app.py --realtime --output result.txt
```

デバイス確認:

```powershell
python app.py --list-openvino-devices
python app.py --list-audio-devices
```

## メモ

- リアルタイム認識は短いチャンクを繰り返し再認識する方式です
- 初回のダウンロードと変換は時間とディスク容量を使います
- `--audio-file` は 16kHz WAV を前提にしています
