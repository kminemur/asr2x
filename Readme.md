# ASR2X

`OpenVINO/whisper-large-v3-int4-ov` を使って音声を文字起こしし、結果をクリップボードへコピーできる Windows デスクトップアプリです。

## できること

- マイクから録音
- OpenVINO Whisper で音声認識
- 認識後に自動でクリップボードへコピー
- 他アプリへ `Ctrl + V` で貼り付け

## セットアップ

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
python app.py
```

初回起動時に Hugging Face から `OpenVINO/whisper-large-v3-int4-ov` を自動で取得し、`models/whisper-large-v3-int4-ov` に保存します。

## 使い方

1. `Start Recording` を押して話す
2. `Stop Recording` を押して認識を待つ
3. 自動コピーが有効なら、そのまま他アプリで貼り付ける
4. 手動コピーしたい場合は `Copy Text` を押す

## 注意

- 初回は大きめのモデルをダウンロードするため時間とディスク容量が必要です
- Windows のマイク権限が必要です
