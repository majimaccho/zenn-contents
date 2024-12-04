---
title: "PrismaでDBのコメントをスキーマと同期させる"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["TypeScript"]
published: false
---

Server Side TypeScript で DB アクセスするツールとして、近年デファクトとなっている Prisma ですが、待望されているけれども決して実装されない機能があります。

「スキーマのコメントと DB のコメントを同期させる」ことです。

Issue としては 2021 年からありますが、いまだに Open のままです。
https://github.com/prisma/prisma/issues/8703

現時点でも実装の予定はなく、Issue としては長らく放置されていましたが、

2023 年にコミュニティーから Generator を使って実現する方法が提示されました。
現状としては Generator 経由でコメントを生成するしか方法はないようです。
https://github.com/prisma/prisma/issues/8703#issuecomment-1725486294

上記の Issue に Generator を自作する方法が載っていますが、

- 毎回全コメントとなり、差分がわかりずらい
- コメントを消した場合に反映されない
  という課題もあります。

その課題を解決した実装の npm パッケージが公開されています。作者は日本人の方のようです。
https://github.com/onozaty/prisma-db-comments-generator

作者様の紹介記事：
https://zenn.dev/onozaty/articles/prisma-comment

## 実際に使ってみた

インストール

設定

実行

作成されるファイル

コメントを追加してみる

コメントを修正してみる

コメントを消してみる

## おまけ

DB のコメント忘れを防ぎたければ、tbls の lint 機能を使うとできるよ
