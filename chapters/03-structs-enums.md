# Chapter 03: 構造体・列挙型

> **この章で学ぶこと**: struct、enum、impl、パターンマッチ — Rustでデータを構造化する方法

---

## 構造体（Struct）— クラスの代わり

Rustには「クラス」がありません。代わりに**構造体（struct）**を使ってデータをまとめます。

### 基本的な構造体

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };
    
    println!("Username: {}", user1.username);
}
```

🔄 **比較（TypeScript）**:
```typescript
interface User {
    username: string;
    email: string;
    signInCount: number;
    active: boolean;
}

const user1: User = {
    email: "user@example.com",
    username: "someuser",
    active: true,
    signInCount: 1,
};
```

🔄 **比較（Python）**:
```python
from dataclasses import dataclass

@dataclass
class User:
    username: str
    email: str
    sign_in_count: int
    active: bool

user1 = User(
    email="user@example.com",
    username="someuser",
    active=True,
    sign_in_count=1,
)
```

### ミュータブルな構造体

構造体全体がミュータブルかイミュータブルかを決めます（フィールド単位ではない）。

```rust
fn main() {
    let mut user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };
    
    user1.email = String::from("new@example.com");  // ✅ OK
}
```

### フィールド初期化省略記法

変数名とフィールド名が同じなら省略できます。

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,      // email: email の省略
        username,   // username: username の省略
        active: true,
        sign_in_count: 1,
    }
}
```

🔄 **比較**: JavaScriptのオブジェクト省略記法と同じです。
```javascript
const user = { email, username, active: true };
```

### 構造体の更新記法

既存のインスタンスから一部だけ変更したい場合：

```rust
fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };
    
    let user2 = User {
        email: String::from("another@example.com"),
        ..user1  // 残りのフィールドはuser1から
    };
}
```

🔄 **比較（TypeScript）**:
```typescript
const user2 = { ...user1, email: "another@example.com" };
```

⚠️ **注意**: `..user1`を使うと、String型のフィールドは**ムーブ**します。

```rust
let user2 = User {
    email: String::from("another@example.com"),
    ..user1
};
// user1.usernameはムーブされたので使えない
// user1.activeやuser1.sign_in_countはCopy型なので使える
```

---

## タプル構造体

フィールド名のない構造体も作れます。

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);
    
    // ColorとPointは別の型（値が同じでも混同できない）
    println!("R: {}", black.0);
}
```

💡 **Tips**: 「ニュータイプパターン」として、既存の型に別の意味を持たせるのに便利です。

---

## メソッドの実装（impl）

構造体にメソッドを追加するには`impl`ブロックを使います。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // メソッド（&selfを取る）
    fn area(&self) -> u32 {
        self.width * self.height
    }
    
    // 可変メソッド（&mut selfを取る）
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }
    
    // 関連関数（selfを取らない = 静的メソッド）
    fn square(size: u32) -> Rectangle {
        Rectangle {
            width: size,
            height: size,
        }
    }
}

fn main() {
    let mut rect = Rectangle { width: 30, height: 50 };
    println!("Area: {}", rect.area());
    
    rect.double();
    println!("New area: {}", rect.area());
    
    // 関連関数の呼び出し（::を使う）
    let square = Rectangle::square(10);
}
```

🔄 **比較**:

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| インスタンスメソッド | `def method(self)` | `method()` | `fn method(&self)` |
| 可変メソッド | `def method(self)` | `method()` | `fn method(&mut self)` |
| 静的メソッド | `@staticmethod` | `static method()` | `fn method()` (selfなし) |
| コンストラクタ | `__init__` | `constructor` | 関連関数（慣習的に`new`） |

### コンストラクタパターン

```rust
impl Rectangle {
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }
}

fn main() {
    let rect = Rectangle::new(30, 50);
}
```

💡 **Tips**: `Self`は実装対象の型（ここでは`Rectangle`）を指します。

---

## 列挙型（Enum）

Rustの列挙型は非常に強力です。単なる定数の集合ではなく、**各バリアントがデータを持てます**。

### 基本的な列挙型

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

fn main() {
    let dir = Direction::Up;
}
```

🔄 **比較（TypeScript）**:
```typescript
enum Direction {
    Up,
    Down,
    Left,
    Right,
}
```

### データを持つ列挙型

ここからがRust独特の強力な機能です。

```rust
enum Message {
    Quit,                       // データなし
    Move { x: i32, y: i32 },    // 名前付きフィールド
    Write(String),              // String型を持つ
    ChangeColor(i32, i32, i32), // 3つの値を持つ
}

fn main() {
    let msg1 = Message::Quit;
    let msg2 = Message::Move { x: 10, y: 20 };
    let msg3 = Message::Write(String::from("hello"));
    let msg4 = Message::ChangeColor(255, 0, 0);
}
```

🔄 **比較（TypeScript）**:
```typescript
// TypeScriptでは判別可能なユニオン型で表現
type Message =
    | { type: "Quit" }
    | { type: "Move"; x: number; y: number }
    | { type: "Write"; content: string }
    | { type: "ChangeColor"; r: number; g: number; b: number };
```

### 列挙型にもメソッドを実装できる

```rust
impl Message {
    fn call(&self) {
        // パターンマッチでバリアントごとに処理
        match self {
            Message::Quit => println!("Quit"),
            Message::Move { x, y } => println!("Move to ({}, {})", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
        }
    }
}
```

---

## パターンマッチ（match）

`match`は`switch`文の強力版です。**すべてのケースを網羅**しなければコンパイルエラーになります。

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

🔄 **比較（TypeScript）**:
```typescript
// TypeScriptは網羅性チェックが弱い
function valueInCents(coin: Coin): number {
    switch (coin) {
        case Coin.Penny: return 1;
        case Coin.Nickel: return 5;
        // Dime, Quarterを忘れてもエラーにならない（設定次第）
    }
}
```

### パターンマッチでの値の取り出し

```rust
enum Option<T> {
    Some(T),
    None,
}

fn plus_one(x: Option<i32>) -> Option<i32> {
    match x {
        Some(i) => Some(i + 1),
        None => None,
    }
}
```

### `_` プレースホルダ

すべてを列挙したくない場合：

```rust
fn main() {
    let value = 3;
    
    match value {
        1 => println!("one"),
        2 => println!("two"),
        _ => println!("other"),  // それ以外すべて
    }
}
```

---

## Option型 — nullの代わり

Rustには`null`がありません。代わりに`Option<T>`を使います。

```rust
enum Option<T> {
    Some(T),  // 値がある
    None,     // 値がない
}
```

標準ライブラリに定義されており、プレリュードに含まれるため`Option::`なしで使えます。

```rust
fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;
    
    // Optionから値を取り出すにはパターンマッチ
    match some_number {
        Some(n) => println!("Number: {}", n),
        None => println!("No number"),
    }
}
```

🔄 **比較**:

| 言語 | 値がない表現 | 安全性 |
|-----|------------|-------|
| Python | `None` | 実行時に`TypeError` |
| TypeScript | `null`, `undefined` | `strictNullChecks`で型エラー |
| Rust | `Option::None` | **コンパイル時に必ずハンドリング** |

### 便利なOptionメソッド

```rust
fn main() {
    let x: Option<i32> = Some(5);
    
    // unwrap: Someなら中身を返す、Noneならパニック
    let value = x.unwrap();
    
    // expect: unwrap + エラーメッセージ付き
    let value = x.expect("値がありません");
    
    // unwrap_or: Noneの場合のデフォルト値
    let value = x.unwrap_or(0);
    
    // map: Someの中身を変換
    let doubled: Option<i32> = x.map(|n| n * 2);
    
    // is_some / is_none: 判定
    if x.is_some() {
        println!("値があります");
    }
}
```

### if let — 簡潔なパターンマッチ

1つのパターンだけ処理したい場合：

```rust
fn main() {
    let some_value: Option<i32> = Some(3);
    
    // matchを使う場合
    match some_value {
        Some(v) => println!("Value: {}", v),
        None => {},
    }
    
    // if letを使う場合（簡潔）
    if let Some(v) = some_value {
        println!("Value: {}", v);
    }
}
```

🔄 **比較（Python 3.10+）**:
```python
match some_value:
    case Some(v):
        print(f"Value: {v}")
    case None:
        pass
```

---

## 派生マクロ（derive）

よく使う機能を自動実装できます。

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();
    
    println!("{:?}", p1);        // Debug: Point { x: 1, y: 2 }
    println!("{}", p1 == p2);    // PartialEq: true
}
```

よく使うderiveマクロ：

| マクロ | 機能 | Python/TS相当 |
|-------|-----|--------------|
| `Debug` | `{:?}`でデバッグ出力 | `__repr__`、`console.log`対応 |
| `Clone` | `.clone()`でコピー | `copy.deepcopy` |
| `PartialEq` | `==`で比較 | `__eq__` |
| `Default` | `Default::default()`で初期値 | デフォルトコンストラクタ |
| `Serialize/Deserialize` | JSON等との変換（serde） | `json.dumps/loads` |

---

## まとめ

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| データ構造 | `class`, `@dataclass` | `interface`, `class` | `struct` |
| メソッド | クラス内に定義 | クラス内に定義 | `impl`ブロック |
| 列挙型 | `Enum` | `enum`, ユニオン型 | `enum`（データ付き可） |
| null表現 | `None` | `null`/`undefined` | `Option<T>` |
| パターンマッチ | `match`文（3.10+） | `switch`（弱い） | `match`（網羅性チェック） |

🎯 **ポイント**:
- Rustにはクラスがない → `struct` + `impl`で同等機能を実現
- `enum`がデータを持てる → 状態や結果を型安全に表現
- `Option`でnull安全 → 「値がないかも」を型で表現
- `match`は網羅的 → 処理忘れをコンパイラが検出

---

## 次のステップ

[Chapter 04: エラーハンドリング](04-error-handling.md) では、Rustのエラー処理パターンを学びます。`Result`型と`?`演算子を使った、try-catchとは異なるアプローチを解説します。
