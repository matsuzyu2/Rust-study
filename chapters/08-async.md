# Chapter 08: 非同期処理

> **この章で学ぶこと**: async/await、Future、tokio — Rustでの非同期プログラミング

---

## なぜ非同期処理が必要か

Web開発では、I/O待ち（ネットワーク、ファイル、DB）が頻繁に発生します。同期処理ではI/O待ちの間スレッドがブロックされ、リソースを無駄にします。

| モデル | 説明 | 例 |
|-------|------|-----|
| 同期（ブロッキング） | 処理が完了するまで待つ | 従来のPython |
| マルチスレッド | 並列に処理 | Java、Go |
| 非同期（ノンブロッキング） | 待ち時間に他の処理 | Node.js、Python asyncio |

Rustは**非同期処理**をサポートし、少ないリソースで多数のI/Oを効率的に処理できます。

---

## async/await の基本

```rust
async fn hello() -> String {
    String::from("Hello, async world!")
}

#[tokio::main]
async fn main() {
    let result = hello().await;
    println!("{}", result);
}
```

🔄 **比較（JavaScript）**:
```javascript
async function hello() {
    return "Hello, async world!";
}

async function main() {
    const result = await hello();
    console.log(result);
}
```

🔄 **比較（Python）**:
```python
import asyncio

async def hello():
    return "Hello, async world!"

async def main():
    result = await hello()
    print(result)

asyncio.run(main())
```

---

## Future — Rustの非同期の基盤

Rustの`async fn`は**Future**を返します。

```rust
use std::future::Future;

// この2つは同等
async fn hello() -> String {
    String::from("Hello")
}

fn hello_explicit() -> impl Future<Output = String> {
    async {
        String::from("Hello")
    }
}
```

### 重要: Futureは遅延評価

**Rustの非同期は遅延評価**です。`await`するまで何も実行されません。

```rust
async fn expensive_computation() -> i32 {
    println!("計算開始!");  // awaitされるまで実行されない
    42
}

#[tokio::main]
async fn main() {
    let future = expensive_computation();  // まだ何も起きない
    println!("Futureを作成");
    
    let result = future.await;  // ここで実行される
    println!("結果: {}", result);
}
```

🔄 **比較（JavaScript）**: JavaScriptのPromiseは作成時に即座に実行開始されます。

```javascript
const promise = expensiveComputation(); // 即座に実行開始
console.log("Promiseを作成");
const result = await promise;
```

---

## tokio ランタイム

Rustの`async/await`は**ランタイム**が必要です。標準ライブラリには含まれず、外部クレートを使います。

最も人気のあるランタイムは**tokio**です。

### セットアップ

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

### 基本的な使い方

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]  // tokioランタイムを起動
async fn main() {
    println!("開始");
    sleep(Duration::from_secs(1)).await;
    println!("1秒後");
}
```

### ランタイムを手動で作成

```rust
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    
    rt.block_on(async {
        println!("非同期コード");
    });
}
```

---

## 並行実行

### join! — 複数のFutureを同時実行

```rust
use tokio::time::{sleep, Duration};

async fn task1() -> String {
    sleep(Duration::from_secs(1)).await;
    String::from("Task 1 完了")
}

async fn task2() -> String {
    sleep(Duration::from_secs(2)).await;
    String::from("Task 2 完了")
}

#[tokio::main]
async fn main() {
    // 並行実行（合計2秒で完了）
    let (result1, result2) = tokio::join!(task1(), task2());
    
    println!("{}", result1);
    println!("{}", result2);
}
```

🔄 **比較（JavaScript）**:
```javascript
const [result1, result2] = await Promise.all([task1(), task2()]);
```

🔄 **比較（Python）**:
```python
result1, result2 = await asyncio.gather(task1(), task2())
```

### select! — 最初に完了したものを取得

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(1)) => {
            println!("1秒経過");
        }
        _ = sleep(Duration::from_secs(2)) => {
            println!("2秒経過");  // これは実行されない
        }
    }
}
```

🔄 **比較（JavaScript）**:
```javascript
await Promise.race([sleep(1000), sleep(2000)]);
```

---

## spawn — タスクの生成

バックグラウンドでタスクを実行：

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // タスクをspawn（バックグラウンドで実行）
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });
    
    println!("タスクを生成しました");
    
    // 結果を待つ
    let result = handle.await.unwrap();
    println!("結果: {}", result);
}
```

### 複数タスクの生成

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    let mut handles = vec![];
    
    for i in 0..5 {
        let handle = tokio::spawn(async move {
            sleep(Duration::from_millis(100 * i)).await;
            i * 2
        });
        handles.push(handle);
    }
    
    for handle in handles {
        let result = handle.await.unwrap();
        println!("結果: {}", result);
    }
}
```

---

## 非同期I/O

### ファイル操作

```rust
use tokio::fs;
use tokio::io::{AsyncReadExt, AsyncWriteExt};

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // ファイル書き込み
    fs::write("hello.txt", "Hello, Tokio!").await?;
    
    // ファイル読み込み
    let content = fs::read_to_string("hello.txt").await?;
    println!("{}", content);
    
    Ok(())
}
```

### HTTPリクエスト（reqwestクレート）

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
```

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct User {
    id: i32,
    name: String,
}

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let user: User = reqwest::get("https://api.example.com/users/1")
        .await?
        .json()
        .await?;
    
    println!("{:?}", user);
    Ok(())
}
```

🔄 **比較（JavaScript）**:
```javascript
const response = await fetch("https://api.example.com/users/1");
const user = await response.json();
```

---

## 非同期ストリーム

連続的なデータを扱う場合、**Stream**を使います。

```rust
use tokio_stream::StreamExt;
use tokio::time::{interval, Duration};

#[tokio::main]
async fn main() {
    let mut interval = interval(Duration::from_millis(100));
    let mut count = 0;
    
    while count < 5 {
        interval.tick().await;
        println!("Tick {}", count);
        count += 1;
    }
}
```

### async-streamクレート

```rust
use async_stream::stream;
use tokio_stream::StreamExt;

fn number_stream() -> impl tokio_stream::Stream<Item = i32> {
    stream! {
        for i in 0..5 {
            tokio::time::sleep(Duration::from_millis(100)).await;
            yield i;
        }
    }
}

#[tokio::main]
async fn main() {
    let mut stream = number_stream();
    
    while let Some(n) = stream.next().await {
        println!("{}", n);
    }
}
```

🔄 **比較（JavaScript）**:
```javascript
async function* numberStream() {
    for (let i = 0; i < 5; i++) {
        await sleep(100);
        yield i;
    }
}
```

---

## エラーハンドリング

非同期関数でも`Result`を使います：

```rust
use tokio::fs;

async fn read_config() -> Result<String, std::io::Error> {
    let content = fs::read_to_string("config.toml").await?;
    Ok(content)
}

#[tokio::main]
async fn main() {
    match read_config().await {
        Ok(config) => println!("Config: {}", config),
        Err(e) => eprintln!("Error: {}", e),
    }
}
```

### anyhowとの組み合わせ

```rust
use anyhow::{Context, Result};
use tokio::fs;

async fn read_config() -> Result<String> {
    let content = fs::read_to_string("config.toml")
        .await
        .context("設定ファイルが読み込めません")?;
    Ok(content)
}
```

---

## 同期コードとの連携

### spawn_blocking — 同期コードを非同期から呼ぶ

```rust
use tokio::task::spawn_blocking;

fn heavy_computation() -> i32 {
    // CPUバウンドな同期処理
    std::thread::sleep(std::time::Duration::from_secs(1));
    42
}

#[tokio::main]
async fn main() {
    // 別スレッドで実行
    let result = spawn_blocking(|| heavy_computation()).await.unwrap();
    println!("結果: {}", result);
}
```

### block_on — 非同期コードを同期から呼ぶ

```rust
fn main() {
    let rt = tokio::runtime::Runtime::new().unwrap();
    
    let result = rt.block_on(async {
        some_async_function().await
    });
}
```

---

## チャネル — タスク間通信

### mpsc（多対一）

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let (tx, mut rx) = mpsc::channel(32);
    
    // 送信側
    tokio::spawn(async move {
        for i in 0..5 {
            tx.send(i).await.unwrap();
        }
    });
    
    // 受信側
    while let Some(value) = rx.recv().await {
        println!("受信: {}", value);
    }
}
```

### oneshot（一対一、一回限り）

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();
    
    tokio::spawn(async move {
        tx.send("完了!").unwrap();
    });
    
    let result = rx.await.unwrap();
    println!("{}", result);
}
```

---

## 実践パターン: 並列API呼び出し

```rust
use reqwest::Client;
use serde::Deserialize;
use anyhow::Result;

#[derive(Debug, Deserialize)]
struct User {
    id: i32,
    name: String,
}

async fn fetch_user(client: &Client, id: i32) -> Result<User> {
    let url = format!("https://api.example.com/users/{}", id);
    let user = client.get(&url).send().await?.json().await?;
    Ok(user)
}

#[tokio::main]
async fn main() -> Result<()> {
    let client = Client::new();
    
    // 複数ユーザーを並行取得
    let ids = vec![1, 2, 3, 4, 5];
    let futures: Vec<_> = ids.iter()
        .map(|&id| fetch_user(&client, id))
        .collect();
    
    let results = futures::future::join_all(futures).await;
    
    for result in results {
        match result {
            Ok(user) => println!("{:?}", user),
            Err(e) => eprintln!("Error: {}", e),
        }
    }
    
    Ok(())
}
```

---

## まとめ

| 概念 | JavaScript | Python | Rust |
|-----|------------|--------|------|
| 非同期関数 | `async function` | `async def` | `async fn` |
| 待機 | `await` | `await` | `.await` |
| Promise/Future | Promise | Coroutine | Future |
| ランタイム | 組み込み | asyncio | tokio（外部） |
| 並行実行 | `Promise.all` | `asyncio.gather` | `tokio::join!` |
| レース | `Promise.race` | `asyncio.wait` | `tokio::select!` |
| 遅延評価 | ❌（即時実行） | ❌（即時実行） | ✅ |

🎯 **ポイント**:
- **Futureは遅延評価** — `await`するまで実行されない
- **ランタイムが必要** — tokioが最もポピュラー
- **join!** = 並行実行、**select!** = レース
- **spawn** = バックグラウンドタスク
- **チャネル** = タスク間通信

---

## 次のステップ

[Chapter 09: Web開発](09-web-development.md) では、Axumフレームワークを使ったREST API開発を学びます。Express/FastAPIとの比較を交えて解説します。
