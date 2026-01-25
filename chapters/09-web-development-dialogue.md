# Chapter 09: Web開発 - 対話編

> **この章で学ぶこと**: Axumフレームワーク、ルーティング、ミドルウェア、JSON API — RustでのWeb開発を実践的に

> 📖 **記号リファレンス**: この章では `async`, `await`, `Arc`, `impl IntoResponse` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。また、非同期処理の復習には [Chapter 08: 非同期処理](08-async.md) を参照してください。

---

## 9-1. RustでWeb開発する理由

**レイ**: お、今日は何やるの？

**ユイ**: 今日はRustでWeb APIを作るよ。AxumっていうWebフレームワークを使う。

**レイ**: Web APIかー。Node.jsのExpressとかPythonのFastAPIなら触ったことあるけど、RustでWeb開発ってどうなの？正直、ちょっと大変そうなんだけど...

**ユイ**: いい質問だね。確かにRustはExpressやFastAPIに比べると学習コストは高いけど、得られるメリットも大きいよ。表で比較してみようか。

| 観点 | Node.js/Express | Python/FastAPI | Rust/Axum |
|-----|-----------------|----------------|-----------|
| パフォーマンス | 中程度 | 中程度 | **非常に高速** |
| メモリ効率 | GCあり | GCあり | **GCなし、効率的** |
| 型安全性 | TypeScriptで改善 | 型ヒント | **コンパイル時に保証** |
| 同時接続 | イベントループ | asyncio | **async + マルチスレッド** |
| 起動時間 | 速い | 普通 | **非常に速い** |

**レイ**: パフォーマンスとメモリ効率がかなり良いんだね。どういう場合にRustを選ぶべき？

**ユイ**: こんな用途に向いてるかな：

- 高負荷なAPIサーバー（秒間数万リクエストとか）
- マイクロサービス（低レイテンシが重要）
- リアルタイム処理（WebSocketとか）
- コンテナで動かすとき（メモリ効率が重要）

**レイ**: なるほど。でも、簡単なCRUD APIならExpressやFastAPIでいいよね？

**ユイ**: そうだね。プロトタイピングならNode.jsやPythonの方が速い。でも、本番で負荷がかかるシステムや、型安全性を最初から担保したい場合はRustが輝くよ。

**レイ**: TypeScriptも型安全じゃない？

**ユイ**: TypeScriptは実行時にJavaScriptに変換されるから、**実行時エラー**のリスクは残るんだ。Rustは**コンパイル時**に型をチェックするから、型エラーがあるコードはそもそも実行できない。

**レイ**: なるほど。じゃあRustのWebフレームワークって何があるの？

**ユイ**: 主要なのはこの3つかな：

| Rust | Node.js | Python |
|------|---------|--------|
| **Axum** | Express | **FastAPI** |
| Actix-web | Koa | Flask |
| Rocket | NestJS | Django |

**レイ**: Axumってどういうフレームワークなの？

**ユイ**: Tokioチームが作ってるWebフレームワークで、こんな特徴があるよ：

```
┌─────────────────────────────────────────────────────────────────┐
│  Axum の特徴                                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 型安全                                                       │
│     → エクストラクタで型安全なリクエスト処理                      │
│     → コンパイル時にルーティングエラーを検出                      │
│                                                                 │
│  2. 非同期ファースト                                             │
│     → tokioベース、async/await完全対応                           │
│                                                                 │
│  3. マクロ不要                                                   │
│     → 属性マクロなしでシンプルなAPI                              │
│     → 他のフレームワーク（Rocket等）より読みやすい                │
│                                                                 │
│  4. tower統合                                                    │
│     → 豊富なミドルウェアエコシステム                              │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: マクロ不要ってのが良さそう。Rocketは `#[get("/")]` とかデコレータっぽいの使うよね？

**ユイ**: そうそう。Axumは普通の関数でルーティングを定義できるから、コードが読みやすいんだ。じゃあ実際にプロジェクトを作ってみよう。

---

## 9-2. プロジェクトセットアップ

**ユイ**: まず新しいプロジェクトを作るよ。

```bash
cargo new my_api
cd my_api
cargo add axum
cargo add tokio --features full
cargo add serde --features derive
cargo add serde_json
cargo add tower-http --features cors,trace
cargo add tracing
cargo add tracing-subscriber
```

**レイ**: なんでこんなにクレート追加するの？`package.json` みたいな感じ？

**ユイ**: そうそう。それぞれの役割を説明するね：

- `axum` - Webフレームワーク本体
- `tokio` - 非同期ランタイム（Chapter 08で学んだやつ）
- `serde` + `serde_json` - JSON シリアライズ/デシリアライズ
- `tower-http` - ミドルウェア（CORS、ロギングなど）
- `tracing` - ログ出力

**レイ**: Express で言うところの `express`, `body-parser`, `cors`, `morgan` みたいな感じか。

**ユイ**: まさにそう！Cargo.toml はこんな感じになる：

```toml
[dependencies]
axum = "0.7"
tokio = { version = "1", features = ["full"] }
serde = { version = "1", features = ["derive"] }
serde_json = "1"
tower-http = { version = "0.5", features = ["cors", "trace"] }
tracing = "0.1"
tracing-subscriber = "0.3"
```

---

## 9-3. Hello World から始めよう

**レイ**: じゃあ最初のAPIサーバー作ってみる？

**ユイ**: いいね。`src/main.rs` にこう書いてみて：

```rust
use axum::{routing::get, Router};

async fn hello() -> &'static str {
    "Hello, World!"
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .route("/", get(hello));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server running on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

**レイ**: おお、意外とシンプル！これで動くの？

**ユイ**: うん。`cargo run` してブラウザで `http://localhost:3000` にアクセスしてみて。

**レイ**: 動いた！"Hello, World!" って表示された。でも各行が何やってるかちゃんと理解したいな。

**ユイ**: OK、1行ずつ見ていこう。

```rust
use axum::{routing::get, Router};
```

**ユイ**: これは必要な型と関数をインポートしてる。`Router` はルーティングを定義する構造体で、`get` はHTTP GETメソッドに対応する関数だよ。

```rust
async fn hello() -> &'static str {
    "Hello, World!"
}
```

**レイ**: これは非同期関数だよね。なんで非同期にする必要があるの？

**ユイ**: Axumは全部非同期ベースだからね。実際にはデータベースアクセスとか非同期処理をすることが多いから、ハンドラも非同期にしておくのが自然なんだ。

**レイ**: `-> &'static str` ってなに？

**ユイ**: 静的文字列リテラルを返すって意味。重要なのは、**Axumはこの戻り値を自動的にHTTPレスポンスに変換してくれる**ってこと。

**レイ**: え、自動で？Express だと `res.send()` とか明示的に書くけど。

**ユイ**: そう。Axumは `IntoResponse` トレイトを実装してる型なら、何でもレスポンスに変換できるんだ。文字列もその1つ。

```rust
let app = Router::new()
    .route("/", get(hello));
```

**レイ**: これがルーティングの定義か。

**ユイ**: そう。`Router::new()` で空のルーターを作って、`.route()` でパスとハンドラを紐付けてる。

- `/` というパスに
- GETリクエストが来たら
- `hello` 関数を呼ぶ

ってことだね。

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
```

**レイ**: これはサーバー起動の部分だね。ポート3000で待ち受けてる。

**ユイ**: そう。`0.0.0.0` は全てのネットワークインターフェースでリッスンするって意味だよ。

**レイ**: Express だとこんな感じだよね：

```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.listen(3000);
```

**ユイ**: うん、構造はかなり似てる。FastAPI だとこうかな：

```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def hello():
    return "Hello, World!"
```

**レイ**: FastAPI のデコレータ形式に比べると、Axumは明示的にルーティングを書く感じだね。

**ユイ**: そうだね。好みもあるけど、Axumの方式は大規模になったときにルーティングを整理しやすいよ。

---

## 9-4. ルーティングを理解する

**レイ**: もうちょっと複雑なルーティングも作ってみたいな。REST APIっぽいやつ。

**ユイ**: いいね。じゃあユーザーのCRUD操作を例にしてみよう。

```rust
use axum::{
    routing::{get, post, put, delete},
    Router,
};

async fn list_users() -> &'static str { "List users" }
async fn get_user() -> &'static str { "Get user" }
async fn create_user() -> &'static str { "Create user" }
async fn update_user() -> &'static str { "Update user" }
async fn delete_user() -> &'static str { "Delete user" }

#[tokio::main]
async fn main() {
    let app = Router::new()
        // 同じパスに複数のHTTPメソッドを登録
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

**レイ**: おお、`.get()`, `.post()` ってチェーンできるんだ！

**ユイ**: そう。同じパスに複数のHTTPメソッドを登録できる。Express でいうところの：

```javascript
app.route('/users')
    .get(listUsers)
    .post(createUser);
```

みたいな感じだね。

**レイ**: `/users/:id` の `:id` はパスパラメータだよね？

**ユイ**: そう！`:id` の部分は動的に変わる値を受け取れる。

```
/users/:id
       ↓
/users/123 → id = "123"
/users/456 → id = "456"
```

**レイ**: これをハンドラ内でどうやって取得するの？

**ユイ**: それは次のセクションで詳しくやるよ。**エクストラクタ**っていう仕組みを使うんだ。

**レイ**: 大きなアプリだとルーティングを分割したくなるよね。Express だと別ファイルに `Router` 作って `app.use('/users', userRouter)` みたいにするけど。

**ユイ**: Axumでも同じことができるよ。ネストしたルーターを使う：

```rust
fn user_routes() -> Router {
    Router::new()
        .route("/", get(list_users).post(create_user))
        .route("/:id", get(get_user).put(update_user).delete(delete_user))
}

fn post_routes() -> Router {
    Router::new()
        .route("/", get(list_posts))
        .route("/:id", get(get_post))
}

#[tokio::main]
async fn main() {
    let app = Router::new()
        .nest("/users", user_routes())  // /users 配下に user_routes を配置
        .nest("/posts", post_routes()); // /posts 配下に post_routes を配置

    // ...
}
```

**レイ**: `.nest()` で階層化できるんだ。これなら大きなアプリでも整理しやすいね。

**ユイ**: そうそう。モジュールごとに Router を分けて、最後に組み合わせるのが定石だよ。

---

## 9-5. エクストラクタ - 型安全なリクエスト処理

**レイ**: さっき言ってた「エクストラクタ」って何？

**ユイ**: Axumの最大の特徴の1つで、**リクエストから型安全にデータを取り出す仕組み**だよ。図で説明するね：

```
┌─────────────────────────────────────────────────────────────────┐
│  エクストラクタとは                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  HTTPリクエスト                                                  │
│      ↓                                                          │
│  ┌──────────────┐                                               │
│  │ Path<T>      │ → パスパラメータ（/users/:id の :id）         │
│  │ Query<T>     │ → クエリパラメータ（?page=1&limit=10）        │
│  │ Json<T>      │ → JSONボディ                                  │
│  │ State<T>     │ → アプリケーション状態                        │
│  │ HeaderMap    │ → ヘッダー                                    │
│  └──────────────┘                                               │
│      ↓                                                          │
│  Rustの型に自動変換                                              │
│                                                                 │
│  型が合わなければ → 自動的に400 Bad Requestを返す                 │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: おお、型に合わないと自動でエラー返してくれるの？

**ユイ**: そう！これがRustの強みだよ。Express だと `req.params.id` を `parseInt()` して、`NaN` チェックして...って手動でやるよね。

**レイ**: うんうん、よくやる。

**ユイ**: Axumならそれが全部自動。実際に見てみよう。

### パスパラメータ

```rust
use axum::extract::Path;

// 単一パラメータ
async fn get_user(Path(id): Path<u32>) -> String {
    format!("User ID: {}", id)
}
```

**レイ**: `Path(id): Path<u32>` ってなんか変な書き方だね。

**ユイ**: これは「パターンマッチ付きの引数」なんだ。分解して説明するね：

```
Path(id): Path<u32>
 ↓     ↓      ↓
 │     │      └─ 型: Path<u32>（u32のパスパラメータ）
 │     └─ 変数名: id（関数内で使う名前）
 └─ パターン: Pathから中身を取り出す
```

**レイ**: あー、`Path<u32>` の中身を `id` として取り出してるのか。

**ユイ**: そう。これは実は以下と同等だよ：

```rust
async fn get_user(path: Path<u32>) -> String {
    let id = path.0;  // Path<u32>から中身を取り出す
    format!("User ID: {}", id)
}
```

**レイ**: なるほど！パターンマッチで一気に中身を取り出してるわけだ。

**ユイ**: Express だとこうなるよね：

```javascript
app.get('/users/:id', (req, res) => {
    const id = parseInt(req.params.id);  // 文字列→数値の手動変換
    if (isNaN(id)) {
        return res.status(400).send('Invalid ID');
    }
    res.send(`User ID: ${id}`);
});
```

**レイ**: Axumは型が合わないリクエストは自動で弾いてくれるんだよね？

**ユイ**: そう。`/users/abc` みたいなリクエストが来たら、自動的に `400 Bad Request` を返してくれる。エラーハンドリングのコードを書く必要がない。

**レイ**: 便利すぎる...！パスパラメータが複数ある場合は？

```rust
// 複数パラメータ
async fn get_post(Path((user_id, post_id)): Path<(u32, u32)>) -> String {
    format!("User: {}, Post: {}", user_id, post_id)
}
// ルート: /users/:user_id/posts/:post_id
```

**レイ**: タプルで受け取るのか！

**ユイ**: そう。順番にマッチしてくれるよ。

### クエリパラメータ

**レイ**: クエリパラメータ（`?page=1&limit=10` みたいなやつ）はどうするの？

**ユイ**: `Query` エクストラクタを使う。まず構造体を定義するよ：

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,   // ?page=1
    limit: Option<u32>,  // ?limit=10
}

async fn list_users(Query(params): Query<Pagination>) -> String {
    let page = params.page.unwrap_or(1);
    let limit = params.limit.unwrap_or(10);
    format!("Page: {}, Limit: {}", page, limit)
}
```

**レイ**: `Option<u32>` にしてるのはなんで？

**ユイ**: クエリパラメータは省略可能だからね。`Option` にすることで「あってもなくてもいい」を表現してる。`.unwrap_or()` でデフォルト値を設定してる。

**レイ**: なるほど。`#[derive(Deserialize)]` ってなに？

**ユイ**: serdeの機能で、クエリストリングを自動的にRustの構造体に変換してくれるんだ。

```
?page=2&limit=20
      ↓ Deserialize
Pagination { page: Some(2), limit: Some(20) }
```

**レイ**: FastAPI でもこういう感じだよね：

```python
@app.get("/users")
async def list_users(page: int = 1, limit: int = 10):
    return f"Page: {page}, Limit: {limit}"
```

**ユイ**: そうそう。FastAPIも型ヒントで同じことやってるね。ただRustは**コンパイル時**に型チェックされるのが違い。

### JSONボディ

**レイ**: POST リクエストでJSONを受け取る場合は？

**ユイ**: `Json` エクストラクタを使う。リクエストとレスポンスの構造体を定義してみよう：

```rust
use axum::Json;
use serde::{Deserialize, Serialize};

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Serialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

async fn create_user(Json(payload): Json<CreateUser>) -> Json<User> {
    // payloadはCreateUser型
    let user = User {
        id: 1,
        name: payload.name,
        email: payload.email,
    };
    Json(user)  // 自動的にJSONレスポンスに変換
}
```

**レイ**: `Deserialize` と `Serialize` の両方が出てきたね。

**ユイ**: そう。

- `Deserialize` = JSON → Rust構造体（リクエストボディの読み取り）
- `Serialize` = Rust構造体 → JSON（レスポンスボディの書き込み）

だよ。リクエストの流れを図にするとこう：

```
リクエスト（POST /users）:
{
    "name": "Alice",
    "email": "alice@example.com"
}
      ↓
Json<CreateUser>エクストラクタ
      ↓
CreateUser { name: "Alice", email: "alice@example.com" }
      ↓
ハンドラ関数でUser構造体を作成
      ↓
Json<User>を返す
      ↓
レスポンス:
{
    "id": 1,
    "name": "Alice",
    "email": "alice@example.com"
}
```

**レイ**: すごい！型安全なまま双方向変換できるんだ。Express だと：

```javascript
app.post('/users', (req, res) => {
    const { name, email } = req.body;  // 型チェックなし
    res.json({ id: 1, name, email });
});
```

**ユイ**: TypeScript + Zod とか使えば型チェックできるけど、Rustは**言語レベル**で保証されてるのが強いよね。

### ヘッダー

**レイ**: HTTPヘッダーも取得できる？

**ユイ**: もちろん。`HeaderMap` を使う：

```rust
use axum::http::HeaderMap;

async fn with_headers(headers: HeaderMap) -> String {
    let user_agent = headers
        .get("user-agent")
        .and_then(|v| v.to_str().ok())
        .unwrap_or("Unknown");

    format!("User-Agent: {}", user_agent)
}
```

**レイ**: ヘッダーの取得ってちょっと複雑だね。

**ユイ**: うん。ヘッダー値は `HeaderValue` 型で、文字列じゃないから `.to_str()` で変換してる。`and_then()` はOptionのメソッドだね（Chapter 05参照）。

---

## 9-6. レスポンスをカスタマイズする

**レイ**: ところで、ステータスコードを変えたい場合はどうするの？404とか500とか。

**ユイ**: `StatusCode` を返せばいいよ：

```rust
use axum::http::StatusCode;

async fn not_found() -> StatusCode {
    StatusCode::NOT_FOUND  // 404
}
```

**レイ**: ステータスコードとボディを両方返したい場合は？

**ユイ**: タプルで返せる：

```rust
async fn created() -> (StatusCode, Json<User>) {
    let user = User {
        id: 1,
        name: "Alice".into(),
        email: "alice@example.com".into()
    };
    (StatusCode::CREATED, Json(user))  // 201 + JSONボディ
}
```

**レイ**: タプルで返すだけでいいの？便利だね。

**ユイ**: Axumは**タプルを自動的にHTTPレスポンスに変換**してくれるんだ。順番に：

1. `StatusCode` → HTTPステータスコード
2. `Json<User>` → レスポンスボディ

って感じ。

**レイ**: ヘッダーも一緒に返せる？

**ユイ**: できるよ。ちょっと複雑になるけど：

```rust
async fn custom_response() -> (StatusCode, [(axum::http::HeaderName, &'static str); 1], &'static str) {
    use axum::http::header::CONTENT_TYPE;
    (
        StatusCode::OK,
        [(CONTENT_TYPE, "text/plain")],
        "Custom response body",
    )
}
```

**レイ**: うーん、型が長い...

**ユイ**: そうだね。実際は `impl IntoResponse` を使ってもっとスマートに書くことが多いよ。

### IntoResponse トレイト

**レイ**: `IntoResponse` ってさっきから出てくるけど、何？

**ユイ**: Axumがレスポンスに変換できる型が実装してるトレイトだよ。これを実装してれば、ハンドラの戻り値として使える：

```rust
use axum::response::{IntoResponse, Response};

// 以下はすべてIntoResponseを実装している
async fn handler1() -> &'static str { "string" }
async fn handler2() -> String { "String".to_string() }
async fn handler3() -> Json<User> { Json(user) }
async fn handler4() -> StatusCode { StatusCode::OK }
async fn handler5() -> (StatusCode, Json<User>) {
    (StatusCode::OK, Json(user))
}
```

**レイ**: なるほど。だからいろんな型を返せるんだね。

---

## 9-7. エラーハンドリング

**レイ**: 実際のアプリだとエラー処理も重要だよね。Axumではどうやるの？

**ユイ**: カスタムエラー型を作って、`IntoResponse` を実装するのが定石だよ。

```rust
use axum::{
    http::StatusCode,
    response::{IntoResponse, Response},
    Json,
};
use serde_json::json;

enum AppError {
    NotFound,
    BadRequest(String),
    InternalError,
}

impl IntoResponse for AppError {
    fn into_response(self) -> Response {
        let (status, message) = match self {
            AppError::NotFound => (StatusCode::NOT_FOUND, "Not found"),
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.leak()),
            AppError::InternalError => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error"),
        };

        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}
```

**レイ**: `IntoResponse` を実装すると、エラー型をレスポンスに変換できるのか。

**ユイ**: そう。そうすると `Result` でエラーを返せるようになる：

```rust
async fn get_user(Path(id): Path<u32>) -> Result<Json<User>, AppError> {
    if id == 0 {
        return Err(AppError::BadRequest("Invalid ID".into()));
    }

    // データベースから検索（仮）
    let user = find_user(id).ok_or(AppError::NotFound)?;

    Ok(Json(user))
}
```

**レイ**: `Result` の `Err` を返すだけでHTTPエラーレスポンスになるんだ！

**ユイ**: そう。Axumが自動的に `IntoResponse::into_response()` を呼んでHTTPレスポンスに変換してくれる。

**レイ**: FastAPI だとこういう感じだね：

```python
from fastapi import HTTPException

@app.get("/users/{id}")
async def get_user(id: int):
    if id == 0:
        raise HTTPException(status_code=400, detail="Invalid ID")
    # ...
    raise HTTPException(status_code=404, detail="Not found")
```

**ユイ**: そうだね。FastAPIは例外を投げるけど、Rustは `Result` で型安全にエラーを扱う。どっちのパスも型で表現されてるのがRustのいいところ。

---

## 9-8. 状態の共有（State）

**レイ**: APIサーバーって、データベース接続とかアプリ全体で共有したいデータがあるよね。Axumではどうやるの？

**ユイ**: `State` エクストラクタを使うよ。まず状態を表す構造体を作る：

```rust
use axum::extract::State;
use std::sync::Arc;

struct AppState {
    db_pool: DatabasePool,  // 仮のデータベースプール
}

async fn list_users(State(state): State<Arc<AppState>>) -> Json<Vec<User>> {
    let users = state.db_pool.get_users().await;
    Json(users)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db_pool: DatabasePool::new().await,
    });

    let app = Router::new()
        .route("/users", get(list_users))
        .with_state(state);  // 状態を登録

    // ...
}
```

**レイ**: `Arc` ってなに？また出てきた。

**ユイ**: **Arc (Atomic Reference Counted)** は、複数のスレッドで安全にデータを共有するための型だよ。図で説明するね：

```
┌─────────────────────────────────────────────────────────────────┐
│  なぜ Arc が必要か                                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Axumは複数のリクエストを並列処理する                            │
│                                                                 │
│  リクエストA ─────┐                                              │
│  リクエストB ─────┼──→ 同じAppStateにアクセス                    │
│  リクエストC ─────┘                                              │
│                                                                 │
│  Arc (Atomic Reference Counted):                                │
│  - 複数スレッドで安全に共有可能                                   │
│  - 参照カウントでメモリ管理                                       │
│  - 読み取り専用データの共有に最適                                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: なるほど。複数のリクエストが同時に同じ状態にアクセスするから、安全に共有する仕組みが必要なのか。

**ユイ**: そう。Express だと `app.locals` とかに保存するけど、JavaScriptはシングルスレッドだから排他制御を意識しなくていい。Rustはマルチスレッドだから明示的に `Arc` を使うんだ。

**レイ**: もし状態を変更したい場合は？例えば、データベースじゃなくてメモリ上にユーザーリストを持つとか。

**ユイ**: その場合は `RwLock` を使う：

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

type SharedUsers = Arc<RwLock<Vec<User>>>;

async fn add_user(
    State(users): State<SharedUsers>,
    Json(user): Json<User>,
) -> StatusCode {
    let mut users = users.write().await;  // 書き込みロック取得
    users.push(user);
    StatusCode::CREATED
}

async fn list_users(State(users): State<SharedUsers>) -> Json<Vec<User>> {
    let users = users.read().await;  // 読み取りロック取得
    Json(users.clone())
}
```

**レイ**: `RwLock` って何？

**ユイ**: **Read-Write Lock** の略で、読み取りと書き込みを分けてロックする仕組みだよ：

- `.read().await` = 読み取りロック（複数同時OK）
- `.write().await` = 書き込みロック（排他的、1つだけ）

**レイ**: 複数のリクエストが同時に読み取りできるけど、書き込み中は排他ロックがかかるってことか。

**ユイ**: そのとおり！これでデータ競合を防げる。

**レイ**: `.clone()` してるのはなんで？

**ユイ**: `users.read().await` で取得したロックガードは関数を抜けるまで保持されるんだ。`.clone()` してデータをコピーすることで、早くロックを解放できる。

---

## 9-9. ミドルウェア

**レイ**: Expressの `app.use()` みたいなミドルウェアは使える？

**ユイ**: もちろん。Axumは `tower` っていうミドルウェアエコシステムを使うよ。

### ロギング

**ユイ**: まずロギングから。`TraceLayer` を使う：

```rust
use tower_http::trace::TraceLayer;
use tracing_subscriber;

#[tokio::main]
async fn main() {
    // トレーシングを初期化
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/", get(hello))
        .layer(TraceLayer::new_for_http());  // ロギングミドルウェア

    // ...
}
```

**レイ**: これで何がログに出るの？

**ユイ**: リクエストとレスポンスの情報が自動でログに出るよ。Expressの `morgan` みたいなもん。

### CORS

**レイ**: フロントエンドから叩くなら CORS も必要だよね。

**ユイ**: `CorsLayer` を使う：

```rust
use tower_http::cors::{CorsLayer, Any};

let cors = CorsLayer::new()
    .allow_origin(Any)
    .allow_methods(Any)
    .allow_headers(Any);

let app = Router::new()
    .route("/", get(hello))
    .layer(cors);
```

**レイ**: Express の `cors` パッケージとほぼ同じ感じだね。

### カスタム認証ミドルウェア

**レイ**: 自分で認証ミドルウェアとか作れる？

**ユイ**: できるよ。`middleware::from_fn` を使う：

```rust
use axum::{
    middleware::{self, Next},
    http::Request,
    response::Response,
};

async fn auth_middleware(
    request: Request<axum::body::Body>,
    next: Next,
) -> Result<Response, StatusCode> {
    // Authorizationヘッダーをチェック
    let auth_header = request
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok());

    match auth_header {
        Some(token) if token.starts_with("Bearer ") => {
            // トークン検証（実際はJWTデコードなど）
            Ok(next.run(request).await)  // 次のハンドラへ
        }
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

// 使用
let app = Router::new()
    .route("/protected", get(protected_handler))
    .layer(middleware::from_fn(auth_middleware));
```

**レイ**: `next.run(request).await` で次のハンドラに処理を渡すのか。Express の `next()` と同じだね。

**ユイ**: そう。ミドルウェアの処理順序はこうなるよ：

```
リクエスト
    ↓
┌─────────────┐
│ CORS        │  ← .layer() の逆順で実行
├─────────────┤
│ Trace       │
├─────────────┤
│ Auth        │
└─────────────┘
    ↓
ハンドラ
    ↓
（レスポンスは逆順）
```

**レイ**: `.layer()` で追加した順と逆になるの？

**ユイ**: そう。最後に追加したものが最初に実行される。ちょっと直感に反するけど、tower の仕様なんだ。

---

## 9-10. 完全な CRUD API を作ってみよう

**レイ**: じゃあ、これまで学んだことを総動員して、ちゃんと動くCRUD APIを作ってみようよ。

**ユイ**: いいね。ユーザーの作成・取得・更新・削除ができるAPIを作ろう。データはメモリ上に持つよ。

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{get, post, put, delete},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;

// データ構造
#[derive(Clone, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct CreateUser {
    name: String,
    email: String,
}

#[derive(Deserialize)]
struct UpdateUser {
    name: Option<String>,
    email: Option<String>,
}

// 共有状態
type AppState = Arc<RwLock<Vec<User>>>;

// ハンドラ
async fn list_users(State(users): State<AppState>) -> Json<Vec<User>> {
    let users = users.read().await;
    Json(users.clone())
}

async fn get_user(
    State(users): State<AppState>,
    Path(id): Path<u32>,
) -> Result<Json<User>, StatusCode> {
    let users = users.read().await;
    users
        .iter()
        .find(|u| u.id == id)
        .cloned()
        .map(Json)
        .ok_or(StatusCode::NOT_FOUND)
}

async fn create_user(
    State(users): State<AppState>,
    Json(payload): Json<CreateUser>,
) -> (StatusCode, Json<User>) {
    let mut users = users.write().await;
    let id = users.len() as u32 + 1;
    let user = User {
        id,
        name: payload.name,
        email: payload.email,
    };
    users.push(user.clone());
    (StatusCode::CREATED, Json(user))
}

async fn update_user(
    State(users): State<AppState>,
    Path(id): Path<u32>,
    Json(payload): Json<UpdateUser>,
) -> Result<Json<User>, StatusCode> {
    let mut users = users.write().await;
    let user = users
        .iter_mut()
        .find(|u| u.id == id)
        .ok_or(StatusCode::NOT_FOUND)?;

    if let Some(name) = payload.name {
        user.name = name;
    }
    if let Some(email) = payload.email {
        user.email = email;
    }

    Ok(Json(user.clone()))
}

async fn delete_user(
    State(users): State<AppState>,
    Path(id): Path<u32>,
) -> StatusCode {
    let mut users = users.write().await;
    let len_before = users.len();
    users.retain(|u| u.id != id);

    if users.len() < len_before {
        StatusCode::NO_CONTENT
    } else {
        StatusCode::NOT_FOUND
    }
}

#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(RwLock::new(vec![]));

    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server running on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
```

**レイ**: おお、結構しっかりしたAPIだ！動かしてみていい？

**ユイ**: どうぞ。`cargo run` して、別のターミナルから `curl` で叩いてみて。

**レイ**: まずユーザー作成。

```bash
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'
```

**レイ**: おお、返ってきた！

```json
{"id":1,"name":"Alice","email":"alice@example.com"}
```

**レイ**: 一覧取得もやってみる。

```bash
curl http://localhost:3000/users
```

```json
[{"id":1,"name":"Alice","email":"alice@example.com"}]
```

**レイ**: 更新もできる？

```bash
curl -X PUT http://localhost:3000/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Smith"}'
```

```json
{"id":1,"name":"Alice Smith","email":"alice@example.com"}
```

**レイ**: 完璧！削除も試す。

```bash
curl -X DELETE http://localhost:3000/users/1
```

**レイ**: ステータス 204 が返ってきた。もう一回 GET してみる。

```bash
curl http://localhost:3000/users
```

```json
[]
```

**レイ**: 空になってる！ちゃんと動いてるね。

**ユイ**: いい感じでしょ？これで基本的なCRUD APIは作れるようになったね。

---

## 9-11. TypeScript/Python エンジニアのための移行ガイド

**レイ**: Axumの基本はわかったけど、Express や FastAPI から移行するときのポイントをまとめておきたいな。

**ユイ**: いいね。いくつか重要な違いをまとめよう。

### 1. 型安全なパラメータ

**ユイ**: Expressだとこうだよね：

```typescript
// Express: 実行時エラーのリスク
app.get('/users/:id', (req, res) => {
    const id = parseInt(req.params.id);  // NaNの可能性
    if (isNaN(id)) {
        return res.status(400).send('Invalid ID');
    }
    // ...
});
```

**レイ**: `parseInt()` が失敗するかもしれないから、エラーチェックが必要。

**ユイ**: Axumだと自動でやってくれる：

```rust
// Axum: コンパイル時に保証
async fn get_user(Path(id): Path<u32>) -> ...
// 無効なIDは自動的に400エラー
```

**レイ**: TypeScript で型ガードをたくさん書いてた手間が省けるのか。

### 2. 非同期処理

**ユイ**: FastAPI だとこうだよね：

```python
# FastAPI: async defで非同期
@app.get("/users")
async def list_users():
    users = await db.get_users()
    return users
```

**ユイ**: Axumも同様：

```rust
// Axum: 同様にasync fn
async fn list_users(State(db): State<DbPool>) -> Json<Vec<User>> {
    let users = db.get_users().await;
    Json(users)
}
```

**レイ**: `await` の位置が違うくらいで、構造は似てるね。

### 3. エラーハンドリング

**ユイ**: Express だと `try-catch`：

```javascript
app.get('/users/:id', async (req, res) => {
    try {
        const user = await db.getUser(req.params.id);
        if (!user) return res.status(404).json({ error: 'Not found' });
        res.json(user);
    } catch (e) {
        res.status(500).json({ error: 'Internal error' });
    }
});
```

**ユイ**: Axumは `Result` 型：

```rust
async fn get_user(Path(id): Path<u32>) -> Result<Json<User>, AppError> {
    let user = db.get_user(id).await.map_err(|_| AppError::InternalError)?;
    user.map(Json).ok_or(AppError::NotFound)
}
```

**レイ**: `try-catch` の代わりに `?` 演算子を使うのか。

**ユイ**: そう。Rustは**エラーを型で表現**するから、コンパイラがエラー処理の漏れを検出してくれるよ。

### よくあるエラー

**レイ**: 初心者がハマりそうなエラーってある？

**ユイ**: あるある。2つ紹介するね。

**エラー1: エクストラクタの順序**

```rust
// JSONは最後に配置（ボディを消費するため）
async fn handler(
    State(state): State<AppState>,  // OK: 先
    Path(id): Path<u32>,            // OK: 先
    Json(body): Json<Body>,         // 必ず最後
) -> ...
```

**レイ**: `Json` は必ず最後なの？

**ユイ**: そう。`Json` はリクエストボディを読み取るから、一度だけしか使えないんだ。だから最後に配置する必要がある。

**エラー2: Cloneの忘れ**

```rust
async fn list_users(State(users): State<AppState>) -> Json<Vec<User>> {
    let users = users.read().await;
    Json(users.clone())  // ← clone()が必要
    // Json(*users) だとムーブエラー
}
```

**レイ**: `.clone()` を忘れるとどうなるの？

**ユイ**: コンパイルエラーになる。`RwLockReadGuard` はムーブできないから、中身を `clone()` してコピーする必要があるんだ。

---

## まとめ

**レイ**: AxumでのWeb開発、めっちゃ楽しかった！

**ユイ**: いい感じだったね。最後にExpressやFastAPIとの対応をまとめておこう。

| 概念 | Express | FastAPI | Axum |
|-----|---------|---------|------|
| ルーティング | `app.get()` | デコレータ | `Router::new().route()` |
| パラメータ | `req.params` | 関数引数 | `Path<T>` |
| クエリ | `req.query` | 関数引数 | `Query<T>` |
| ボディ | `req.body` | Pydanticモデル | `Json<T>` |
| 状態 | `app.locals` | Depends | `State<T>` |
| ミドルウェア | `app.use()` | middleware | `.layer()` |

**レイ**: この章のポイントをまとめると：

**ユイ**: こんな感じかな：

- **エクストラクタ** = 型安全なリクエスト処理（`Path`, `Query`, `Json`）
- **IntoResponse** = 様々な型をレスポンスに変換
- **State** = `Arc<T>` で共有状態を管理
- **Router** = ルーティングとネスト
- **tower** = ミドルウェアエコシステム
- **すべて型安全** = コンパイル時にエラー検出

**レイ**: Rustでも普通にWeb開発できるんだね。最初は難しそうって思ってたけど。

**ユイ**: 慣れればExpressやFastAPIと同じくらい生産的だよ。しかもパフォーマンスと型安全性が段違いに良い。

**レイ**: 次は何やるの？

**ユイ**: 次はSolana/Anchorかな。ブロックチェーンのスマートコントラクトをRustで書くんだ。

**レイ**: え、ブロックチェーン！？めっちゃ興味ある。

**ユイ**: この章で学んだ非同期処理、型安全性、エラーハンドリングの知識が全部活きるから、楽しみにしててね。

---

## 次のステップ

[Chapter 10: Solana/Anchor](10-solana-anchor.md) では、RustでSolanaブロックチェーンのスマートコントラクト（Program）を開発する方法を学びます。この章で学んだ非同期処理、型安全性、エラーハンドリングの知識がすべて活きます。
