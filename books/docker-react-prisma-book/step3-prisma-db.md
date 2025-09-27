---
title: "ステップ3: PostgreSQL × Prisma 設定"
---

# ステップ3: PostgreSQL × Prisma 設定

続いてデータベースを用意し、Prisma 経由で扱えるように設定します。まずは必要なパッケージを追加しましょう。

```bash
npm install prisma @prisma/client --save-dev
```

次に Prisma の初期設定を行います。PostgreSQL を使いたいので、`--datasource-provider` オプションで指定します。

```bash
npx prisma init --datasource-provider postgresql
```

このコマンドでプロジェクト直下に `prisma/` ディレクトリが作成され、`prisma/schema.prisma` と `.env` が生成されます。スキーマファイルには `env("DATABASE_URL")` を通じて接続情報を参照する記述が追加されているので、`.env` に正しい接続文字列を書き込む必要があります。

PostgreSQL の接続文字列は以下の形式です。

```
postgresql://USER:PASSWORD@HOST:PORT/DATABASE
```

詳細は公式ドキュメントを参照してください。他データベースの際はどう接続するか等の情報もあります。
https://www.prisma.io/docs/orm/overview/databases/postgresql

Compose で立ち上げる DB コンテナと合わせるために、`.env` を次のように整えます。

```env
DATABASE_URL="postgresql://postgres:postgres@db:5432/react_develop"
```

- `USER` / `PASSWORD` / `DATABASE` は Compose の `db` サービスで設定する `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` と一致させます
- `HOST` は Compose 内でのサービス名（`db`）を指定することで、コンテナ間ネットワーク越しに接続できます
- `PORT` は PostgreSQL のデフォルト 5432 を使用します

よって`Dockercompose.yml` の `db` サービスは以下のように追記します。

```yaml
services:
  db:
    image: postgres
    platform: linux/amd64
    volumes:
      - db-data:/var/lib/postgresql/data
    ports:
      - "5432:5432"
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_DB=react_develop
      - POSTGRES_PASSWORD=postgres
    healthcheck:
      test: "psql -U postgres"
      interval: 5s
      timeout: 5s
      retries: 5
```

公式イメージを使い、永続化用のボリュームとヘルスチェックを設定しています。ヘルスチェックが成功すると `service_healthy` になり、後続の `api` サービスはそれを待ってから起動できるようにします（次のステップで活用）。

つづいて `schema.prisma` にタスクテーブルのモデルを定義します。初期生成されたテンプレートを次の内容に差し替えましょう。

```prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// 以下を差し替え

model Task {
  id          Int      @id @default(autoincrement())
  title       String
  description String?
  completed   Boolean  @default(false)
  createdAt   DateTime @default(now())
  updatedAt   DateTime @updatedAt
}
```

モデルを定義したら、実際のデータベースに反映します。

```bash
npx prisma db push
```

コマンドが成功すると `Task` テーブルが Postgres に作成され、同時にPrisma Client が生成されます。このPrismaa Clientを、後続のexpressの環境で利用しフロントとデータベースの連携に利用します。

最後に、DB コンテナが正しく起動し続けているか確認します。

```bash
docker compose up -d db
docker compose logs db
```

ログに `database system is ready to accept connections` のようなメッセージが出れば起動成功です。`docker compose up -d` で `web` を一緒に立ち上げても、React 側はこれまでと同じように初期画面を表示できるはずです。ここまでで Prisma から Postgres へ接続する土台が整いました。次のステップでは Express API を追加し、フロントエンドから DB のデータを取得できるようにしていきます。

コンテナとしては最後の設定になります。
