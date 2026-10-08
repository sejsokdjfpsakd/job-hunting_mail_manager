# 就活メール自動振り分けシステム (Job-Hunting Mail Manager)

IMAP を利用してメールを自動的にフォルダ分けするシステムです。
AIエージェント（Claudeなど）から MCP (Model Context Protocol) 経由でメールの閲覧、ルールの作成・検証・実行を操作できるように設計されています。

## アーキテクチャ

システムは以下の2つのコンポーネントで構成されています。

1. **`rental_server/`**: Python CGI として動作するバックエンド API サーバー。常駐プロセスが利用できない安価なレンタルサーバー（さくらのレンタルサーバなど）での稼働を前提とし、JSONファイルベースで状態を管理します。
2. **`mcp_server/`**: Next.js ベースの MCP サーバー。AIエージェントからのリクエストを受け付け、認証を行った上で `rental_server` へ処理を委譲します。Vercel 等のサーバーレス環境でのホスティングを想定しています。

## テスト環境（ローカル）での実行方法

ローカル環境で開発や動作確認を行う手順です。

### 1. バックエンド API (rental_server) の起動
```bash
cd rental_server
# 仮想環境の作成と依存関係のインストール（初回のみ）
python -m venv .venv
.venv\Scripts\activate  # Windows の場合
pip install -r requirements.txt

# 開発用サーバーを起動
# ※開発時は dev_server.py がデフォルト設定を読み込みますが、
#   カスタム設定を行う場合や本番運用の場合は .env を作成してください (本番用テンプレート: .env.example)
python dev_server.py
```
> ローカルサーバーは通常 `http://localhost:8000` で立ち上がります。

### 2. MCP サーバー (mcp_server) の起動
```bash
cd mcp_server
# 依存関係のインストール（初回のみ）
npm install

# .env.local を作成して必要な環境変数を設定（初回のみ必須）
cp .env.local.example .env.local

# Next.js 開発サーバーの起動
npm run dev
```
> MCPサーバーは `http://localhost:3000/api/mcp` エンドポイントを提供します。

### 3. MCP Inspector でのテスト
MCP クライアントからの接続をテストするには、公式の Inspector を利用します。当プロジェクトは Streamable HTTP 方式（POST ベース）を採用しているため、`mcp-remote` アダプタを介して接続します。

別のターミナルを開き、以下を実行します：
```bash
npx @modelcontextprotocol/inspector npx -y mcp-remote http://localhost:3000/api/mcp
```
ブラウザで Inspector の UI が立ち上がり、利用可能なツール（`get_folders`, `create_rule`, `search_emails` など）をテストできます。

## 本番環境へのデプロイ
各ディレクトリの README を参照してください。
- `rental_server/README.md`: レンタルサーバーへの CGI デプロイ手順
- `mcp_server/README.md`: Vercel へのデプロイ手順

