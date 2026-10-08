---
description: "Job-Hunting Mail Manager のアーキテクチャ制約とコードのベストプラクティス"
---

# Project Architecture & Constraints

このプロジェクトは、就活メール自動振り分けシステムをAIエージェント（MCPクライアント）から操作できるようにするためのシステムです。
今後の機能追加やバグ修正を行う際は、以下のアーキテクチャ制約とルールを遵守してください。

## 1. 2層アーキテクチャの制約

- **rental_server (バックエンド API)**
  - **環境制約**: 常駐プロセス (Daemon) は使用不可。安価なレンタルサーバーで動かすため、呼び出しごとに起動・終了する **CGI (ステートレス)** として設計されています。
  - **状態管理**: データベースの代わりに JSON ファイル (`data/` ディレクトリ内) を使用します。並行アクセス対策のため、ファイル書き込み時は必ず `storage.py` にあるファイルロック機能を利用してください。
  - **ライブラリ**: `pip install` が利用できない環境を想定し、サードパーティライブラリは `packages/` ディレクトリにバンドルして `sys.path` を追加する方式をとっています。

- **mcp_server (AI用プロキシ/UI)**
  - **環境制約**: Vercel 等のサーバーレス環境で稼働する Next.js アプリケーション。
  - **通信方式**: 長時間の接続が必要な SSE (Server-Sent Events) ではなく、`mcp-handler` パッケージを用いた **Streamable HTTP** 方式 (POST ベース) を使用しています。
  - **役割**: クライアントからの認証処理と MCP リクエストのルーティングに専念し、実際のIMAP処理やルール管理はすべて `rental_server` に委譲します。

## 2. MCP JSON Schema の厳密な型定義

MCP クライアント（Claude 等）にツールを提供する際、Zod を用いて `inputSchema` を定義しています。互換性維持のため、以下の点に注意してください。

- **`z.any()` を使用しないでください**。JSON Schema に変換された際、空のスキーマ `{}` (bare `true`) となり、一部の厳格な MCP クライアントで警告やエラーを引き起こします。
- 任意の型が必要な場合でも、可能な限り `z.union([z.string(), z.number(), z.boolean(), z.array(z.string())])` のように具体的な型の組み合わせを明示してください。
- **`z.object().passthrough()` の使用は避けてください**。`additionalProperties: {}` に変換され、上記と同様の警告を引き起こします。未定義のプロパティを通すのではなく、必要なプロパティをすべて明示的に定義するアプローチをとってください。

## 3. テストとデバッグ

- MCP Inspector を用いたテストを行う場合、`/api/mcp` は POST を期待するため、URL 直打ちではなく `mcp-remote` アダプタを利用することが推奨されます。
  - コマンド例: `npx @modelcontextprotocol/inspector npx -y mcp-remote http://localhost:3000/api/mcp`
- ローカル検証時には `.env.local` に `DEV_BYPASS_AUTH=true` を設定することで JWT 認証をバイパスできます。
