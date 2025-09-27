---
title: "ステップ5: React フロントエンドから API を呼ぶ"
---

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

![画像の説明](/images/fetchSuccess.png)

これですべてのコンテナの連携が取れました。最後に一度現状のコンテナを削除し、docker compose up -dで再起動し連携が一発で取れることを確認しましょう。
