# ASR2X

`OpenVINO/whisper-large-v3-int4-ov` を使って音声を文字起こしし、結果をクリップボードへコピーできる Windows デスクトップアプリです。

## 概要

- マイクから 16kHz / モノラルで録音
- `OpenVINO/whisper-large-v3-int4-ov` で文字起こし
- 認識結果を画面に表示
- 認識結果をクリップボードへコピー
- 自動コピー有効時は認識完了後にそのままコピー
- 他アプリへ `Ctrl + V` で貼り付け可能

## モデル保存先

モデルは `app.py` と同じディレクトリ配下の `models/whisper-large-v3-int4-ov` に保存されます。

## デバイス選択

OpenVINO の利用可能デバイスを確認し、以下の優先順で推論デバイスを自動選択します。

1. `NPU`
2. `GPU`
3. `CPU`

NPU があれば NPU、なければ GPU、どちらも使えなければ CPU で動作します。

## セットアップ

```powershell
python -m venv .venv
.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r requirements.txt
python app.py
```

初回起動時または初回認識時に、Hugging Face からモデルを自動ダウンロードします。

## 使い方

1. `Start Recording` を押して話す
2. `Stop Recording` を押して認識を開始する
3. 認識完了後、必要に応じて `Copy Text` を押す
4. 他アプリで `Ctrl + V` を押して貼り付ける

`Auto copy after recognition` が有効なら、認識完了後に自動でクリップボードへコピーされます。

## UI の挙動

- 録音中は録音経過時間を表示します
- モデルダウンロード中、モデルロード中、認識中はステータスを表示します
- 重い処理の間は録音ボタンとコピーボタンを一時的に無効化します
- 初回のモデルロードは数十秒から数分かかることがあります

## 注意

- Windows のマイク権限が必要です
- large-v3 モデルはサイズが大きく、初回ダウンロードと初回ロードに時間がかかります
- Hugging Face を未認証で使うとレート制限が低くなることがあります
