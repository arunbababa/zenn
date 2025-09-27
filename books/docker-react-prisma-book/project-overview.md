---
title: "プロジェクト全体像"
---

# プロジェクト全体像

## ディレクトリ構造（本記事で触る範囲）

今回の手順で操作する主なファイルに絞り、実際のプロジェクト順に並べると次のようになります。

```text
├── .env
├── Dockercompose.yml
├── Dockerfile
├── package.json
├── prisma/
│   ├── schema.prisma
│   └── seed.js
└── src/
    ├── API/
    │   └── index.ts
    ├── App.tsx
    └── server/
        └── server.js
```

本当はこんな感じ

![ディレクトリ構造](/images/dir.png)

このプロジェクトのgithubもありますのでcloneしていただければ手元のマシンで再現できます。

https://github.com/

## コンテナ構成と役割

![ディレクトリ構造](/images/containerRel.png)

- `web` コンテナ: Vite の開発サーバーを 5173 番ポートで起動し、React でタスク一覧を描画します。コードはホストのワークスペースをボリューム共有してホットリロードが効く構成です。
- `api` コンテナ: Express サーバー（`server.js`）を起動し、Prisma で PostgreSQL に接続してタスクの CRUD API を提供します。`DATABASE_URL` は Compose で `db` コンテナを参照するように設定されています。
- `db` コンテナ: PostgreSQL の公式イメージを利用し、タスクデータを永続化します。`db-data` ボリュームでデータディレクトリを保持し、再起動しても内容が消えないようにしています。

Compose ファイルでは `api` が `db` のヘルスチェック完了を待ってから起動し、`web` は `api` に依存するように設定。これにより、API や DB の準備が整う前にフロントが接続エラーを出す問題を避けています。

## 主なファイルの役割

ディレクトリ構造再掲

```text
├── .env
├── Dockercompose.yml
├── Dockerfile
├── package.json
├── prisma/
│   ├── schema.prisma
│   └── seed.js
└── src/
    ├── API/
    │   └── index.ts
    ├── App.tsx
    └── server/
        └── server.js
```

- `.env`: 環境変数の設定ファイル
- `Dockerfile`: Node.js ベースイメージを元に `/code` ディレクトリへアプリ一式をコピーし、web/api 両方のコンテナで共通して使うベースを定義します。
- `compose.yml`: `web`・`api`・`db` の 3 サービスと `db-data` ボリュームを定義。ポート公開、環境変数、依存関係の指定など実行時の挙動をここでコントロールします。
- `server.js`: Express アプリのエントリーポイント。Prisma Client で `Task` テーブルを操作し、`GET /api/tasks` と `POST /api/tasks` を公開します。
- `prisma/schema.prisma`: モデル定義。タスクのタイトルや説明、作成日時などを持つ `Task` モデルを記述し、`prisma migrate` で DB に反映させます。
- `src/API/index.ts`: フロントエンドから API を叩くためのラッパー。開発環境では `http://localhost:3001` を、コンテナ内では `http://api:3001` をベース URL に選択しています。
- `src/App.tsx`: `getTasks` を使ってタスク一覧を取得し、React コンポーネントとして描画する UI の中核です。

この全体像を押さえておくと、どのファイルを編集するとどのコンテナに影響が出るのかをイメージしながら読み進められます。
