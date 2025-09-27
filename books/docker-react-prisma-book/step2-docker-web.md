---
title: "ステップ2: Docker 基盤づくり（web コンテナ）"
---

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

![vite正常起動後の画像](/images/viteAndReactLogo.png)

Step1 で確認したローカル実行と同じ見た目になれば、フロントエンドのコンテナ化は成功です。ここまでの時点ではバックエンドや DB はまだ用意していないので、そのまま `docker compose down` で一度停止して次のステップに進みましょう。
