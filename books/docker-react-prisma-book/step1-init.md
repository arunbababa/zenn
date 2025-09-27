---
title: "ステップ1: リポジトリ初期化とベース設定"
---

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

![vite正常起動後の画像](/images/viteAndReactLogo.png)

ここまでができれば、React + TypeScript の土台が整いました。次のステップから Docker 化の作業に入っていきます。
