---
title: "データエンジニアに嫌われないPrismaの使い方"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript", "prisma", "dataengineering"]
published: true
---

みなさんこんにちは。CADDi で CADDi Quote というプロダクトの開発をしている majimaccho です。
こちらは CADDi アドベントカレンダー 14 日目の投稿です。

前回は モリ さんの [開発中の「モヤモヤ」が教えてくれること](https://note.com/morik/n/nf79c96959b7a)でした。
こちらもぜひ合わせてお読みください！

CADDi ではサーバーサイド TypeScript で DB アクセスをするのに Prisma を利用しています。また、Drawer と Quote という 2 つの製品があり、それぞれ連携して使われるための ID 基盤なども整備されていたり、プロダクト横断のデータ基盤によって、それぞれのアプリケーションのデータを連結して解析できるようになっています。

## はじめに

みなさんは Prisma を使っている時に、データエンジニアの方がやりづらそうにしていて申し訳ない気持ちになったことはないでしょうか？
近年はコンパウンドスタートアップが増えてきたり、大企業でも全社のデータ基盤を整備したりしており、サーバーサイドエンジニアとデータエンジニアが協業する機会も増えてきていると思います。

そんな中で、公式ドキュメント通りに Prisma を使っているとデータ基盤との連携の際にデータエンジニアを困らせてしまうこともあります。データエンジニアの仕事は企業としての意思決定を行うための重要な分析に影響するためビジネスとしての価値は大きいです。アプリケーションエンジニアとしても彼らが困らないような DB の設計やコミュニケーションをしていきたいところです。
今回はデータエンジニアに嫌われない、もしくはデータエンジニアを困らせない Prisma の使い方についてご紹介します！

## Prisma 由来の問題でデータエンジニアが困ること

データエンジニアがアプリケーションエンジニアとのやり取りで最も困るのは、そもそものデータベース設計が適切になっていないというのが大きいです。しかし、中には**Prisma を使っているせいで**データエンジニアを困らせてしまうことがあります。その中でも今回の記事で紹介するのは以下の 2 つです。

- Prisma のスキーマのコメントとメタデータのコメントが同期していない
- テーブル名・カラム名がキャメルケースになっている

## Prisma のスキーマのコメントとメタデータのコメントが同期していない

### 課題

Prisma のコメントは`///`と書くことで Prisma のスキーマの定義と生成される TypeScript の型のコメントを同期させることができます。

多くのアプリケーションエンジニアにとってはこれだけあれば十分ですが、データエンジニアは基本的には DB の中しか見に行かないため Prisma のスキーマに書かれている情報は完全に抜け落ちてしまいます。Prisma のスキーマのコメントと DB のコメントを同期させる需要は大きく、GitHub の Issue は 2021 年からありますが、いまだに Open のままです。

https://github.com/prisma/prisma/issues/8703

現時点でも実装の予定はなく、Issue としては長らく放置されていましたが、2023 年にコミュニティーから Generator を使って実現する方法が提示されました。[現状としては Generator 経由でコメントを生成するしか方法はないようです。](https://github.com/prisma/prisma/issues/8703#issuecomment-1725486294)

### 解決策

上記の Issue に Generator を自作する方法が載っていますが、

- 毎回全コメントとなり、差分がわかりずらい
- コメントを消した場合に反映されない
  という課題もあります。

その課題を解決した実装の npm パッケージが公開されています。作者は日本人の方のようです。
https://github.com/onozaty/prisma-db-comments-generator

作者様の記事：
https://zenn.dev/onozaty/articles/prisma-comment

上記 GitHub からの転載ですが、以下のような Prisma のスキーマファイルがあったときにその下のようなコメントを生成する DDL が生成されます。

```prisma
generator client {
  provider = "prisma-client-js"
}

generator comments {
  provider = "prisma-db-comments-generator"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

/// Product type enumeration
enum ProductType {
  BOOK
  TOY
  FASHION

  @@map("enum_product_type")
}

/// Customer
model Customer {
  /// Customer ID
  customerId Int      @id @default(autoincrement()) @map("customer_id")
  /// Customer Name
  name       String
  /// e-mail
  email      String   @unique
  createdAt  DateTime @default(dbgenerated("statement_timestamp()")) @map("created_at") @db.Timestamptz()
  sales      Sale[]

  @@map("customers")
}

/// Product
model Product {
  /// Product ID
  productId   Int         @id @default(autoincrement()) @map("product_id")
  /// Product Name
  name        String
  /// Product Type
  type        ProductType
  /// Product Description
  description String?
  /// Price
  price       Float
  createdAt   DateTime    @default(dbgenerated("statement_timestamp()")) @map("created_at") @db.Timestamptz()
  sales       Sale[]

  @@map("products")
}

model Sale {
  /// Sale ID
  saleId     Int      @id @default(autoincrement()) @map("sale_id")
  customer   Customer @relation(fields: [customerId], references: [customerId])
  /// Customer ID
  customerId Int      @map("customer_id")
  product    Product  @relation(fields: [productId], references: [productId])
  /// Product ID
  productId  Int      @map("product_id")
  /// Quantity
  quantity   Int
  /// Total Price
  totalPrice Float    @map("total_price")
  createdAt  DateTime @default(dbgenerated("statement_timestamp()")) @map("created_at") @db.Timestamptz()

  @@map("sales")
}
```

```sql
-- Prisma Database Comments Generator v1.0.1

-- customers comments
COMMENT ON TABLE "customers" IS 'Customer';
COMMENT ON COLUMN "customers"."customer_id" IS 'Customer ID';
COMMENT ON COLUMN "customers"."name" IS 'Customer Name';
COMMENT ON COLUMN "customers"."email" IS 'e-mail';

-- products comments
COMMENT ON TABLE "products" IS 'Product';
COMMENT ON COLUMN "products"."product_id" IS 'Product ID';
COMMENT ON COLUMN "products"."name" IS 'Product Name';
COMMENT ON COLUMN "products"."type" IS 'Product Type';
COMMENT ON COLUMN "products"."description" IS 'Product Description';
COMMENT ON COLUMN "products"."price" IS 'Price';

-- sales comments
COMMENT ON COLUMN "sales"."sale_id" IS 'Sale ID';
COMMENT ON COLUMN "sales"."customer_id" IS 'Customer ID';
COMMENT ON COLUMN "sales"."product_id" IS 'Product ID';
COMMENT ON COLUMN "sales"."quantity" IS 'Quantity';
COMMENT ON COLUMN "sales"."total_price" IS 'Total Price';
```

このパッケージを使うとコメント用の migration の SQL ファイルに加えて、コメントのメタデータが格納された JSON ファイルも生成されます。この JSON ファイルに今まで反映されたコメントの最新状態をキャッシュしておき、`prisma generate`のタイミングで Prisma のスキーマと JSON 内のコメントの差分のみをコメントとして生成するという仕組みになっています。

## テーブル名・カラム名がパスカル・キャメルケース＆単数系になっている

### 課題

Prisma はデフォルトでテーブル名をパスカルケース、カラム名をキャメルケースで生成します。
Prisma の model 名が`ProgrammingLanguage`で、フィールド名が`releasedAt`だった場合、データベース上でもそのままテーブル名が`ProgrammingLanguage`でカラム名が`releasedAt`になります。

TypeScript ではこのようにパスカル・キャメルケースにすることが自然ですが RDB の世界では、大文字と小文字を区別せず、スネークケースで記述することが一般的です。そのため、TypeScript の事情を DB にまで持ち込むのは、データエンジニアにとってあまり自然なこととは感じられません。

現代では適切にエスケープをすることで大文字小文字を区別できる RDBMS もあるのでそこまで大きな問題にはならないかもしれません。ただし、データエンジニアが利用する一部のツールでは、データソースで大文字小文字が区別できていても、変換時に全て小文字にされてしまうことがあります。そもそも普段 SQL を使ってデータを操作する人にとって大文字小文字の区別があると扱いづらい形式です。Prisma でも`@map`を使ってデフォルトとは違うテーブル・カラム名にマッピングすることは可能ですが、アプリケーションエンジニアにとっては必要のない作業を増やすのはあまり好ましくないでしょう。

また、prisma のモデルがそのままテーブル名になっているため、テーブル名は単数系になります。
~~RDB のテーブルは集合であるため、複数形の方が適切であり、単数系にしていると RDB への理解が足りないエンジニアが設計しているという不信感を芽生えさせてしまうかもしれません。~~

### 解決策

Prisma のスキーマに対して自動で`@map`を設定してくれる`prisma-case-format`という CLI ツール があります。

https://github.com/iiian/prisma-case-format

以下のような Prisma のスキーマファイルがあったときに

```prisma
...
model HouseRating {
  id       Int    @id @default(autoincrement())
  houseId  String
  house    House  @relation(fields: [houseId], references: [id])
}

model House {
  id           String        @id @default(uuid())
  houseRating HouseRating[]
}
```

下記のコマンドを実行すると Prisma のスキーマファイルが以下のように整形されます。

```shell
# --map-table-case=snake,pluralを設定するとテーブル名をスネークケース＆複数形にできる
prisma-case-format --map-table-case=snake,plural
```

```prisma
model HouseRating {
  id       Int    @id @default(autoincrement())
  houseId  String @map("house_id")
  house    House  @relation(fields: [houseId], references: [id])

  @@map("house_ratings")
}
...
model House {
  id           String        @id @default(uuid())
  houseRating HouseRating[]

  @@map("houses")
}
```

ちゃんとテーブル名はスネークケースの複数形、カラム名はスネークケースの単数系にマッピングされています。
これで、データエンジニアに Prisma のせいで不自然なテーブル・カラム名になると言われることはなくなるでしょう。

デフォルトでは特に辞書などは用意されていないので、不自然な複数形になってしまうこともありますが、設定ファイルからテーブルごとの辞書を設定することもでき、ルールベースで変換されたくないテーブル、カラム名などを独自に設定することができます。

## 番外編

Prisma とは関連がないですが、DB スキーマを理解するのを助けるツールとして tbls があります。

https://github.com/k1LoW/tbls

DB のテーブル・カラムコメントを必須にするリントや、任意の単位でテーブルをグルーピングできる Viewpoint など、初めてそのプロジェクトの DB を見る人でもすぐにキャッチアップできるような仕組みがたくさんあります。データエンジニアとのコミュニケーションに限らず、アプリケーションエンジニアにとっても DB スキーマを管理・整理する上で非常に便利なツールです。

詳しくは以下の記事で書いたのでご一読いただけると幸いです。

https://zenn.dev/mj2mkt/articles/2023-12-05-majimaccho-tbls
