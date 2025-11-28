# esa.io RAG Q&A System with Local LLM

esa.io の記事データを知識源（ナレッジベース）とし、ローカルLLM (`Llama-3-ELYZA-JP-8B`) を用いて回答を生成する社内用Q&Aシステムです。

## 🏗 システムの仕組み (Architecture)

このシステムは **Retrieval-Augmented Generation (RAG)** アーキテクチャを採用しており、以下のコンポーネントで構成されています。

1.  **Frontend (`frontend_streamlit.py`)**
    * Streamlitを使用したユーザーインターフェース。
    * ログイン認証、チャット画面、参照元ソースの表示を行います。
2.  **Backend (`main_fastapi.py`)**
    * FastAPIによるAPIサーバー。
    * ユーザーからの質問を受け取り、RAGロジック（検索＋生成）を実行します。
    * LLM (`elyza/Llama-3-ELYZA-JP-8B`) のロードと推論を担当します。
3.  **Data Connector (`esa_connector.py`)**
    * esa.io APIから記事を取得し、テキストのクリーニングと分割（チャンク化）を行います。
    * 環境変数 (`.env`) から設定を読み込みます。
    * SentenceTransformerとFAISSを使用してベクトルインデックスを作成・管理します。
4.  **Database (`database.py`)**
    * SQLiteを使用し、ユーザー情報とQ&Aの履歴ログを保存します。

## ⚠️ 前提条件 (Prerequisites)

このシステムは高性能なLLMをローカル環境で動作させるため、以下の環境を推奨します。

* **OS**: Linux, Windows (WSL2), または macOS (Apple Silicon)
* **GPU**: NVIDIA GPU (VRAM 12GB以上推奨) または Apple Silicon (M1/M2/M3)
    * *※CPUのみでも動作しますが、応答生成に非常に時間がかかります。*
* **Python**: 3.9 以上
* **Hugging Face アカウント**: 使用モデル `elyza/Llama-3-ELYZA-JP-8B` へのアクセス承認が必要です。

## 🚀 セットアップ手順 (Installation)

### 1. リポジトリのクローン

```
git clone https://github.com/TakaYama7/Kensakun.git
cd Kensakun
````

### 2\. ライブラリのインストール

`requirements.txt` (python-dotenv等を含む) を使用して、必要なパッケージを一括インストールします。

`
pip install -r requirements.txt
`

> **Note**: GPU環境での最適化や、モデルのロード設定（`device_map="auto"`）のために、環境によっては以下を追加でインストールする必要があります。
> `pip install accelerate bitsandbytes`

### 3\. Hugging Face へのログイン

`Llama-3-ELYZA-JP-8B` モデル（Gated Model）を使用するために、Hugging Faceの認証が必要です。

1.  Hugging Faceのモデルページで利用規約に同意してください。
2.  以下のコマンドでログインし、アクセストークン（Read権限）を入力します。

<!-- end list -->

```bash
# requirements.txtに含まれていない場合のみインストール
pip install huggingface_hub

huggingface-cli login
```


### 4\. 環境変数の設定 (重要)

APIキーなどの機密情報を管理するため、プロジェクトのルートディレクトリに `.env` ファイルを作成します。

1.  ルートディレクトリに `.env` という名前のファイルを作成します。
2.  以下の内容を記述し、ご自身のesaチーム情報とAPIトークンを入力してください。

<!-- end list -->

```env
# .env file
ESA_TEAM_NAME=your_team_subdomain
ESA_ACCESS_TOKEN=your_access_token_here
```

  * **ESA\_TEAM\_NAME**: `https://[team-name].esa.io` の `[team-name]` 部分のみ
  * **ESA\_ACCESS\_TOKEN**: esaの設定画面から発行したRead権限のあるトークン

> **注意**: `.env` ファイルには機密情報が含まれるため、GitHub等へアップロードしないよう `.gitignore` に追加されていることを確認してください。

### 5\. データベースの初期化

初回起動前にデータベースファイルを作成し、初期ユーザーを登録します。

```bash
python database.py
```

> 成功すると `Database initialized.` と表示され、`qa_logs.db` が作成されます。

-----

## 🏃‍♂️ 起動方法 (How to Run)

システムを動作させるには、**バックエンド**と**フロントエンド**をそれぞれ別のターミナル（コマンドプロンプト）で起動する必要があります。

### 手順 1: バックエンド (FastAPI) の起動

1つ目のターミナルで以下を実行します。初回はモデルのダウンロードとインデックス作成が行われるため、数分かかる場合があります。

```bash
uvicorn main_fastapi:app --reload
```

  * `✅ 生成モデル (...) のロード完了。` と表示されれば準備完了です。

### 手順 2: フロントエンド (Streamlit) の起動

2つ目のターミナルを開き、以下を実行します。

```bash
streamlit run frontend_streamlit.py
```

### 手順 3: 利用開始

ブラウザが自動的に立ち上がり、ログイン画面が表示されます。
（自動で開かない場合は `http://localhost:8501` にアクセスしてください）

**デモ用ログイン情報:**

  * **ユーザー名**: `testuser`
  * **パスワード**: `password123`

-----

## 📂 ファイル構成

| ファイル名              | 役割                                           |
| ----------------------- | ---------------------------------------------- |
| `frontend_streamlit.py` | ユーザー操作画面 (UI/UX)                       |
| `main_fastapi.py`       | APIサーバー、RAGロジック、LLM推論              |
| `esa_connector.py`      | esaデータ取得、ベクトル検索構築 (.env読み込み) |
| `database.py`           | データベース初期化、ユーザー・ログ管理         |
| `.env`                  | **(要作成)** APIキー等の環境変数設定ファイル   |
| `requirements.txt`      | 依存ライブラリ一覧                             |
| `qa_logs.db`            | 自動生成されるSQLiteデータベース               |
| `esa_data_cache.json`   | 自動生成される記事データのキャッシュ           |
| `esa_faiss_index.bin`   | 自動生成されるベクトルインデックスファイル     |

## 🛠 トラブルシューティング

  * **ValueError: 環境変数 ... が設定されていません**:
      * `.env` ファイルが存在しないか、変数名が間違っています。手順4を確認してください。
  * **CUDA Out of Memory エラー**:
      * VRAM不足です。`main_fastapi.py` のモデルロード部分で `load_in_8bit=True` または `load_in_4bit=True` (要 `bitsandbytes`) を設定して量子化を有効にしてください。
  * **esaの記事が更新されない**:
      * キャッシュが効いているためです。`esa_data_cache.json` と `esa_faiss_index.bin` を削除してバックエンドを再起動すると、最新データを再取得します。

<!-- end list -->