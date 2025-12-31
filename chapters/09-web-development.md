# Chapter 09: Web開発

> **この章で学ぶこと**: Axumフレームワーク、ルーティング、ミドルウェア、JSON API — RustでのWeb開発

> 📖 **記号リファレンス**: この章では `async`, `await`, `Arc`, `impl IntoResponse` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。また、非同期処理の復習には [Chapter 08: 非同期処理](08-async.md) を参照してください。

---

## 🎯 なぜRustでWeb開発？

| 観点 | Node.js/Express | Python/FastAPI | Rust/Axum |
|-----|-----------------|----------------|-----------|
| パフォーマンス | 中程度 | 中程度 | **非常に高速** |
| メモリ効率 | GCあり | GCあり | **GCなし、効率的** |
| 型安全性 | TypeScriptで改善 | 型ヒント | **コンパイル時に保証** |
| 同時接続 | イベントループ | asyncio | **async + マルチスレッド** |
| 起動時間 | 速い | 普通 | **非常に速い** |

**Rustが向いている用途**:
- 高負荷なAPIサーバー
- マイクロサービス
- リアルタイム処理（WebSocket等）
- 低レイテンシが求められるシステム

---

## Axumとは

**Axum**はTokioチームが開発するWebフレームワークです。

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

🔄 **比較**:
| Rust | Node.js | Python |
|------|---------|--------|
| Axum | Express | FastAPI |
| Actix-web | Koa | Flask |
| Rocket | NestJS | Django |

---

## プロジェクトセットアップ

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

**Cargo.toml**:
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

## Hello World

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

### 各行を読み解く

```rust
use axum::{routing::get, Router};
```
- `Router` = ルーティングを定義する構造体
- `get` = HTTPのGETメソッドに対応する関数

```rust
async fn hello() -> &'static str {
    "Hello, World!"
}
```
- `async fn` = 非同期関数（Chapter 08参照）
- `-> &'static str` = 静的文字列を返す
- **Axumはこの戻り値を自動的にHTTPレスポンスに変換**

```rust
let app = Router::new()
    .route("/", get(hello));
```
- `Router::new()` = 空のルーターを作成
- `.route("/", get(hello))` = パス"/"にGETリクエストが来たら`hello`関数を呼ぶ

```rust
let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
axum::serve(listener, app).await.unwrap();
```
- ポート3000でTCPリスナーを起動
- `axum::serve`でHTTPサーバーを起動

🔄 **比較（Express）**:
```javascript
const express = require('express');
const app = express();

app.get('/', (req, res) => {
    res.send('Hello, World!');
});

app.listen(3000);
```

🔄 **比較（FastAPI）**:
```python
from fastapi import FastAPI

app = FastAPI()

@app.get("/")
async def hello():
    return "Hello, World!"
```

---

## ルーティング

### 基本的なルーティング

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

### `:id`パスパラメータ

```
/users/:id
       ↓
コロン(:)で始まる部分は「パスパラメータ」
/users/123 → id = "123"
/users/456 → id = "456"
```

### ネストしたルーター

大きなアプリケーションではルーターを分割します。

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

---

## エクストラクタ — 型安全なリクエスト処理

Axumの**エクストラクタ**は、リクエストから型安全にデータを取り出す仕組みです。

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

### パスパラメータ

```rust
use axum::extract::Path;

// 単一パラメータ
async fn get_user(Path(id): Path<u32>) -> String {
    format!("User ID: {}", id)
}

// 複数パラメータ
async fn get_post(Path((user_id, post_id)): Path<(u32, u32)>) -> String {
    format!("User: {}, Post: {}", user_id, post_id)
}
// ルート: /users/:user_id/posts/:post_id
```

### エクストラクタの読み解き方

```rust
async fn get_user(Path(id): Path<u32>) -> String {
```

```
Path(id): Path<u32>
 ↓     ↓      ↓
 │     │      └─ 型: Path<u32>（u32のパスパラメータ）
 │     └─ 変数名: id（関数内で使う名前）
 └─ パターン: Pathから中身を取り出す
```

これは「パターンマッチ付きの引数」で、以下と同等です：
```rust
async fn get_user(path: Path<u32>) -> String {
    let id = path.0;  // Path<u32>から中身を取り出す
    format!("User ID: {}", id)
}
```

🔄 **比較（Express）**:
```javascript
app.get('/users/:id', (req, res) => {
    const id = req.params.id;  // 文字列、型チェックなし
});
```

Expressではパラメータは常に文字列で、型変換は手動です。Axumは**自動的に型変換**し、失敗すれば400エラーを返します。

### クエリパラメータ

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

### Deserializeトレイトの重要性

```rust
#[derive(Deserialize)]
struct Pagination { ... }
```

`Deserialize`を実装することで、AxumがクエリストリングをRustの構造体に**自動変換**します。

🔄 **比較（FastAPI）**:
```python
@app.get("/users")
async def list_users(page: int = 1, limit: int = 10):
    return f"Page: {page}, Limit: {limit}"
```

FastAPIも型ヒントで同様のことをしますが、Rustはコンパイル時に型をチェックします。

### JSONボディ

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

### リクエストとレスポンスの流れ

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

🔄 **比較（Express）**:
```javascript
app.post('/users', (req, res) => {
    const { name, email } = req.body;  // 型チェックなし
    res.json({ id: 1, name, email });
});
```

### ヘッダー

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

---

## レスポンス

### ステータスコード

```rust
use axum::http::StatusCode;

async fn not_found() -> StatusCode {
    StatusCode::NOT_FOUND  // 404
}

// ステータスコード + ボディ
async fn created() -> (StatusCode, Json<User>) {
    let user = User { id: 1, name: "Alice".into(), email: "alice@example.com".into() };
    (StatusCode::CREATED, Json(user))  // 201 + JSONボディ
}
```

### タプルでレスポンスを構築

Axumではタプルで複数の情報を返せます：

```rust
// (ステータスコード, ヘッダー, ボディ)
async fn custom_response() -> (StatusCode, [(axum::http::HeaderName, &'static str); 1], &'static str) {
    (
        StatusCode::OK,
        [(axum::http::header::CONTENT_TYPE, "text/plain")],
        "Custom response body",
    )
}
```

### IntoResponse トレイト

Axumは`IntoResponse`トレイトを実装した型をレスポンスとして返せます：

```rust
use axum::response::{IntoResponse, Response};

// 以下はすべてIntoResponseを実装している
async fn handler1() -> &'static str { "string" }
async fn handler2() -> String { "String".to_string() }
async fn handler3() -> Json<User> { Json(user) }
async fn handler4() -> StatusCode { StatusCode::OK }
async fn handler5() -> (StatusCode, Json<User>) { (StatusCode::OK, Json(user)) }
```

---

## エラーハンドリング

### カスタムエラー型

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
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.leak()),  // 簡略化
            AppError::InternalError => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error"),
        };

        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}
```

### ハンドラでのエラー返却

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

### IntoResponseの実装を読み解く

```rust
impl IntoResponse for AppError {
    fn into_response(self) -> Response {
```

- `IntoResponse`トレイトを実装することで、`AppError`をHTTPレスポンスに変換可能に
- `Result<T, E>`で`E`が`IntoResponse`を実装していれば、`Err(e)`を返せる

🔄 **比較（FastAPI）**:
```python
from fastapi import HTTPException

@app.get("/users/{id}")
async def get_user(id: int):
    if id == 0:
        raise HTTPException(status_code=400, detail="Invalid ID")
    raise HTTPException(status_code=404, detail="Not found")
```

---

## 状態の共有（State）

アプリケーション全体で共有するデータ（データベース接続など）を管理します。

### 基本的な使い方

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

### なぜArc<T>が必要か

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

### 可変状態（RwLock）

状態を変更したい場合は`RwLock`を使います：

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

**RwLock（Read-Write Lock）**:
- 複数の読み取り or 1つの書き込み を同時に許可
- `read().await` = 読み取りロック（複数同時OK）
- `write().await` = 書き込みロック（排他的）

---

## ミドルウェア

### ロギング

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

### CORS

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

### カスタム認証ミドルウェア

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

### ミドルウェアの処理順序

```
リクエスト
    ↓
┌─────────────┐
│ CORS        │  ← layer() の逆順で実行
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

---

## 完全な例: CRUD API

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

### テストコマンド

```bash
# ユーザー作成
curl -X POST http://localhost:3000/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# ユーザー一覧
curl http://localhost:3000/users

# ユーザー取得
curl http://localhost:3000/users/1

# ユーザー更新
curl -X PUT http://localhost:3000/users/1 \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice Smith"}'

# ユーザー削除
curl -X DELETE http://localhost:3000/users/1
```

---

## まとめ

| 概念 | Express | FastAPI | Axum |
|-----|---------|---------|------|
| ルーティング | `app.get()` | デコレータ | `Router::new().route()` |
| パラメータ | `req.params` | 関数引数 | `Path<T>` |
| クエリ | `req.query` | 関数引数 | `Query<T>` |
| ボディ | `req.body` | Pydanticモデル | `Json<T>` |
| 状態 | `app.locals` | Depends | `State<T>` |
| ミドルウェア | `app.use()` | middleware | `.layer()` |

🎯 **この章のポイント**:
- **エクストラクタ** = 型安全なリクエスト処理（`Path`, `Query`, `Json`）
- **IntoResponse** = 様々な型をレスポンスに変換
- **State** = `Arc<T>`で共有状態を管理
- **Router** = ルーティングとネスト
- **tower** = ミドルウェアエコシステム
- **すべて型安全** = コンパイル時にエラー検出

---

## TypeScript/Pythonエンジニアのための移行コラム

### Express/FastAPIからの移行ポイント

**1. 型安全なパラメータ**
```typescript
// Express: 実行時エラーのリスク
app.get('/users/:id', (req, res) => {
    const id = parseInt(req.params.id);  // NaNの可能性
});
```

```rust
// Axum: コンパイル時に保証
async fn get_user(Path(id): Path<u32>) -> ...
// 無効なIDは自動的に400エラー
```

**2. 非同期処理**
```python
# FastAPI: async defで非同期
@app.get("/users")
async def list_users():
    users = await db.get_users()
    return users
```

```rust
// Axum: 同様にasync fn
async fn list_users(State(db): State<DbPool>) -> Json<Vec<User>> {
    let users = db.get_users().await;
    Json(users)
}
```

**3. エラーハンドリング**
```javascript
// Express: try-catch
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

```rust
// Axum: Result型
async fn get_user(Path(id): Path<u32>) -> Result<Json<User>, AppError> {
    let user = db.get_user(id).await.map_err(|_| AppError::InternalError)?;
    user.map(Json).ok_or(AppError::NotFound)
}
```

### よくあるエラー

**エラー1: エクストラクタの順序**
```rust
// JSONは最後に配置（ボディを消費するため）
async fn handler(
    State(state): State<AppState>,  // OK: 先
    Path(id): Path<u32>,            // OK: 先
    Json(body): Json<Body>,         // 最後
) -> ...
```

**エラー2: Cloneの忘れ**
```rust
async fn list_users(State(users): State<AppState>) -> Json<Vec<User>> {
    let users = users.read().await;
    Json(users.clone())  // ← clone()が必要
    // Json(*users) だとムーブエラー
}
```

---

## 次のステップ

[Chapter 10: Solana/Anchor](10-solana-anchor.md) では、RustでSolanaブロックチェーンのスマートコントラクト（Program）を開発する方法を学びます。この章で学んだ非同期処理、型安全性、エラーハンドリングの知識がすべて活きます。
