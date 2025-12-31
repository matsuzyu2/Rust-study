# Chapter 04: エラーハンドリング

> **この章で学ぶこと**: Result型、`?`演算子、panic — Rustのエラー処理パターン

---

## Rustのエラー処理哲学

Rustには**例外（Exception）がありません**。代わりに、エラーを**型**で表現します。

🔄 **比較**:

| 言語 | エラー処理方法 | 問題点 |
|-----|--------------|-------|
| Python | `try`/`except` | どの関数が例外を投げるか分からない |
| JavaScript | `try`/`catch` | 同上 + 非同期エラーが複雑 |
| Rust | `Result<T, E>` | **エラーの可能性が型に現れる** |

```python
# Python: read_file()が例外を投げる可能性は型から分からない
def process_file(path: str) -> str:
    content = read_file(path)  # IOError?  PermissionError?
    return content.upper()
```

```rust
// Rust: 戻り値の型がResultなので、エラーの可能性が明確
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?;
    Ok(content.to_uppercase())
}
```

---

## Result型

`Result<T, E>`は成功と失敗を表す列挙型です。

```rust
enum Result<T, E> {
    Ok(T),   // 成功: 値Tを持つ
    Err(E),  // 失敗: エラーEを持つ
}
```

```rust
use std::fs::File;

fn main() {
    let file_result: Result<File, std::io::Error> = File::open("hello.txt");
    
    match file_result {
        Ok(file) => println!("ファイルを開きました: {:?}", file),
        Err(error) => println!("エラー: {}", error),
    }
}
```

🔄 **比較（TypeScript）**:
```typescript
// TypeScriptで同様のパターンを実装する場合
type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

function openFile(path: string): Result<File, Error> {
    try {
        return { ok: true, value: fs.openSync(path) };
    } catch (e) {
        return { ok: false, error: e as Error };
    }
}
```

---

## `?`演算子 — エラーの伝播

エラーを呼び出し元に返したい場合、`?`演算子を使います。

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;  // Errなら即return
    let mut username = String::new();
    file.read_to_string(&mut username)?;         // Errなら即return
    Ok(username)
}
```

`?`の動作：
- `Ok(value)` → `value`を取り出して続行
- `Err(e)` → 即座に`Err(e)`を返す

🔄 **比較（手動でmatch）**:
```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let file_result = File::open("username.txt");
    let mut file = match file_result {
        Ok(f) => f,
        Err(e) => return Err(e),
    };
    
    let mut username = String::new();
    match file.read_to_string(&mut username) {
        Ok(_) => Ok(username),
        Err(e) => Err(e),
    }
}
```

`?`を使うと劇的に短くなります。

### `?`のチェーン

```rust
use std::fs;
use std::io;

fn read_username_from_file() -> Result<String, io::Error> {
    fs::read_to_string("username.txt")
}
// ↑ 実はこれだけでOK（標準ライブラリに便利関数がある）
```

メソッドチェーンでも使えます：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("username.txt")?.read_to_string(&mut username)?;
    Ok(username)
}
```

---

## main関数でのエラー処理

`main`関数も`Result`を返せます。

```rust
use std::fs::File;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("hello.txt")?;
    println!("ファイルを開きました: {:?}", file);
    Ok(())
}
```

`Box<dyn std::error::Error>`は「あらゆるエラー型を受け入れる」という意味です。

---

## カスタムエラー型

実際のプロジェクトでは、独自のエラー型を定義することが多いです。

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    NotFound(String),
    PermissionDenied,
    InvalidInput { message: String },
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::NotFound(name) => write!(f, "Not found: {}", name),
            AppError::PermissionDenied => write!(f, "Permission denied"),
            AppError::InvalidInput { message } => write!(f, "Invalid input: {}", message),
        }
    }
}

impl std::error::Error for AppError {}
```

### thiserrorクレート（実務で便利）

手動での実装は面倒なので、`thiserror`クレートを使うのが一般的です。

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("Not found: {0}")]
    NotFound(String),
    
    #[error("Permission denied")]
    PermissionDenied,
    
    #[error("Invalid input: {message}")]
    InvalidInput { message: String },
    
    #[error("IO error")]
    Io(#[from] std::io::Error),  // io::Errorから自動変換
}
```

---

## anyhowクレート（アプリケーション向け）

ライブラリではなくアプリケーションを書く場合、`anyhow`が便利です。

```rust
use anyhow::{Context, Result};

fn read_config() -> Result<Config> {
    let content = std::fs::read_to_string("config.json")
        .context("設定ファイルが読み込めません")?;
    
    let config: Config = serde_json::from_str(&content)
        .context("JSONパースに失敗しました")?;
    
    Ok(config)
}
```

- `anyhow::Result<T>` = `Result<T, anyhow::Error>`
- `.context()`でエラーにコンテキストを追加
- 異なるエラー型を自動変換

🔄 **比較**:
| 用途 | 推奨クレート |
|-----|-------------|
| ライブラリ開発 | `thiserror`（具体的なエラー型） |
| アプリケーション開発 | `anyhow`（簡便なエラー処理） |

---

## panic! — 回復不能なエラー

どうしても回復できないエラーは`panic!`で処理します。

```rust
fn main() {
    panic!("Something went terribly wrong!");
}
```

```
thread 'main' panicked at 'Something went terribly wrong!', src/main.rs:2:5
```

### いつpanicを使うか

| 状況 | 推奨 |
|-----|------|
| ファイルが開けない | `Result`（回復可能） |
| 配列の範囲外アクセス | `panic`（プログラムのバグ） |
| 不正な設定値 | `Result`または`panic`（状況による） |
| テストの失敗 | `panic`（`assert!`マクロ） |

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("ゼロ除算は許可されていません");
    }
    a / b
}
```

### unwrapとexpect

`Option`や`Result`から値を取り出す際、失敗時にpanicします。

```rust
fn main() {
    let x: Option<i32> = Some(5);
    
    // unwrap: Noneならpanic
    let value = x.unwrap();
    
    // expect: panicメッセージ付き
    let value = x.expect("値がNoneです");
    
    // より安全な方法
    let value = x.unwrap_or(0);           // デフォルト値
    let value = x.unwrap_or_else(|| 0);   // デフォルト値（遅延評価）
}
```

⚠️ **注意**: 本番コードでは`unwrap()`を避け、適切なエラーハンドリングを行いましょう。

---

## Option と Result の変換

```rust
fn main() {
    // Option → Result
    let x: Option<i32> = Some(5);
    let result: Result<i32, &str> = x.ok_or("値がありません");
    
    // Result → Option
    let y: Result<i32, &str> = Ok(10);
    let option: Option<i32> = y.ok();  // Errは捨てられる
}
```

---

## 実践パターン

### パターン1: 早期リターン

```rust
fn process_user(user_id: Option<i32>) -> Result<User, AppError> {
    let id = user_id.ok_or(AppError::InvalidInput {
        message: "user_id is required".into(),
    })?;
    
    let user = find_user(id)?;
    Ok(user)
}
```

### パターン2: 複数のエラーをまとめる

```rust
fn validate_input(name: &str, age: i32) -> Result<(), Vec<String>> {
    let mut errors = Vec::new();
    
    if name.is_empty() {
        errors.push("名前が空です".to_string());
    }
    if age < 0 {
        errors.push("年齢は0以上である必要があります".to_string());
    }
    
    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}
```

### パターン3: map と and_then

```rust
fn main() {
    let result: Result<i32, &str> = Ok(5);
    
    // map: 成功時の値を変換
    let doubled: Result<i32, &str> = result.map(|x| x * 2);
    
    // and_then: 成功時に別のResultを返す処理
    let chained: Result<i32, &str> = result.and_then(|x| {
        if x > 0 {
            Ok(x * 2)
        } else {
            Err("正の数が必要です")
        }
    });
}
```

🔄 **比較（JavaScript Promise）**:
```javascript
promise
    .then(x => x * 2)           // map相当
    .then(x => anotherPromise)  // and_then相当
    .catch(err => handleError); // Err処理
```

---

## エラー処理のベストプラクティス

| ルール | 説明 |
|-------|------|
| `unwrap()`は避ける | テストやプロトタイプ以外では使わない |
| `?`を活用 | エラー伝播を簡潔に |
| エラー型を設計 | ライブラリなら`thiserror`で専用型を定義 |
| コンテキストを追加 | `anyhow::Context`でエラーの原因を明確に |
| 早期リターン | エラーケースを先に処理して正常系を分かりやすく |

---

## まとめ

| 概念 | Python/JS | Rust |
|-----|-----------|------|
| エラー表現 | 例外（Exception） | `Result<T, E>`型 |
| エラー伝播 | `raise`/`throw` | `?`演算子 |
| 値がない | `None`/`null` | `Option<T>` |
| 回復不能エラー | — | `panic!` |
| エラーキャッチ | `try`/`except`/`catch` | `match`/`if let` |

🎯 **ポイント**:
- **エラーは型で表現** — 関数のシグネチャを見ればエラーの可能性が分かる
- **`?`演算子** — エラー伝播を1文字で表現
- **網羅的な処理** — コンパイラがエラー処理忘れを検出
- **panic vs Result** — 回復可能かどうかで使い分け

---

## 次のステップ

[Chapter 05: トレイト・ジェネリクス](05-traits-generics.md) では、Rustの抽象化メカニズムを学びます。TypeScriptの`interface`やPythonの`Protocol`に相当する「トレイト」と、ジェネリクスを解説します。
