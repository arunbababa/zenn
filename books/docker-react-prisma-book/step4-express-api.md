---
title: "ステップ4: Express API 実装"
---

# ステップ4: Express API 実装

Prisma でデータベースの準備が整ったので、React から扱えるように Express API を追加します。フロントと DB の橋渡し役として API サーバーをコンテナ化し、エンドポイント経由でタスクを取得できるようにします。

## Compose に `api` サービスを追加

`Dockercompose.yml` に API 用のサービスを追記します。`db` が起動してから立ち上がるよう `depends_on` も設定しておきます。

```yaml
services:
  api:
    build: .
    command: node src/server/server.js
    volumes:
      - .:/code
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/react_develop
      - PORT=3000
    depends_on:
      db:
        condition: service_healthy
```

- `command` で Express サーバーのエントリーポイントを指定
- `DATABASE_URL` はステップ3で設定した文字列をそのまま流用し、Prisma が DB に接続できるようにする
- `PORT=3000` を渡してサーバーの待ち受けポートを固定（ホスト側へも 3000 番で公開）
- `depends_on` により DB のヘルスチェックが成功してから API が起動する

この段階で `web` サービスの `depends_on` に `api` を追加しておくと、フロントが API 起動前に 404 を返す状況を避けられます。

```yaml
web:
    build: .
    command: npm run dev -- --host --port 5173
    volumes:
      - .:/code
    ports:
      - "5173:5173"
    # ↓これ↓
    depends_on:
      - api
```

## `src/server/server.js` のベース

API サーバーでは Express と Prisma Client を組み合わせて DB とやり取りします。`src/server/server.js` は次のような構成にします。

```js
import express from 'express';
import cors from 'cors';
import dotenv from 'dotenv';
import { PrismaClient } from '@prisma/client';

dotenv.config();

const app = express();
const prisma = new PrismaClient();
const PORT = 3000;

app.use(cors());

app.listen(PORT, () => {
  console.log(`Server is running on port ${PORT}`);
});
```

- `dotenv.config()` で `.env` を読み込み、`DATABASE_URL` を参照可能にしています
- `app.use(cors())` を有効にしておくと、別ポートで動くフロントエンドからのリクエストも許可できます
  - 今回の場合viteのreactアプリは5173ポートで動かしていますので、この設定がないとリクエストが拒否されます
  - corsについては詳細は[こちら](https://developer.mozilla.org/ja/docs/Web/HTTP/Guides/CORS/Errors)を参考にしてください

## タスク取得エンドポイント

Prisma Client を使ってタスクを取得し、API から返す処理を実装します。エラーハンドリングを簡単に入れておくと原因調査が楽になります。

```js
app.get('/api/tasks', async (req, res) => {
  try {
    const tasks = await prisma.task.findMany();
    res.json(tasks);
  } catch (error) {
    console.error('Error fetching tasks:', error);
    res.status(500).json({ error: 'Failed to fetch tasks' });
  }
});
```

この段階で `npm run dev` を止め、`docker compose up -d api` を実行すれば API コンテナが起動します。`docker compose logs -f api` で `Server is running on port 3000` が出ていれば準備完了です。試しに `curl http://localhost:3000/api/tasks` を叩くと、現時点では空配列 `[]` が返るはずです。なお試す場合はdocker compose exec web bashでコンテナ内のwebサービスでbashを開いた後に`curl http://localhost:3000/api/tasks`を叩いてください。

## シードデータを投入する

表示用のデータが欲しい場合は、`prisma/seed.js` を使って初期タスクを登録しておきます。プロジェクト直下のprisamaディレクトリ内にseed.jsを作成し以下の様に追記してください。

```js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const main = async () => {

    const task1 = await prisma.task.create({
      data: {
        title: 'Prismaで作成したタスク1',
        description: 'これはPrismaのシードスクリプトで作成されました',
        completed: false,
      },
    })

    console.log('Created tasks:', { task1 })
}

const runSeed = async () => {
  try {
    await main()
  } catch (e) {
    console.error('Error seeding database:', e)
    process.exit(1)
  } finally {
    await prisma.$disconnect()
  }
}

runSeed()
```

コンテナ上でスクリプトを実行するには、API サービスが起動した状態で次のコマンドを実行します。

```bash
docker compose exec api node prisma/seed.js
```

シードが終わったら再度 `curl` やブラウザから `/api/tasks` を確認し、タスクが配列で返ってくることを確かめましょう。これでフロントエンドが利用できる API レイヤーが整いました。
