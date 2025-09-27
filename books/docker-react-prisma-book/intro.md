---
title: "はじめに"
---

# はじめに

Docker、Docker Compose で React・Express・Prisma・PostgreSQL をまとめて立ち上げるフルスタック環境の作り方を、一歩ずつ辿れるチュートリアルです。

最終的には `docker compose up -d` を実行すると `http://localhost:5173` にアクセスするだけで PostgreSQL から取得したタスク一覧が React の画面に描画される状態を再現します。

![GIFの説明](/images/Videotogif%20(20).gif)

![画像の説明](/images/image.png)

## 学べること

- Docker Compose を使ってフロントエンド・バックエンド・データベースを一体で起動する手順と考え方
- Express と Prisma を組み合わせて DB のデータを API 経由で扱う実践的なパターン
- ローカル環境でも再現性高くフルスタック開発を始めるための初期セットアップの流れ

## 想定読者と前提知識

- Web アプリ開発の基本（React のコンポーネント構造や Express のルーティングなど）を一通り触ったことがある
- Docker や Prisma、PostgreSQL を単体で試したことはあるが、複数スタックを組み合わせて動かすのは初挑戦
- CLI での作業や各種設定ファイルの編集に抵抗がない

本チュートリアルはコンテナの概念や REST API の基礎をゼロから解説するものではありません。まったくの未経験から学びたい場合は、先に各技術の入門チュートリアルで雰囲気を掴んでおくと安心です。
