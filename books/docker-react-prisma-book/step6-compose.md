---
title: "ステップ6: Docker Compose で一気に起動"
---

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

![画像の説明](/images/fetchSuccess.png)

もしデータが表示されない場合は、以下をチェックしてみてください。

- `docker compose logs api` でエラーが出ていないか（`DATABASE_URL` の設定漏れなど）
- `docker compose logs db` で DB が起動完了しているか
- フロントのネットワークタブで `http://localhost:3000/api/tasks` が成功しているか

問題なく表示できたら、最後に `docker compose down` でコンテナを停止しておきます。これで Docker Compose を使った React × Express × Prisma × PostgreSQL のフルスタック環境構築は完了です。お疲れ様でした。

疑問点等あればお気軽にコメントください。
