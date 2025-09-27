---
title: "補足"
---

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

![画像の説明](/images/moreSeed.png)
