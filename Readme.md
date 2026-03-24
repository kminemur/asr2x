# ASR2X

Windows で動く、CLI 専用の音声文字起こしツールです。`OpenVINO/whisper-large-v3-int4-ov` を使ってマイク入力を文字起こしし、結果を標準出力またはテキストファイルへ出力します。

## 特徴

- GUI なしの CLI 専用実装
- 16kHz / モノラルでマイク録音
- `OpenVINO/whisper-large-v3-int4-ov` を自動ダウンロード
- OpenVINO デバイスは `NPU -> GPU -> CPU` の順で選択
- 端末操作で録音開始 / 停止
- 固定秒数録音 (`--duration`) に対応
- 文字起こし結果をファイル保存 (`--output`) 可能

## セットアップ

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
```

## 使い方

対話モード:

```powershell
python app.py
```

1. Enter で録音開始
2. もう一度 Enter で録音停止
3. 文字起こし結果が標準出力に表示される

固定秒数録音:

```powershell
python app.py --duration 5
```

ファイルへ保存:

```powershell
python app.py --duration 5 --output result.txt
```

OpenVINO デバイス指定:

```powershell
python app.py --device GPU
```

利用可能なデバイス確認:

```powershell
python app.py --list-openvino-devices
python app.py --list-audio-devices
```

## 動作仕様

- モデルは初回実行時に [`models/whisper-large-v3-int4-ov`](C:/Users/Intel/projects/asr2x/models/whisper-large-v3-int4-ov) へ保存されます
- モデルロード中や認識中の進捗は標準エラー出力へ表示します
- 文字起こし結果そのものは標準出力へ表示するため、リダイレクトやパイプで扱えます
- `--device` 未指定時は `NPU -> GPU -> CPU` の順で利用可能なデバイスを試します

## 注意

- Windows のマイク権限が必要です
- 初回はモデルのダウンロードとロードに時間がかかります
- Hugging Face からモデルを取得するためネットワーク接続が必要です
