# Reference Prompt For `app.py`

以下の要件を満たす `app.py` を Python で実装してください。

## 目的

Windows 上で動くシンプルなデスクトップ音声認識アプリを作る。  
マイクから録音した音声を `OpenVINO/whisper-large-v3-int4-ov` で文字起こしし、結果をクリップボードへコピーして、別アプリに貼り付けられるようにする。

## 必須要件

- 言語: Python
- GUI: Tkinter
- 音声入力: `sounddevice`
- 推論: `openvino-genai`
- モデル取得: `huggingface_hub.snapshot_download`
- 使用モデル: `OpenVINO/whisper-large-v3-int4-ov`
- モデル保存先: `app.py` と同じディレクトリ配下の `models/whisper-large-v3-int4-ov`
- サンプリングレート: 16kHz
- モノラル録音
- 録音開始/停止ボタンを持つ
- 認識結果を画面に表示する
- 認識結果をクリップボードへコピーできる
- 認識完了後に自動コピーするオプションを持つ
- 初回起動時にモデルがなければ自動ダウンロードする
- UI が固まらないように、認識処理はバックグラウンドスレッドで行う

## 実装方針

- `MODEL_ID = "OpenVINO/whisper-large-v3-int4-ov"`
- `BASE_DIR = Path(__file__).resolve().parent`
- `MODEL_DIR = BASE_DIR / "models" / "whisper-large-v3-int4-ov"`
- `WhisperPipeline(str(MODEL_DIR), "AUTO")` でロードする
- モデル未配置なら `snapshot_download(repo_id=MODEL_ID, local_dir=str(MODEL_DIR), local_dir_use_symlinks=False)` を使う
- `WhisperGenerationConfig(task="transcribe", max_new_tokens=256)` を使う
- `pipeline.generate(audio.tolist(), config)` で推論する
- 録音データは `float32` の `numpy.ndarray` として扱う
- GUI スレッドとワーカースレッドの受け渡しには `queue.Queue` を使う

## 推奨構成

- `Recorder` クラス
  - `start()`
  - `stop() -> np.ndarray`
  - `is_recording`
- `WhisperService` クラス
  - `_ensure_model() -> Path`
  - `_ensure_pipeline() -> WhisperPipeline`
  - `transcribe(audio: np.ndarray) -> str`
- `App` クラス
  - Tkinter UI の構築
  - 録音開始/停止
  - バックグラウンドでの文字起こし
  - 結果表示
  - クリップボードコピー

## UI 要件

- タイトル表示
- 説明文表示
- `Start Recording` / `Stop Recording` の切り替え
- `Copy Text` ボタン
- `Auto copy after recognition` チェックボックス
- 録音時間表示
- 認識結果の複数行表示
- ステータスメッセージ表示

## 例外処理

- マイク開始失敗時はダイアログを出す
- 録音データが空なら認識しない
- 推論失敗時はトレースバックをダイアログ表示する
- コピー対象が空ならその旨をステータスに出す

## 完成条件

- `python app.py` で起動できる
- 初回はモデルを `models/` にダウンロードする
- 録音停止後に文字起こしが走る
- 認識結果を `Ctrl + V` で他アプリへ貼り付けられる
