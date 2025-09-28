# はじめに

Docker、Docker Compose で React・Express・Prisma・PostgreSQL をまとめて立ち上げるフルスタック環境の作り方を、一歩ずつ辿れるチュートリアルです。

最終的には `docker compose up -d` を実行すると `http://localhost:5173` にアクセスするだけで PostgreSQL から取得したタスク一覧が React の画面に描画される状態を再現します。

![GIFの説明](images/Videotogif%20(20).gif)

![画像の説明](images/image.png)

## 学べること

- Docker Compose を使ってフロントエンド・バックエンド・データベースを一体で起動する手順と考え方
- Express と Prisma を組み合わせて DB のデータを API 経由で扱う実践的なパターン
- ローカル環境でも再現性高くフルスタック開発を始めるための初期セットアップの流れ

## 想定読者と前提知識

- Web アプリ開発の基本（React のコンポーネント構造や Express のルーティングなど）を一通り触ったことがある
- Docker や Prisma、PostgreSQL を単体で試したことはあるが、複数スタックを組み合わせて動かすのは初挑戦
- CLI での作業や各種設定ファイルの編集に抵抗がない

本チュートリアルはコンテナの概念や REST API の基礎をゼロから解説するものではありません。まったくの未経験から学びたい場合は、先に各技術の入門チュートリアルで雰囲気を掴んでおくと安心です。

# 前提と準備物

## 用意しておく環境

- Docker Desktop もしくは Docker Engine と Docker Compose v2
- Node.js
  - 今回は22.16.0
- npm または pnpm（プロジェクトは npm を想定）

手元の環境で以下のコマンドを実行し、バージョンが表示されるか確認しておきましょう。

```bash
docker --version
docker compose version
node --version
npm --version
```

Docker のバックグラウンドサービスが起動していることと、`docker compose` コマンドが使えることを事前にチェックしておくと、後続の手順でのつまずきを減らせます。WSL2 やリモート環境を使っている場合は、ファイル共有やポートフォワードの設定が想定どおりに機能するかも合わせて確認しておくと安心です。

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

![ディレクトリ構造](images/dir.png)

このプロジェクトのgithubもありますのでcloneしていただければ手元のマシンで再現できます。

https://github.com/

## コンテナ構成と役割

![ディレクトリ構造](images/containerRel.png)

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

# ステップ1: リポジトリ初期化とベース設定

まずはローカルで React + TypeScript の開発環境を用意し、Vite の開発サーバーが正しく動くことを確認します。プロジェクトを置きたいディレクトリで以下のコマンドを実行してください。

```bash
npm create vite@latest
```

対話式のプロンプトが表示されたら、次のように選択します。

1. プロジェクト名を入力（例: `react-docker-prisma`）
2. フレームワークに `React` を選ぶ
3. バリアントは `TypeScript`（`TypeScript + SWC` ではなくプレーンな TypeScript）を選ぶ

生成が完了したら、案内されたコマンドで依存関係をインストールし、開発サーバーを起動します。

```bash
cd react-docker-prisma
npm install
npm run dev
```

ブラウザで `http://localhost:5173` を開き、Vite の初期画面が表示されることを確認してください。

![vite正常起動後の画像](images/viteAndReactLogo.png)

ここまでができれば、React + TypeScript の土台が整いました。次のステップから Docker 化の作業に入っていきます。

# ステップ2: Docker 基盤づくり（web コンテナ）

ここからはローカルで動いている Vite アプリをコンテナ化し、Docker 上でも同じ画面が表示できるかを確かめます。まずは web コンテナのベースとなる `Dockerfile` をプロジェクト直下に用意しましょう。

```Dockerfile
FROM node:22.20.0
WORKDIR /code
COPY package.json /code/
RUN npm install
COPY . /code/
```

最小構成ですが、次のポイントを押さえています。

- `node:22.20.0` をベースにすることで、ホスト環境に依存せず同じ Node.js を使い回せる
- 作業ディレクトリを `/code` に固定して、以降のコマンドのルートを揃える
- 先に `package.json` をコピーして `npm install` を実行することで、依存関係のキャッシュを効かせる（その後にソースコード全体をコピー）

続いて `Dockercompose.yml` をプロジェクト直下に用意しましょう。そして `web` サービスを追加します。後続ステップで `apiサービス` や `dbサービス` を足しますが、まずはフロント単体の起動に集中します。

```yaml
services:
  web:
    build: .
    command: npm run dev -- --host --port 5173
    volumes:
      - .:/code
    ports:
      - "5173:5173"
```

- `build: .` で先ほどの `Dockerfile` を使ってイメージを組み立てます
- `command` に `--host` を付けているのは、コンテナ外（ホスト）からのアクセスを受け付けるためです
  - 注意：これがないとホストマシンや外部からのアクセスを受け付けないので必ず追加しましょう。
- `volumes: .:/code` でホストの作業ディレクトリと同期させ、React ファイルを編集するとコンテナ側でも即時反映されます
- `ports` でホストの 5173 番ポートを公開し、ブラウザからアクセスできるようにします

設定ができたら、web サービスだけを起動して挙動を確認します。

```bash
docker compose up -d web
```

初回はイメージのビルドが入るので少し時間がかかります。起動後にログを確認し、Vite が `5173` で待ち受けていることを確かめましょう。

```bash
docker compose logs -f web
```

ブラウザで `http://localhost:5173` を開くと、コンテナ上で動いている Vite の初期画面が表示されます。

![vite正常起動後の画像](images/viteAndReactLogo.png)

Step1 で確認したローカル実行と同じ見た目になれば、フロントエンドのコンテナ化は成功です。ここまでの時点ではバックエンドや DB はまだ用意していないので、そのまま `docker compose down` で一度停止して次のステップに進みましょう。

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

# ステップ5: React フロントエンドから API を呼ぶ

API が整ったので、フロントエンドからタスク一覧を取得して描画します。ここでは API 呼び出し用のラッパーと、`App.tsx` の最低限の実装を確認します。

## 以下の内容をApp.tsxにコピペ（既存の内容は削除してください）

```tsx
import { getTasks } from "./API"
import { useState, useEffect } from 'react'

type Task = {
  id: number,
  title: string,
  description: string,
  completed: boolean,
  createdAt: string,
  updatedAt: string,
}

const App = () => {
  const [tasks, setTasks] = useState<Task[]>([])
  const [loading, setLoading] = useState<boolean>(true)
  const [error, setError] = useState<Error | null>(null)

  useEffect(() => {
    const fetchTasks = async () => {
      try {
        const fetchedTasks = await getTasks()
        setTasks(fetchedTasks)
      } catch (err) {
        setError(err as Error)
      } finally {
        setLoading(false)
      }
    }
    fetchTasks()
  }, [])

  if (loading) return <h1>Loading tasks...</h1>
  if (error) return <h1>Error: {error.message}</h1>

  return (
    <>
    {/* 中央寄せ */}
    <div style={{marginLeft: '200px', marginTop: '0px'}}>
      {tasks.map((task) => (
        <div key={task.id}>
          <h2>{task.title}</h2>
          <p>{task.description}</p>
        </div>
      ))}
    </div>
    </>
  )
}
export default App;
```

## デフォルトcssの削除

App.cssとindex.cssがあるかと思いますが、両方不要なので削除してください。

ここまで反映したら、`docker compose up -d api db` でバックエンドを起動した状態のまま `docker compose up -d web` を実行し、ブラウザで `http://localhost:5173` を確認してみてください。シード済みのタスクが画面に表示されれば、フロントエンドと API の連携は完了です。

![画像の説明](images/fetchSuccess.png)

これですべてのコンテナの連携が取れました。最後に一度現状のコンテナを削除し、docker compose up -dで再起動し連携が一発で取れることを確認しましょう。

# ステップ6: Docker Compose で一気に起動

ここまで作成した `web`・`api`・`db` をまとめて立ち上げ、フロント画面からタスクが取得できることを最終確認します。

## 今あるコンテナを削除する

以下のコマンドで現状動いているコンテナを消しましょう。

```bash
docker compose down
```

## 3つのコンテナを一気に起動する

以下コマンドで今まで作成して3つのコンテナを同時に起動します。imageはすでにあるのでキャッシュが走って比較的高速に立ち上がるかと思います。

```bash
docker compose up -d
```

## ブラウザで動作を確認する

ブラウザで `http://localhost:5173` を開き、Prisma のシードで投入したタスクが一覧表示されることを確認します。ページが表示された瞬間に `GET /api/tasks` が呼ばれ、API 経由で取得したデータが描画される流れになります。

![画像の説明](images/fetchSuccess.png)

もしデータが表示されない場合は、以下をチェックしてみてください。

- `docker compose logs api` でエラーが出ていないか（`DATABASE_URL` の設定漏れなど）
- `docker compose logs db` で DB が起動完了しているか
- フロントのネットワークタブで `http://localhost:3000/api/tasks` が成功しているか

問題なく表示できたら、最後に `docker compose down` でコンテナを停止しておきます。これで Docker Compose を使った React × Express × Prisma × PostgreSQL のフルスタック環境構築は完了です。お疲れ様でした。

疑問点等あればお気軽にコメントください。

# 補足

## もっとシードデータを用意して試す

seed.jsを以下の様に編集してください

```js
import { PrismaClient } from '@prisma/client'

const prisma = new PrismaClient()

const main = async () => {
  const task1 = await prisma.task.create({
    data: {
      title: 'タスク1',
      description: '1番目のタスク',
      completed: false,
    },
  })

  const task2 = await prisma.task.create({
    data: {
      title: 'タスク2',
      description: '2番目のタスク',
      completed: true,
    },
  })

  const task3 = await prisma.task.create({
    data: {
      title: 'タスク3',
      description: '3番目のタスク',
      completed: false,
    },
  })

  const task4 = await prisma.task.create({
    data: {
      title: 'タスク4',
      description: '4番目のタスク',
      completed: true,
    },
  })

  const task5 = await prisma.task.create({
    data: {
      title: 'タスク5',
      description: '5番目のタスク',
      completed: false,
    },
  })

  console.log('Created 5 tasks:', { task1, task2, task3, task4, task5 })
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

docker compose exec apiを実施する

```bash
docker compose exec api node prisma/seed.js
```

ローカルで開いている5173ポートにアクセスしているタブを再読み込みすると以下の様に表示されます。

![画像の説明](images/moreSeed.png)
