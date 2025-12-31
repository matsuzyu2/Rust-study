# Chapter 04: エラーハンドリング

> **この章で学ぶこと**: Result型、`?`演算子、panic — Rustのエラー処理パターン

> 📖 **記号リファレンス**: この章では `?`, `Ok()`, `Err()`, `unwrap()` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## 🎯 Rustのエラー処理哲学 — なぜ例外がないのか

Rustには**例外（Exception）がありません**。これは意図的な設計です。

### 例外の問題点

```
┌─────────────────────────────────────────────────────────────────┐
│  try-catch の問題点                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. 「どの関数がエラーを投げるか」が分からない                    │
│     def process_file(path):                                     │
│         content = read_file(path)  # IOError? PermissionError?  │
│         data = parse_json(content) # JSONDecodeError?           │
│         return data                                             │
│     → 関数のシグネチャを見てもエラーの可能性が分からない          │
│                                                                 │
│  2. エラーを「catch し忘れる」ことがある                         │
│     → 実行時に初めて問題が発覚                                   │
│     → 本番環境でクラッシュ                                       │
│                                                                 │
│  3. どこまで例外が飛ぶか予測困難                                 │
│     → デバッグが大変                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Rustの解決策

```
┌─────────────────────────────────────────────────────────────────┐
│  Rustのエラー処理                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  fn process_file(path: &str) -> Result<Data, Error> {           │
│      let content = read_file(path)?;                            │
│      let data = parse_json(&content)?;                          │
│      Ok(data)                                                   │
│  }                                                              │
│                                                                 │
│  特徴:                                                           │
│  ✅ 戻り値の型がResult → エラーの可能性が明確                    │
│  ✅ エラー処理を忘れると → コンパイルエラー                      │
│  ✅ エラーがどこで発生するか → ?が付いている箇所                 │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

🔄 **比較**:

| 言語 | エラー処理方法 | 問題点 |
|-----|--------------|-------|
| Python | `try`/`except` | どの関数が例外を投げるか分からない |
| JavaScript | `try`/`catch` | 同上 + 非同期エラーが複雑 |
| Java | チェック例外 | 冗長、例外が伝播 |
| Go | 多値返却 | `err`の無視が可能 |
| Rust | `Result<T, E>` | **エラーの可能性が型に現れる** |

---

## Result型 — 成功と失敗を表す列挙型

`Result<T, E>`は標準ライブラリで定義されている列挙型です。

```rust
enum Result<T, E> {
    Ok(T),   // 成功: 値Tを持つ
    Err(E),  // 失敗: エラーEを持つ
}
```

### Result型の使い方

```rust
use std::fs::File;

fn main() {
    // File::openはResult<File, std::io::Error>を返す
    let file_result: Result<File, std::io::Error> = File::open("hello.txt");
    
    match file_result {
        Ok(file) => println!("ファイルを開きました: {:?}", file),
        Err(error) => println!("エラー: {}", error),
    }
}
```

### 各部分を読み解く

```rust
let file_result: Result<File, std::io::Error> = File::open("hello.txt");
```

```
Result<File, std::io::Error>
   ↓      ↓           ↓
Result<成功時の型, 失敗時の型>

Ok(file)  → 成功時、File型の値が入っている
Err(error) → 失敗時、io::Error型の値が入っている
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

// 使用時
const result = openFile("hello.txt");
if (result.ok) {
    console.log(result.value);  // File
} else {
    console.log(result.error);  // Error
}
```

---

## `?`演算子 — エラーの伝播を1文字で

エラーを呼び出し元に返したい場合、`?`演算子を使います。

### `?`の動作

```rust
use std::fs::File;
use std::io::{self, Read};

fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;  // ① Errなら即return
    let mut username = String::new();
    file.read_to_string(&mut username)?;         // ② Errなら即return
    Ok(username)                                  // ③ 成功時
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│  ? 演算子の動作                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  let mut file = File::open("username.txt")?;                    │
│                                                   ↓              │
│  ┌─────────────────────────────────────────────────┐            │
│  │ File::open() の結果をチェック                   │            │
│  ├─────────────────────────────────────────────────┤            │
│  │ Ok(file)  → file を取り出して変数に代入         │            │
│  │ Err(e)    → 即座に Err(e) を return            │            │
│  └─────────────────────────────────────────────────┘            │
│                                                                 │
│  つまり ? は以下の省略形:                                        │
│  let mut file = match File::open("username.txt") {              │
│      Ok(f) => f,                                                │
│      Err(e) => return Err(e),                                   │
│  };                                                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### `?`なしで書くとどうなるか

```rust
// ?を使わない場合（冗長）
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

// ?を使う場合（簡潔）
fn read_username_from_file() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

### `?`のチェーン

メソッドチェーンでも使えます：

```rust
fn read_username_from_file() -> Result<String, io::Error> {
    let mut username = String::new();
    File::open("username.txt")?.read_to_string(&mut username)?;
    Ok(username)
}

// さらに短く（標準ライブラリに便利関数がある）
fn read_username_from_file() -> Result<String, io::Error> {
    std::fs::read_to_string("username.txt")
}
```

### `?`が使える条件

`?`は**Resultを返す関数内でのみ**使えます。

```rust
fn main() {
    let file = File::open("hello.txt")?;  // ❌ エラー！
    // main()はデフォルトでResultを返さない
}
```

**解決策**:

```rust
// main関数もResultを返せる
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("hello.txt")?;  // ✅ OK
    println!("ファイルを開きました: {:?}", file);
    Ok(())
}
```

---

## 複数のエラー型を扱う

実際のプログラムでは、異なる種類のエラーが発生します。

### 問題: エラー型が一致しない

```rust
use std::fs::File;
use std::io::Read;

fn read_number_from_file() -> Result<i32, ???> {  // どのエラー型？
    let mut file = File::open("number.txt")?;  // io::Error
    let mut content = String::new();
    file.read_to_string(&mut content)?;         // io::Error
    let number: i32 = content.trim().parse()?;  // ParseIntError ← 違う型！
    Ok(number)
}
```

### 解決策1: `Box<dyn Error>`（簡単）

```rust
use std::error::Error;
use std::fs::File;
use std::io::Read;

fn read_number_from_file() -> Result<i32, Box<dyn Error>> {
    let mut file = File::open("number.txt")?;
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    let number: i32 = content.trim().parse()?;
    Ok(number)
}
```

`Box<dyn Error>`は「Errorトレイトを実装するあらゆる型」を受け入れます。

### 解決策2: カスタムエラー型（より良い）

```rust
use std::fmt;

#[derive(Debug)]
enum AppError {
    Io(std::io::Error),
    Parse(std::num::ParseIntError),
}

impl fmt::Display for AppError {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        match self {
            AppError::Io(e) => write!(f, "IO error: {}", e),
            AppError::Parse(e) => write!(f, "Parse error: {}", e),
        }
    }
}

impl std::error::Error for AppError {}

// From トレイトで自動変換を実装
impl From<std::io::Error> for AppError {
    fn from(error: std::io::Error) -> Self {
        AppError::Io(error)
    }
}

impl From<std::num::ParseIntError> for AppError {
    fn from(error: std::num::ParseIntError) -> Self {
        AppError::Parse(error)
    }
}

fn read_number_from_file() -> Result<i32, AppError> {
    let mut file = File::open("number.txt")?;  // 自動でAppError::Ioに変換
    let mut content = String::new();
    file.read_to_string(&mut content)?;
    let number: i32 = content.trim().parse()?;  // 自動でAppError::Parseに変換
    Ok(number)
}
```

### 解決策3: `thiserror`クレート（実務で推奨）

手動での実装は面倒なので、`thiserror`クレートを使うのが一般的です。

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum AppError {
    #[error("ファイルを読み込めませんでした: {0}")]
    Io(#[from] std::io::Error),  // #[from]で自動変換
    
    #[error("数値をパースできませんでした: {0}")]
    Parse(#[from] std::num::ParseIntError),
    
    #[error("不正な入力: {message}")]
    InvalidInput { message: String },
}

fn read_number_from_file() -> Result<i32, AppError> {
    let content = std::fs::read_to_string("number.txt")?;  // ?で自動変換
    let number: i32 = content.trim().parse()?;
    
    if number < 0 {
        return Err(AppError::InvalidInput {
            message: "負の数は許可されていません".to_string(),
        });
    }
    
    Ok(number)
}
```

### 解決策4: `anyhow`クレート（アプリケーション向け）

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

fn main() -> Result<()> {
    let config = read_config()?;
    println!("Config: {:?}", config);
    Ok(())
}
```

**thiserror vs anyhow の使い分け**:

| 用途 | 推奨クレート | 理由 |
|-----|-------------|------|
| ライブラリ開発 | `thiserror` | 利用者が具体的なエラー型を知れる |
| アプリケーション開発 | `anyhow` | エラー処理が簡潔、コンテキスト追加が容易 |

---

## panic! — 回復不能なエラー

どうしても回復できないエラーは`panic!`で処理します。

```rust
fn main() {
    panic!("Something went terribly wrong!");
}
```

出力：
```
thread 'main' panicked at 'Something went terribly wrong!', src/main.rs:2:5
```

### いつpanicを使うか

```
┌─────────────────────────────────────────────────────────────────┐
│  panic vs Result の使い分け                                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Result を使うべき場合（回復可能なエラー）:                       │
│  - ファイルが見つからない → 別のパスを試すか、ユーザーに通知      │
│  - ネットワークエラー → リトライするか、エラーメッセージ表示      │
│  - 不正なユーザー入力 → バリデーションエラーを返す                │
│                                                                 │
│  panic を使うべき場合（プログラムのバグ）:                        │
│  - 配列の範囲外アクセス → ロジックエラー                         │
│  - 不変条件の違反 → プログラムの前提が崩れている                  │
│  - 「ここに来ることはありえない」状況                             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```rust
fn divide(a: i32, b: i32) -> i32 {
    if b == 0 {
        panic!("ゼロ除算は許可されていません");  // バグ
    }
    a / b
}

// より良い設計: Resultを返す
fn divide_safe(a: i32, b: i32) -> Result<i32, &'static str> {
    if b == 0 {
        Err("ゼロ除算")
    } else {
        Ok(a / b)
    }
}
```

---

## unwrapとexpect — 手軽だが危険

`Option`や`Result`から値を取り出す最も簡単な方法ですが、失敗時にpanicします。

```rust
fn main() {
    let x: Option<i32> = Some(5);
    let none: Option<i32> = None;
    
    // unwrap: Some/Okなら中身を返す、None/Errならpanic
    let value = x.unwrap();  // 5
    // let crash = none.unwrap();  // ⚠️ panic!
    
    // expect: unwrap + panicメッセージ付き
    let value = x.expect("値がNoneです");
    
    // Resultでも同様
    let result: Result<i32, &str> = Ok(10);
    let value = result.unwrap();  // 10
    let value = result.expect("エラーが発生しました");
}
```

### unwrapを使っても良い場面

```rust
// 1. テストコード
#[test]
fn test_something() {
    let result = some_function();
    assert_eq!(result.unwrap(), expected_value);  // テストなので失敗してOK
}

// 2. プロトタイプ・サンプルコード
fn main() {
    let file = File::open("example.txt").unwrap();  // 学習目的
}

// 3. 確実に成功する場合
fn main() {
    let number: i32 = "42".parse().unwrap();  // リテラルなので失敗しない
}
```

⚠️ **本番コードでは基本的にunwrapを避けましょう**。

### より安全な代替手段

```rust
fn main() {
    let x: Option<i32> = None;
    
    // unwrap_or: デフォルト値を指定
    let value = x.unwrap_or(0);  // 0
    
    // unwrap_or_default: 型のデフォルト値を使用
    let value = x.unwrap_or_default();  // 0 (i32のデフォルト)
    
    // unwrap_or_else: デフォルト値を遅延評価
    let value = x.unwrap_or_else(|| {
        println!("デフォルト値を計算中...");
        expensive_computation()
    });
    
    // if let / match: パターンマッチ
    if let Some(v) = x {
        println!("値: {}", v);
    } else {
        println!("値がありません");
    }
}
```

---

## Option と Result の変換

両者は頻繁に変換が必要になります。

```rust
fn main() {
    // Option → Result
    let x: Option<i32> = Some(5);
    let result: Result<i32, &str> = x.ok_or("値がありません");
    
    // None → Errに変換
    let none: Option<i32> = None;
    let result: Result<i32, &str> = none.ok_or("値がありません");  // Err("値がありません")
    
    // Result → Option
    let ok: Result<i32, &str> = Ok(10);
    let option: Option<i32> = ok.ok();  // Some(10)
    
    let err: Result<i32, &str> = Err("error");
    let option: Option<i32> = err.ok();  // None（エラーは捨てられる）
}
```

---

## map と and_then — 関数型スタイル

ResultやOptionを関数型スタイルで処理できます。

### map: 成功時の値を変換

```rust
fn main() {
    let result: Result<i32, &str> = Ok(5);
    
    // mapで成功時の値を変換
    let doubled: Result<i32, &str> = result.map(|x| x * 2);
    // Ok(10)
    
    let err: Result<i32, &str> = Err("error");
    let doubled: Result<i32, &str> = err.map(|x| x * 2);
    // Err("error") ← Errはそのまま通過
}
```

### and_then: 成功時に別のResultを返す

```rust
fn parse_then_double(s: &str) -> Result<i32, std::num::ParseIntError> {
    s.parse::<i32>()
        .and_then(|n| Ok(n * 2))  // パース成功したら2倍
}

// より実用的な例
fn get_user_and_posts(user_id: u32) -> Result<(User, Vec<Post>), AppError> {
    find_user(user_id)
        .and_then(|user| {
            let posts = find_posts_by_user(user.id)?;
            Ok((user, posts))
        })
}
```

🔄 **比較（JavaScript Promise）**:
```javascript
promise
    .then(x => x * 2)           // map相当
    .then(x => anotherPromise)  // and_then相当
    .catch(err => handleError); // Err処理
```

### map_err: エラーを変換

```rust
fn read_config() -> Result<Config, AppError> {
    std::fs::read_to_string("config.json")
        .map_err(|e| AppError::ConfigLoadError(e))?  // io::Error → AppError
        // ...
}
```

---

## 実践パターン

### パターン1: 早期リターン

```rust
fn process_user(user_id: Option<i32>) -> Result<User, AppError> {
    // Noneなら早期リターン
    let id = user_id.ok_or(AppError::InvalidInput {
        message: "user_id is required".into(),
    })?;
    
    // 以降はidが確実に存在する
    let user = find_user(id)?;
    validate_user(&user)?;
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
    if name.len() > 100 {
        errors.push("名前が長すぎます".to_string());
    }
    if age < 0 {
        errors.push("年齢は0以上である必要があります".to_string());
    }
    if age > 150 {
        errors.push("年齢が不正です".to_string());
    }
    
    if errors.is_empty() {
        Ok(())
    } else {
        Err(errors)
    }
}
```

### パターン3: デフォルト値でフォールバック

```rust
fn get_config_value(key: &str) -> String {
    std::env::var(key)
        .unwrap_or_else(|_| load_from_file(key)
            .unwrap_or_else(|_| default_value(key)))
}
```

### パターン4: エラーのロギングと継続

```rust
fn process_items(items: Vec<Item>) -> Vec<ProcessedItem> {
    items
        .into_iter()
        .filter_map(|item| {
            match process_item(item) {
                Ok(processed) => Some(processed),
                Err(e) => {
                    eprintln!("Warning: Failed to process item: {}", e);
                    None  // エラーは無視して続行
                }
            }
        })
        .collect()
}
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
| エラーメッセージは具体的に | 「エラー」ではなく「ファイル'config.json'が見つかりません」 |

---

## まとめ

| 概念 | Python/JS | Rust |
|-----|-----------|------|
| エラー表現 | 例外（Exception） | `Result<T, E>`型 |
| エラー伝播 | `raise`/`throw` | `?`演算子 |
| 値がない | `None`/`null` | `Option<T>` |
| 回復不能エラー | — | `panic!` |
| エラーキャッチ | `try`/`except`/`catch` | `match`/`if let` |

🎯 **この章のポイント**:
- **エラーは型で表現** — 関数のシグネチャを見ればエラーの可能性が分かる
- **`?`演算子** — エラー伝播を1文字で表現、最重要
- **Result<T, E>** — 成功(Ok)と失敗(Err)を持つ列挙型
- **網羅的な処理** — コンパイラがエラー処理忘れを検出
- **panic vs Result** — 回復可能かどうかで使い分け
- **thiserror/anyhow** — 実務では必須のクレート

---

## TypeScript/Pythonエンジニアのための移行コラム

### try-catchからの移行

```typescript
// TypeScript: try-catch
async function fetchUser(id: number): Promise<User> {
    try {
        const response = await fetch(`/users/${id}`);
        if (!response.ok) throw new Error("Not found");
        return response.json();
    } catch (e) {
        console.error(e);
        throw e;
    }
}
```

```rust
// Rust: Result
async fn fetch_user(id: u32) -> Result<User, AppError> {
    let response = reqwest::get(&format!("/users/{}", id))
        .await
        .map_err(|e| AppError::Network(e))?;
    
    if !response.status().is_success() {
        return Err(AppError::NotFound);
    }
    
    response.json().await.map_err(|e| AppError::Parse(e))
}
```

### よくあるエラー

**エラー1: `?`を返せない型の関数で使う**
```
error[E0277]: the `?` operator can only be used in a function that returns `Result`
```
解決: 関数の戻り値型を`Result<T, E>`に変更

**エラー2: エラー型が一致しない**
```
error[E0277]: `?` couldn't convert the error to `MyError`
```
解決: `From`トレイトを実装するか、`.map_err()`で変換

---

## 次のステップ

[Chapter 05: トレイト・ジェネリクス](05-traits-generics.md) では、Rustの抽象化メカニズムを学びます。`Error`トレイトの実装で使った`fmt::Display`や`From`トレイトの仕組みを深く理解できます。
