# Chapter 09: Web開発

> **この章で学ぶこと**: Axumフレームワーク、ルーティング、ミドルウェア、JSON API — RustでのWeb開発

---

## なぜRustでWeb開発？

| 観点 | Node.js/Express | Python/FastAPI | Rust/Axum |
|-----|-----------------|----------------|-----------|
| パフォーマンス | 中程度 | 中程度 | **非常に高速** |
| メモリ効率 | GCあり | GCあり | **GCなし、効率的** |
| 型安全性 | TypeScriptで改善 | 型ヒント | **コンパイル時に保証** |
| 同時接続 | イベントループ | asyncio | **async + マルチスレッド** |
| 起動時間 | 速い | 普通 | **非常に速い** |

Rustは高負荷なAPIサーバー、マイクロサービス、リアルタイム処理に適しています。

---

## Axumとは

**Axum**はTokioチームが開発するWebフレームワークです。

- **型安全**: エクストラクタで型安全なリクエスト処理
- **非同期ファースト**: tokioベース
- **ミドルウェア**: towerエコシステムと統合
- **エルゴノミクス**: マクロを使わずにシンプルなAPI

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
cargo add tower-http --features cors
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
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user).put(update_user).delete(delete_user));

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    axum::serve(listener, app).await.unwrap();
}
```

### ネストしたルーター

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
        .nest("/users", user_routes())
        .nest("/posts", post_routes());

    // ...
}
```

---

## エクストラクタ

Axumの**エクストラクタ**は、リクエストから型安全にデータを取り出します。

### パスパラメータ

```rust
use axum::extract::Path;

async fn get_user(Path(id): Path<u32>) -> String {
    format!("User ID: {}", id)
}

// 複数パラメータ
async fn get_post(Path((user_id, post_id)): Path<(u32, u32)>) -> String {
    format!("User: {}, Post: {}", user_id, post_id)
}
```

🔄 **比較（Express）**:
```javascript
app.get('/users/:id', (req, res) => {
    const id = req.params.id;  // 文字列、型安全でない
});
```

### クエリパラメータ

```rust
use axum::extract::Query;
use serde::Deserialize;

#[derive(Deserialize)]
struct Pagination {
    page: Option<u32>,
    limit: Option<u32>,
}

async fn list_users(Query(params): Query<Pagination>) -> String {
    let page = params.page.unwrap_or(1);
    let limit = params.limit.unwrap_or(10);
    format!("Page: {}, Limit: {}", page, limit)
}
```

🔄 **比較（FastAPI）**:
```python
@app.get("/users")
async def list_users(page: int = 1, limit: int = 10):
    return f"Page: {page}, Limit: {limit}"
```

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
    let user = User {
        id: 1,
        name: payload.name,
        email: payload.email,
    };
    Json(user)
}
```

🔄 **比較（Express）**:
```javascript
app.post('/users', (req, res) => {
    const { name, email } = req.body;  // 型安全でない
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
    StatusCode::NOT_FOUND
}

async fn created() -> (StatusCode, Json<User>) {
    let user = User { id: 1, name: "Alice".into(), email: "alice@example.com".into() };
    (StatusCode::CREATED, Json(user))
}
```

### カスタムレスポンス

```rust
use axum::response::{IntoResponse, Response};
use axum::http::{StatusCode, header};

async fn custom_response() -> Response {
    (
        StatusCode::OK,
        [(header::CONTENT_TYPE, "text/plain")],
        "Custom response body",
    ).into_response()
}
```

---

## エラーハンドリング

### 自前のエラー型

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
            AppError::BadRequest(msg) => (StatusCode::BAD_REQUEST, msg.as_str()),
            AppError::InternalError => (StatusCode::INTERNAL_SERVER_ERROR, "Internal error"),
        };

        let body = Json(json!({ "error": message }));
        (status, body).into_response()
    }
}

async fn get_user(Path(id): Path<u32>) -> Result<Json<User>, AppError> {
    if id == 0 {
        return Err(AppError::BadRequest("Invalid ID".into()));
    }
    
    // ユーザーを検索...
    Err(AppError::NotFound)
}
```

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

アプリケーション全体で共有するデータ：

```rust
use axum::extract::State;
use std::sync::Arc;
use tokio::sync::RwLock;

struct AppState {
    db: DatabasePool,
    config: Config,
}

async fn list_users(State(state): State<Arc<AppState>>) -> Json<Vec<User>> {
    let users = state.db.get_users().await;
    Json(users)
}

#[tokio::main]
async fn main() {
    let state = Arc::new(AppState {
        db: DatabasePool::new().await,
        config: Config::load(),
    });

    let app = Router::new()
        .route("/users", get(list_users))
        .with_state(state);

    // ...
}
```

### 可変状態

```rust
use std::sync::Arc;
use tokio::sync::RwLock;

type SharedState = Arc<RwLock<Vec<User>>>;

async fn add_user(
    State(users): State<SharedState>,
    Json(user): Json<User>,
) -> StatusCode {
    let mut users = users.write().await;
    users.push(user);
    StatusCode::CREATED
}
```

---

## ミドルウェア

### ロギング

```rust
use tower_http::trace::TraceLayer;
use tracing_subscriber;

#[tokio::main]
async fn main() {
    tracing_subscriber::fmt::init();

    let app = Router::new()
        .route("/", get(hello))
        .layer(TraceLayer::new_for_http());

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

### 認証ミドルウェア

```rust
use axum::{
    middleware::{self, Next},
    http::Request,
    response::Response,
};

async fn auth_middleware<B>(
    request: Request<B>,
    next: Next<B>,
) -> Result<Response, StatusCode> {
    let auth_header = request
        .headers()
        .get("Authorization")
        .and_then(|v| v.to_str().ok());

    match auth_header {
        Some(token) if token.starts_with("Bearer ") => {
            // トークン検証...
            Ok(next.run(request).await)
        }
        _ => Err(StatusCode::UNAUTHORIZED),
    }
}

let app = Router::new()
    .route("/protected", get(protected_handler))
    .layer(middleware::from_fn(auth_middleware));
```

---

## 完全な例: CRUD API

```rust
use axum::{
    extract::{Path, State},
    http::StatusCode,
    routing::{get, post},
    Json, Router,
};
use serde::{Deserialize, Serialize};
use std::sync::Arc;
use tokio::sync::RwLock;

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

type AppState = Arc<RwLock<Vec<User>>>;

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

#[tokio::main]
async fn main() {
    let state: AppState = Arc::new(RwLock::new(vec![]));

    let app = Router::new()
        .route("/users", get(list_users).post(create_user))
        .route("/users/:id", get(get_user))
        .with_state(state);

    let listener = tokio::net::TcpListener::bind("0.0.0.0:3000").await.unwrap();
    println!("Server running on http://localhost:3000");
    axum::serve(listener, app).await.unwrap();
}
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

🎯 **ポイント**:
- **エクストラクタ** = 型安全なリクエスト処理
- **Router** = ルーティングとミドルウェアの構築
- **State** = アプリケーション状態の共有
- **IntoResponse** = カスタムレスポンス型
- **tower** = ミドルウェアエコシステム

---

## 次のステップ

[Chapter 10: Solana/Anchor](10-solana-anchor.md) では、RustでSolanaブロックチェーンのスマートコントラクト（Program）を開発する方法を学びます。
