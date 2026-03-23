# Reference Prompt For `app.py`

以下の仕様を満たす `app.py` を Python で実装してください。

## 目的

Windows で動くシンプルなデスクトップ音声認識アプリを作る。  
マイクから録音した音声を `OpenVINO/whisper-large-v3-int4-ov` で文字起こしし、結果をクリップボードへコピーして別アプリへ貼り付けられるようにする。

## 必須要件

- 言語: Python
- GUI: Tkinter
- 音声入力: `sounddevice`
- 数値配列: `numpy`
- モデル取得: `huggingface_hub.snapshot_download`
- 推論: `openvino-genai`
- OpenVINO デバイス列挙: `openvino`
- 使用モデル: `OpenVINO/whisper-large-v3-int4-ov`
- モデル保存先: `app.py` と同じディレクトリ配下の `models/whisper-large-v3-int4-ov`
- サンプリングレート: 16kHz
- モノラル録音
- 録音開始/停止ボタンを持つ
- 認識結果を複数行で表示する
- 認識結果をクリップボードへコピーできる
- 認識完了後に自動コピーするオプションを持つ
- UI が固まらないように推論処理はバックグラウンドスレッドで実行する

## 実装方針

- `MODEL_ID = "OpenVINO/whisper-large-v3-int4-ov"`
- `BASE_DIR = Path(__file__).resolve().parent`
- `MODEL_DIR = BASE_DIR / "models" / "whisper-large-v3-int4-ov"`
- `SAMPLE_RATE = 16000`
- `CHANNELS = 1`
- `openvino.Core().available_devices` を見てデバイスを自動選択する
- 優先順は `NPU -> GPU -> CPU`
- 選択したデバイスで `WhisperPipeline(str(MODEL_DIR), device)` を生成する
- モデルが存在しない場合は `snapshot_download(repo_id=MODEL_ID, local_dir=str(MODEL_DIR))` を使って保存する
- 推論設定は `WhisperGenerationConfig(task="transcribe", max_new_tokens=256)` を使う
- 推論呼び出しは `pipeline.generate(audio.tolist(), config)` を使う
- 結果は `result.texts[0]` を優先して取得する
- 録音データは `float32` の `numpy.ndarray` として扱う
- GUI スレッドとワーカースレッド間の通知には `queue.Queue` を使う

## 望ましいクラス構成

- `Recorder`
  - `start()`
  - `stop() -> np.ndarray`
  - `is_recording`
  - `sounddevice.InputStream` を内部で使う
- `WhisperService`
  - `device` を保持する
  - `_ensure_model() -> Path`
  - `_ensure_pipeline() -> WhisperPipeline`
  - `transcribe(audio: np.ndarray) -> str`
- `App`
  - Tkinter UI 構築
  - 録音開始/停止
  - バックグラウンドでの文字起こし
  - ステータス更新
  - クリップボードコピー
  - ビジー状態の管理

## UI 要件

- タイトル表示
- 説明文表示
- 説明文には現在選択された推論デバイス名を含める
- `Start Recording` と `Stop Recording` を同じボタンで切り替える
- `Copy Text` ボタン
- `Auto copy after recognition` チェックボックス
- 経過時間表示
- 認識結果表示用の複数行 `Text`
- ステータスメッセージ表示

## ステータスと挙動

- 録音中は録音経過時間を表示する
- ダウンロード中は `Downloading model...` を表示する
- モデルロード中は `Loading model on {device}... first run can take a few minutes` を表示する
- 認識中は `Recognizing...` を表示する
- ビジー状態では録音ボタンとコピーボタンを無効化する
- ビジー中は経過時間とドット付きステータス更新を続ける

## 例外処理

- マイク開始失敗時は `messagebox.showerror` を出す
- 録音データが空なら認識しない
- 推論失敗時は `traceback.format_exc()` をダイアログ表示する
- コピー対象テキストが空なら `Nothing to copy` をステータスに表示する

## 完成条件

- `python app.py` で起動できる
- 初回はモデルを `models/whisper-large-v3-int4-ov` に保存する
- NPU があれば NPU、なければ GPU、なければ CPU を使う
- 録音停止後に文字起こしが走る
- 認識結果をクリップボード経由で他アプリへ貼り付けられる
