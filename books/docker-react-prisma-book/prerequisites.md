---
title: "前提と準備物"
---

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
