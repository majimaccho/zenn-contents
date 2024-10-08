---
title: "Rustのgardeトレイトを使うときはUnvalidatedを使おう"
emoji: "🦍"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["rust"]
published: true
---

## gardeトレイトの概要
gardeトレイトは`Varidate`というマクロを提供しており、structのフィールドの検証を行えます。
[公式ドキュメント](https://docs.rs/garde/latest/garde/#basic-usage-example)に記載されている下記の例ではUserというstructのフィールドの文字列がASCII文字列であるかどうか、と長さが適切かどうかを検証しています。

```rust
use garde::{Validate, Valid};

#[derive(Validate)]
struct User<'a> {
    #[garde(ascii, length(min=3, max=25))]
    username: &'a str,
    #[garde(length(min=15))]
    password: &'a str,
}

let user = User {
    username: "test",
    password: "not_a_very_good_password",
};

// Userが不正な値を保持していないか検証する
if let Err(e) = user.validate() {
    println!("invalid user: {e}");
}
```

## 課題感
上記のように`Validate`マクロの`validate`関数を使う場合、
検証済の型と未検証の型が一緒くたにされてしまいます。

公式ドキュメントの例や筆者が観測する限りのインターネット上のコード例ではそのように実装されています。

関数型ドメインモデリング（原題はDomain Modeling Made Functional）で提唱されているように厳格な型システムによる、堅牢なプログラミングのためには
「不正な状態を定義不能にせよ」["make illegal states unrepresentable"](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/)や「検証せず、パースせよ」[Parse, don’t validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)といった考え方が重要になります。

そのため、理想としては明示的に未検証の値と検証済の値を分けて定義したいところです。

## 解決方法

`garde`には`Unvalidated`と`Valid`というstructがあり、これを利用します。

`Unvalidated<T>`エンティティの`.validate()`の戻り値は`Result<Valid<T>, Report>`であるため、`Valid<T>`という検証済の型を作ることができます

RustのgardeトレイトはRustの代表的Webサーバーのトレイトであるaxumとも連携して、リクエストの値などの検証を宣言的に行うことができます。
Webの開発ではaxumと連携して以下のように使われるのではないでしょうか。


### 修正前
```rust
use garde::Validate;
use serde::Deserialize;

// gardeを使ったstructの定義
#[derive(Deserialize, Validate)]
#[serde(rename_all = "camelCase")]
pub struct CreateUserRequest {
    #[garde(length(min = 1))]
    name: String,
    #[garde(email)]
    email: String,
    #[garde(length(min = 1))]
    password: String,
}

// Fromトレイトで型の詰め替えを定義
impl From<CreateUserRequest> for CreateUser {
    fn from(value: CreateUserRequest) -> Self {
        let CreateUserRequest {
            name,
            email,
            password,
        } = value;
        Self {
            name,
            email,
            password,
        }
    }
}

/// ユーザーを追加する
pub async fn register_user(
    /// 他引数は省略
    Json(req): Json<CreateUserRequest>,
) -> AppResult<Json<UserResponse>> {
    // 認証関連の処理など省略
    
    req.validate(&())?; // ←　ここで値を検証

    let registered_user = registry.user_repository()
        .create(req.into()) // ←　ここでモデルの型（CreateUser）にinto()で変換する
        .await?;
    Ok(Json(registered_user.into()))
}

```

これを`Unvalidated`と`Valid`を使って以下のように記述します。

### 修正後

```rust
use garde::{Validate, Unvalidated}; // <- Unvalidatedを追加

///register_user以前は同じ

pub async fn register_user(
    /// 他引数は省略
    Json(raw_req): Json<CreateUserRequest>,
) -> AppResult<Json<UserResponse>> {
    // 認証関連の処理など省略
    let valid_req = Unvalidated::new(raw_req).validate(&())?; // ←　ここで値を検証、valid_reqは検証済の値

    let registered_user = registry.user_repository()
        .create(valid_req.into_inner().into()) // ←　into_innerでCreateUserRequestを取り出してから変換する
        .await?;
    Ok(Json(registered_user.into()))
}

```

このようにすることで、`valid_req`を`req`とは分けて定義することができ、不正な状態があり得る型を未検証のまま使うこともなくなりました。
