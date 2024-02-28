---
title: "TypeScriptのSymbol.DisposeとHOCで安全&イミュータブルにDBとのコネクションを使用する"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript"]
published: true
---

TypeScriptでsqlcやpgTypedなどの薄いクエリビルダーを活用する際には、DBコネクションの状態管理をする必要があります。

DBコネクションの状態管理をする際には、以下のようにコネクションの接続と解放を意識する必要があります。
1. コネクションの接続
2. クエリの実行
3. コネクションの解放

このような状態管理を手続き型で実装することには以下のような問題があります。
- コネクションの解放漏れが発生する可能性がある
- 状態管理を行うため、安全&イミュータブルにできない
- コネクションの接続&解放を意識する必要があるため、ドメインロジックに集中できない

これらを解決するために、HOC（高階関数）とSymbol.Disposeを使って安全&イミュータブルにDBとのコネクションを使用する方法を紹介します。

実際に手続き型的に素直に書くとこうなります


```typescript
// 手続き型コード例

import { pool } from 'lib/db'

// ユーザー情報を取得するControllerもしくはResolver
export const users = async () => {
  // 入力値の検証、値の変換、エラーハンドリングなどは省略
    
  // コネクションの管理を意識して書く必要があり、ロジックに集中できない
  const client = await pool.connect()

  try {
    const res = queryUser(client)
    return res
  } finally {
    client.release()
  }
}
```


このように書くとコネクションの利用後に解放する処理を忘れないように気をつけなければならず認知負荷が高まります。
ここで、HOCを使って、接続の状態管理を隠蔽します。

```typescript
// HOCを使ってコネクションの接続&解放を隠蔽
import { Pool, PoolClient } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
})

// HOCによる実装。接続されたClientを利用する関数を引数に取り、接続後に解放する
export const usePgConnection = async <T>(callback: (client: Client) => Promise<T>):Promise<T> => {
  const pool = new Pool()
  const client = await pool.connect()
  try {
    return callback(client)
  } finally {
    client.release()
  }
}

```

```typescript
// HOCを利用したコード
export const users = async () => {
  return usePgConnection(async (client) => {
    const res = queryUser(client)
    return res
  })
}

```

このようにすることで、呼び出し側でコネクションの接続を意識する必要がなくなり、ドメインロジックに集中できます。

今回のケースではtry句でreturnしているので問題にならないですが、一般的にtry/finalyを使う場合、コネクションが開放された後のクライアントにアクセスしてしまう可能性があります。
そこで、[TypeScript 5.2で入ったSymbol.Dispose](https://www.typescriptlang.org/docs/handbook/release-notes/typescript-5-2.html#using-declarations-and-explicit-resource-management)でスコープから抜けた際に安全にコネクションを解放できます。

コネクションプールとクライアントの利用を以下のようにAsyncDisposableクラスとして実装します。


```typescript
// AsyncDisposableクラスの実装

import { Pool, PoolClient } from 'pg'

const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
})

/**
 * PgPoolのコネクション管理を行うクラス
 * AsyncDisposableを実装しており、new された関数が終了するときに
 * コネクションを解放する
 */

class PgConnection implements AsyncDisposable {
  #client?: PoolClient

  constructor() {
    this.#pool = new Pool()
  }

  async connect() {
    this.#client = await this.#pool.connect()
  }

  getClient() {
    return this.#client
  }

  // new PgConnection()で生成したインスタンスがスコープから抜けた際に呼ばれる
  async [Symbol.asyncDispose]() {
    this.#client.release()
  }
}

export const usePgConnection = async <T>(callback: (client: Client) => Promise<T>):Promise<T> => {
  await using pgConnection = new PgConnection() // usingはDisposable/AsyncDisposableを実装したクラスを利用する際に使う
  await pgConnection.connect()
  const client = pgConnection.getClient()
  return callback(client)
}

```

このようにすれば、`try/finally`を書かなくても、usePgConnection関数の完了時にコネクションが解放されます。

HOCとSymbol.Disposeを使うことでcallback関数の終了時にコネクションを解放するようにすると、利用側でコネクションの接続&解放を意識する必要がなくなります。
HOCで隠蔽することで安全&イミュータブルにコネクションが管理できるようになります。
今回の例では、try句内でreturnしているので、Symbol.Disposeを利用する必要は必ずしもありませんが、
Symbol.Disposeを利用することで、try/finallyよりも安全にコネクションの解放漏れを防ぐことができます。