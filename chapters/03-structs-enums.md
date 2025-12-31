# Chapter 03: 構造体・列挙型

> **この章で学ぶこと**: struct、enum、impl、パターンマッチ — Rustでデータを構造化する方法

> 📖 **記号リファレンス**: この章では `impl`, `Self`, `::`, `&self` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## 🎯 なぜ構造体と列挙型が重要なのか

Python/TypeScriptから来た方にとって、Rustの構造体と列挙型は馴染みのある概念ですが、**重要な違い**があります。

```
┌─────────────────────────────────────────────────────────────────┐
│  各言語のデータ構造化                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python:                                                        │
│  - class で データ + メソッド を定義                             │
│  - 継承でコードを再利用                                          │
│  - @dataclass で単純なデータ構造                                 │
│                                                                 │
│  TypeScript:                                                    │
│  - interface で型を定義                                          │
│  - class でデータ + メソッド                                     │
│  - type で ユニオン型                                            │
│                                                                 │
│  Rust:                                                          │
│  - struct でデータを定義                                         │
│  - impl でメソッドを別に定義 ← 分離されている！                    │
│  - enum でデータを持つバリアント（TypeScriptのユニオン型相当）    │
│  - trait で共通インターフェース（継承の代わり）                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

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

### 各行を読み解く

```rust
struct User {
```
- `struct` = 構造体を定義するキーワード
- `User` = 構造体の名前（PascalCase）

```rust
    username: String,
    email: String,
```
- `フィールド名: 型` の形式
- カンマ区切りで複数フィールドを定義
- **すべてのフィールドに型が必要**（TypeScriptの`interface`と同じ）

```rust
let user1 = User {
    email: String::from("user@example.com"),
    ...
};
```
- 構造体のインスタンスを作成
- **すべてのフィールドに値を指定する必要がある**

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
    // mutなしの場合、すべてのフィールドが変更不可
    let user1 = User { ... };
    // user1.email = ...;  // ❌ エラー
    
    // mutありの場合、すべてのフィールドが変更可能
    let mut user2 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };
    
    user2.email = String::from("new@example.com");  // ✅ OK
}
```

**TypeScript/Pythonとの違い**:
- TypeScript/Python: プロパティごとに`readonly`/`Final`を指定
- Rust: インスタンス全体で変更可能性を決定

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

⚠️ **所有権に注意**: `..user1`を使うと、String型のフィールドは**ムーブ**します。

```rust
let user2 = User {
    email: String::from("another@example.com"),
    ..user1
};

// user1.usernameはムーブされた
println!("{}", user1.username);  // ❌ エラー！

// Copy型（u64, bool）は使える
println!("{}", user1.active);    // ✅ OK
println!("{}", user1.sign_in_count);  // ✅ OK
```

```
┌─────────────────────────────────────────────────────────────────┐
│  ..user1 のメモリ動作                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  user1:                       user2:                            │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │ username ───────│──×      │ username ───────│──┐            │
│  │ email    ───────│──┐      │ email    ───────│──│─→"another" │
│  │ sign_in_count: 1│  │      │ sign_in_count: 1│  │            │
│  │ active: true    │  │      │ active: true    │  │            │
│  └─────────────────┘  │      └─────────────────┘  │            │
│                       │                            ↓            │
│                       └→ "user@example"        "someuser"      │
│                          ↑                                      │
│                          ムーブでuser1.usernameは無効に         │
│                                                                 │
│  u64, boolはCopy型なのでコピーされる                             │
│  StringはCopy型でないのでムーブされる                            │
└─────────────────────────────────────────────────────────────────┘
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
    println!("R: {}", black.0);  // インデックスでアクセス
    println!("G: {}", black.1);
    println!("B: {}", black.2);
}
```

### ニュータイプパターン

タプル構造体の重要な使い方として**ニュータイプパターン**があります。

```rust
// 単位を型で区別する
struct Meters(f64);
struct Kilometers(f64);

fn travel_distance(meters: Meters) {
    println!("距離: {}m", meters.0);
}

fn main() {
    let distance_m = Meters(1000.0);
    let distance_km = Kilometers(1.0);
    
    travel_distance(distance_m);   // ✅ OK
    // travel_distance(distance_km);  // ❌ 型が違う！
}
```

これにより、「メートルとキロメートルを間違えて渡す」バグを**コンパイル時に防止**できます。

---

## メソッドの実装（impl）— 分離されたメソッド定義

Rustでは、データ（struct）とメソッド（impl）を**分離して定義**します。

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // メソッドを定義
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect = Rectangle { width: 30, height: 50 };
    println!("Area: {}", rect.area());
}
```

### `impl`ブロックを読み解く

```rust
impl Rectangle {
```
- `impl` = implementation（実装）の略
- `Rectangle` = メソッドを追加する対象の型
- **1つの型に複数の`impl`ブロックを書ける**

```rust
    fn area(&self) -> u32 {
```
- `fn area` = メソッド名
- `&self` = **このインスタンスへの不変参照**
- `-> u32` = 戻り値の型

### `self`の3つの形態 — 重要！

```rust
impl Rectangle {
    // 1. &self — 不変借用（読み取り専用）
    fn area(&self) -> u32 {
        self.width * self.height
    }
    
    // 2. &mut self — 可変借用（変更可能）
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }
    
    // 3. self — 所有権を取得（消費）
    fn destroy(self) -> (u32, u32) {
        (self.width, self.height)
    }
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│  self の3形態                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  &self（不変借用）                                               │
│  ├─ インスタンスを読み取るだけ                                    │
│  ├─ 呼び出し後もインスタンスを使える                              │
│  └─ 最も一般的                                                   │
│                                                                 │
│  &mut self（可変借用）                                           │
│  ├─ インスタンスを変更できる                                     │
│  ├─ 呼び出し後もインスタンスを使える                              │
│  └─ 例: vec.push(), string.push_str()                           │
│                                                                 │
│  self（所有権取得）                                              │
│  ├─ インスタンスの所有権をメソッドに移動                          │
│  ├─ 呼び出し後はインスタンスを使えない                            │
│  └─ 例: 変換メソッド、ビルダーパターン                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**使用例**:

```rust
fn main() {
    let mut rect = Rectangle { width: 30, height: 50 };
    
    // &self — 読み取り
    println!("Area: {}", rect.area());
    println!("Still usable: {:?}", rect);  // ✅ OK
    
    // &mut self — 変更
    rect.double();
    println!("Doubled: {:?}", rect);  // ✅ OK
    
    // self — 消費
    let dimensions = rect.destroy();
    // println!("{:?}", rect);  // ❌ rectはムーブされた
    println!("Dimensions: {:?}", dimensions);
}
```

🔄 **比較**:

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| 読み取りメソッド | `def method(self)` | `method()` | `fn method(&self)` |
| 変更メソッド | `def method(self)` | `method()` | `fn method(&mut self)` |
| 消費メソッド | — | — | `fn method(self)` |

Python/TypeScriptでは区別がありませんが、Rustは**コンパイラが使い方を強制**します。

### 関連関数（Associated Functions）— 静的メソッド

`self`を取らない関数も`impl`ブロックに書けます。これを**関連関数**と呼びます。

```rust
impl Rectangle {
    // 関連関数（selfなし）
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }
    
    fn square(size: u32) -> Self {
        Self { width: size, height: size }
    }
}

fn main() {
    // 関連関数は :: で呼び出す
    let rect = Rectangle::new(30, 50);
    let square = Rectangle::square(10);
}
```

### `Self`と`::`を読み解く

```rust
fn new(width: u32, height: u32) -> Self {
    Self { width, height }
}
```

- `Self` = 「この`impl`ブロックの対象の型」を指す
- ここでは`Self` = `Rectangle`
- `Rectangle::new()` の `::` は「名前空間アクセス」

```
┌─────────────────────────────────────────────────────────────────┐
│  . と :: の使い分け                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  rect.area()           ← ドット: インスタンスメソッド呼び出し    │
│                          （selfを取る）                          │
│                                                                 │
│  Rectangle::new(30, 50) ← ダブルコロン: 関連関数呼び出し         │
│                          （selfを取らない）                      │
│                                                                 │
│  同様に:                                                         │
│  String::from("hello")  ← 関連関数                              │
│  Vec::new()             ← 関連関数                              │
│  Option::Some(5)        ← 列挙型のバリアント                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 列挙型（Enum）— データを持てる強力な型

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
    
    match dir {
        Direction::Up => println!("上"),
        Direction::Down => println!("下"),
        Direction::Left => println!("左"),
        Direction::Right => println!("右"),
    }
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

### データを持つ列挙型 — Rust独特の強力な機能

```rust
enum Message {
    Quit,                       // データなし
    Move { x: i32, y: i32 },    // 名前付きフィールド（構造体のような形）
    Write(String),              // String型のデータを1つ持つ
    ChangeColor(i32, i32, i32), // 3つの値を持つ
}

fn main() {
    let msg1 = Message::Quit;
    let msg2 = Message::Move { x: 10, y: 20 };
    let msg3 = Message::Write(String::from("hello"));
    let msg4 = Message::ChangeColor(255, 0, 0);
}
```

### 各バリアントの形式

```
┌─────────────────────────────────────────────────────────────────┐
│  列挙型バリアントの形式                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ユニット型（データなし）                                     │
│     Quit                                                        │
│                                                                 │
│  2. 構造体型（名前付きフィールド）                               │
│     Move { x: i32, y: i32 }                                     │
│                                                                 │
│  3. タプル型（位置でアクセス）                                   │
│     Write(String)                                               │
│     ChangeColor(i32, i32, i32)                                  │
│                                                                 │
│  これらを1つのenum内に混在させられる！                           │
└─────────────────────────────────────────────────────────────────┘
```

🔄 **比較（TypeScript）**:
```typescript
// TypeScriptでは判別可能なユニオン型で表現
type Message =
    | { type: "Quit" }
    | { type: "Move"; x: number; y: number }
    | { type: "Write"; content: string }
    | { type: "ChangeColor"; r: number; g: number; b: number };

// Rustの方がシンプルに書ける
```

### 列挙型にもメソッドを実装できる

```rust
impl Message {
    fn call(&self) {
        match self {
            Message::Quit => println!("Quit"),
            Message::Move { x, y } => println!("Move to ({}, {})", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
        }
    }
}

fn main() {
    let msg = Message::Write(String::from("hello"));
    msg.call();  // Write: hello
}
```

---

## パターンマッチ（match）— 網羅性チェック付きswitch

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

### なぜ網羅性チェックが重要か

```rust
fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        // Dime と Quarter を忘れている
    }
}
// ❌ コンパイルエラー: patterns `Dime` and `Quarter` not covered
```

🔄 **比較（TypeScript）**:
```typescript
// TypeScriptは設定次第でチェックが緩い
function valueInCents(coin: Coin): number {
    switch (coin) {
        case Coin.Penny: return 1;
        case Coin.Nickel: return 5;
        // Dime, Quarterを忘れてもエラーにならない場合がある
    }
}
```

### パターンマッチでの値の取り出し

```rust
fn process_message(msg: Message) {
    match msg {
        Message::Quit => {
            println!("終了します");
        }
        Message::Move { x, y } => {
            // 名前付きフィールドを分解
            println!("({}, {})に移動", x, y);
        }
        Message::Write(text) => {
            // タプルの値を取り出し
            println!("メッセージ: {}", text);
        }
        Message::ChangeColor(r, g, b) => {
            // 複数の値を取り出し
            println!("色をRGB({}, {}, {})に変更", r, g, b);
        }
    }
}
```

### `_` プレースホルダ — その他すべて

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

⚠️ **注意**: `_`を使うと網羅性チェックが甘くなるので、列挙型では慎重に使いましょう。

### ガード条件

```rust
fn describe_number(n: i32) {
    match n {
        0 => println!("ゼロ"),
        n if n < 0 => println!("負の数: {}", n),
        n if n % 2 == 0 => println!("正の偶数: {}", n),
        _ => println!("正の奇数: {}", n),
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

**なぜOptionが必要か**:

```
┌─────────────────────────────────────────────────────────────────┐
│  null vs Option                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JavaScript/Python (null/None):                                 │
│  - どの変数でもnull/Noneになりうる                               │
│  - 「値がないかも」が型に現れない                                │
│  - 実行時にNullPointerException/TypeError                       │
│                                                                 │
│  Rust (Option<T>):                                              │
│  - Option<T>型の変数だけが「値がないかも」                       │
│  - 「値がないかも」が型に明示される                              │
│  - Noneの処理を忘れるとコンパイルエラー                          │
│                                                                 │
│  例:                                                             │
│  String         → 必ず文字列がある                               │
│  Option<String> → 文字列があるかもしれないし、ないかもしれない   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Optionの使用例

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

### 便利なOptionメソッド

```rust
fn main() {
    let x: Option<i32> = Some(5);
    let none: Option<i32> = None;
    
    // unwrap: Someなら中身を返す、Noneならpanic
    let value = x.unwrap();  // 5
    // let crash = none.unwrap();  // ⚠️ panic!
    
    // expect: unwrap + エラーメッセージ付き
    let value = x.expect("値がありません");
    
    // unwrap_or: Noneの場合のデフォルト値
    let value = none.unwrap_or(0);  // 0
    
    // unwrap_or_else: デフォルト値を遅延評価
    let value = none.unwrap_or_else(|| {
        println!("デフォルト値を計算中...");
        compute_default()
    });
    
    // map: Someの中身を変換
    let doubled: Option<i32> = x.map(|n| n * 2);  // Some(10)
    let doubled_none: Option<i32> = none.map(|n| n * 2);  // None
    
    // and_then: Someなら別のOptionを返す処理を実行
    let result: Option<i32> = x.and_then(|n| {
        if n > 0 { Some(n * 2) } else { None }
    });
    
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
        None => {},  // Noneは何もしない（冗長）
    }
    
    // if letを使う場合（簡潔）
    if let Some(v) = some_value {
        println!("Value: {}", v);
    }
    
    // elseも書ける
    if let Some(v) = some_value {
        println!("Value: {}", v);
    } else {
        println!("値がありません");
    }
}
```

### let-else — 早期リターンパターン

```rust
fn process_number(opt: Option<i32>) -> i32 {
    // Noneなら早期リターン
    let Some(n) = opt else {
        return 0;
    };
    
    // ここではnが使える
    n * 2
}
```

---

## 派生マクロ（derive）— 自動実装

よく使う機能を自動実装できます。

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();  // Clone
    
    println!("{:?}", p1);        // Debug: Point { x: 1, y: 2 }
    println!("{}", p1 == p2);    // PartialEq: true
}
```

### よく使うderiveマクロ

| マクロ | 機能 | 使用例 |
|-------|-----|--------|
| `Debug` | `{:?}`でデバッグ出力 | `println!("{:?}", value)` |
| `Clone` | `.clone()`でコピー | `let copy = value.clone()` |
| `Copy` | 暗黙的コピー（小さい値向け） | 代入時に自動コピー |
| `PartialEq` | `==`で比較 | `if a == b { }` |
| `Eq` | 完全な等価性（NaN等を含まない） | HashMapのキーに必要 |
| `Hash` | ハッシュ値を計算 | HashSetやHashMapのキーに必要 |
| `Default` | デフォルト値を生成 | `Default::default()` |
| `Serialize/Deserialize` | JSON等との変換（serde） | API開発で必須 |

### deriveの組み合わせ例

```rust
use serde::{Deserialize, Serialize};

// API用のデータ構造
#[derive(Debug, Clone, Serialize, Deserialize)]
struct User {
    id: u32,
    name: String,
    email: String,
}

// HashMapのキーとして使う場合
#[derive(Debug, Clone, PartialEq, Eq, Hash)]
struct UserId(u32);
```

---

## 実践パターン

### パターン1: ビルダーパターン

複雑な構造体を段階的に構築：

```rust
#[derive(Debug)]
struct Request {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
}

struct RequestBuilder {
    url: String,
    method: String,
    headers: Vec<(String, String)>,
    body: Option<String>,
}

impl RequestBuilder {
    fn new(url: &str) -> Self {
        Self {
            url: url.to_string(),
            method: "GET".to_string(),
            headers: vec![],
            body: None,
        }
    }
    
    fn method(mut self, method: &str) -> Self {
        self.method = method.to_string();
        self  // selfを返してチェーン可能に
    }
    
    fn header(mut self, key: &str, value: &str) -> Self {
        self.headers.push((key.to_string(), value.to_string()));
        self
    }
    
    fn body(mut self, body: &str) -> Self {
        self.body = Some(body.to_string());
        self
    }
    
    fn build(self) -> Request {
        Request {
            url: self.url,
            method: self.method,
            headers: self.headers,
            body: self.body,
        }
    }
}

fn main() {
    let request = RequestBuilder::new("https://api.example.com")
        .method("POST")
        .header("Content-Type", "application/json")
        .body(r#"{"name": "Alice"}"#)
        .build();
    
    println!("{:?}", request);
}
```

### パターン2: 状態を列挙型で表現

```rust
enum ConnectionState {
    Disconnected,
    Connecting { attempt: u32 },
    Connected { session_id: String },
    Error { message: String },
}

struct Connection {
    state: ConnectionState,
}

impl Connection {
    fn new() -> Self {
        Self { state: ConnectionState::Disconnected }
    }
    
    fn connect(&mut self) {
        match &self.state {
            ConnectionState::Disconnected => {
                self.state = ConnectionState::Connecting { attempt: 1 };
            }
            ConnectionState::Connecting { attempt } => {
                self.state = ConnectionState::Connecting { attempt: attempt + 1 };
            }
            ConnectionState::Connected { .. } => {
                println!("Already connected");
            }
            ConnectionState::Error { .. } => {
                self.state = ConnectionState::Connecting { attempt: 1 };
            }
        }
    }
}
```

---

## まとめ

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| データ構造 | `class`, `@dataclass` | `interface`, `class` | `struct` |
| メソッド | クラス内に定義 | クラス内に定義 | `impl`ブロック（分離） |
| 列挙型 | `Enum` | `enum`, ユニオン型 | `enum`（データ付き可） |
| null表現 | `None` | `null`/`undefined` | `Option<T>` |
| パターンマッチ | `match`文（3.10+） | `switch`（弱い） | `match`（網羅性チェック） |

🎯 **この章のポイント**:
- **struct + impl** = クラスの代わり（データとメソッドが分離）
- **&self / &mut self / self** = メソッドがインスタンスをどう扱うか
- **Self** = impl対象の型を指す
- **::** = 関連関数やバリアントのアクセス
- **enum** = データを持てる強力な列挙型
- **Option** = null安全（値がないかもを型で表現）
- **match** = 網羅性チェック付きパターンマッチ

---

## TypeScript/Pythonエンジニアのための移行コラム

### クラスがないことへの適応

Python/TypeScriptでは「クラス」が中心ですが、Rustでは：

```python
# Python: 継承でコード再利用
class Animal:
    def speak(self): pass

class Dog(Animal):
    def speak(self):
        return "Woof!"
```

```rust
// Rust: トレイトでインターフェース定義（Chapter 05で詳しく解説）
trait Animal {
    fn speak(&self) -> &str;
}

struct Dog;

impl Animal for Dog {
    fn speak(&self) -> &str {
        "Woof!"
    }
}
```

### Option型への慣れ

最初は面倒に感じますが、すぐに慣れます：

```typescript
// TypeScript: null チェック
function getUser(id: number): User | null {
    return users.find(u => u.id === id) ?? null;
}

const user = getUser(1);
if (user !== null) {
    console.log(user.name);
}
```

```rust
// Rust: Option
fn get_user(id: u32) -> Option<User> {
    users.iter().find(|u| u.id == id).cloned()
}

if let Some(user) = get_user(1) {
    println!("{}", user.name);
}
// または
let name = get_user(1).map(|u| u.name).unwrap_or_default();
```

---

## 次のステップ

[Chapter 04: エラーハンドリング](04-error-handling.md) では、Rustのエラー処理パターンを学びます。`Result`型と`?`演算子を使った、try-catchとは異なるアプローチを解説します。Option型で学んだパターンマッチの知識が活きます。
