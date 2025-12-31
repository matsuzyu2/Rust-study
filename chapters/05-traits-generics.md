# Chapter 05: トレイト・ジェネリクス

> **この章で学ぶこと**: トレイト（振る舞いの抽象化）、ジェネリクス（型の抽象化）、トレイト境界

---

## トレイトとは

**トレイト（Trait）**は「型が実装すべき振る舞い」を定義します。TypeScriptの`interface`やPythonの`Protocol`（またはABC）に似ています。

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

これは「`summarize`メソッドを持つ」という契約を定義しています。

### トレイトの実装

```rust
struct Article {
    title: String,
    author: String,
    content: String,
}

struct Tweet {
    username: String,
    content: String,
}

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.title, self.author)
    }
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.content)
    }
}

fn main() {
    let article = Article {
        title: String::from("Rust入門"),
        author: String::from("Alice"),
        content: String::from("..."),
    };
    
    println!("{}", article.summarize());  // "Rust入門, by Alice"
}
```

🔄 **比較（TypeScript）**:
```typescript
interface Summary {
    summarize(): string;
}

class Article implements Summary {
    constructor(
        public title: string,
        public author: string,
        public content: string
    ) {}
    
    summarize(): string {
        return `${this.title}, by ${this.author}`;
    }
}
```

🔄 **比較（Python）**:
```python
from typing import Protocol

class Summary(Protocol):
    def summarize(self) -> str: ...

class Article:
    def __init__(self, title: str, author: str, content: str):
        self.title = title
        self.author = author
        self.content = content
    
    def summarize(self) -> str:
        return f"{self.title}, by {self.author}"
```

---

## デフォルト実装

トレイトにはデフォルト実装を持たせられます。

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(続きを読む...)")
    }
}

struct Article {
    title: String,
}

// デフォルト実装をそのまま使う
impl Summary for Article {}

fn main() {
    let article = Article { title: String::from("Rust") };
    println!("{}", article.summarize());  // "(続きを読む...)"
}
```

デフォルト実装を上書きすることもできます：

```rust
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("Title: {}", self.title)  // 上書き
    }
}
```

---

## トレイト境界（Trait Bounds）

「このトレイトを実装している型」という制約を付けられます。

```rust
// Summaryトレイトを実装している型のみ受け入れる
fn notify(item: &impl Summary) {
    println!("速報！ {}", item.summarize());
}

// より明示的な書き方（トレイト境界構文）
fn notify<T: Summary>(item: &T) {
    println!("速報！ {}", item.summarize());
}
```

🔄 **比較（TypeScript）**:
```typescript
function notify(item: Summary): void {
    console.log(`速報！ ${item.summarize()}`);
}
```

### 複数のトレイト境界

```rust
use std::fmt::{Debug, Display};

// DisplayとDebugの両方を実装している型
fn print_info<T: Display + Debug>(item: &T) {
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
}

// where句を使った書き方（長い場合に見やすい）
fn print_info<T>(item: &T)
where
    T: Display + Debug,
{
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
}
```

---

## ジェネリクス

**ジェネリクス**は「型をパラメータ化」する機能です。

### ジェネリック関数

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}

fn main() {
    let numbers = vec![34, 50, 25, 100, 65];
    println!("最大: {}", largest(&numbers));
    
    let chars = vec!['y', 'm', 'a', 'q'];
    println!("最大: {}", largest(&chars));
}
```

🔄 **比較（TypeScript）**:
```typescript
function largest<T>(list: T[]): T {
    // TypeScriptは比較演算子の制約を型で表現しにくい
    return list.reduce((a, b) => a > b ? a : b);
}
```

### ジェネリック構造体

```rust
struct Point<T> {
    x: T,
    y: T,
}

impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}

// 特定の型にのみメソッドを実装
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let int_point = Point { x: 5, y: 10 };
    let float_point = Point { x: 1.0, y: 4.0 };
    
    println!("x = {}", float_point.x());
    println!("距離 = {}", float_point.distance_from_origin());
    
    // int_point.distance_from_origin();  // ❌ エラー！f64専用
}
```

### 複数の型パラメータ

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let point = Point { x: 5, y: 4.0 };  // Point<i32, f64>
}
```

---

## よく使う標準トレイト

Rustの標準ライブラリには多くの便利なトレイトがあります。

### Debug — デバッグ出力

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);   // Point { x: 1, y: 2 }
    println!("{:#?}", p);  // 整形出力
}
```

### Clone と Copy

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();  // 明示的なコピー
}
```

`Copy`は暗黙のコピーを可能にします（スタック上の単純な型のみ）：

```rust
#[derive(Copy, Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;  // コピー（ムーブではない）
    println!("{:?}", p1);  // ✅ まだ使える
}
```

### PartialEq と Eq — 等価比較

```rust
#[derive(PartialEq, Eq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = Point { x: 1, y: 2 };
    println!("{}", p1 == p2);  // true
}
```

### PartialOrd と Ord — 順序比較

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Score(i32);

fn main() {
    let scores = vec![Score(80), Score(95), Score(70)];
    let max = scores.iter().max();  // Some(Score(95))
}
```

### Default — デフォルト値

```rust
#[derive(Default)]
struct Config {
    debug: bool,
    port: u16,
}

impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            port: 8080,
        }
    }
}

fn main() {
    let config = Config::default();
    let custom = Config { debug: true, ..Default::default() };
}
```

### From と Into — 型変換

```rust
struct Millimeters(u32);
struct Meters(u32);

impl From<Meters> for Millimeters {
    fn from(m: Meters) -> Self {
        Millimeters(m.0 * 1000)
    }
}

fn main() {
    let m = Meters(5);
    let mm: Millimeters = m.into();  // FromがあればIntoは自動実装
    let mm2 = Millimeters::from(Meters(3));
}
```

---

## トレイトオブジェクト — 動的ディスパッチ

異なる型を同じ変数で扱いたい場合、**トレイトオブジェクト**を使います。

```rust
trait Draw {
    fn draw(&self);
}

struct Circle {
    radius: f64,
}

struct Rectangle {
    width: f64,
    height: f64,
}

impl Draw for Circle {
    fn draw(&self) {
        println!("Circle with radius {}", self.radius);
    }
}

impl Draw for Rectangle {
    fn draw(&self) {
        println!("Rectangle {}x{}", self.width, self.height);
    }
}

fn main() {
    // dyn Draw = 「Drawトレイトを実装した何か」
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle { radius: 5.0 }),
        Box::new(Rectangle { width: 10.0, height: 5.0 }),
    ];
    
    for shape in shapes {
        shape.draw();
    }
}
```

🔄 **比較（TypeScript）**:
```typescript
interface Draw {
    draw(): void;
}

const shapes: Draw[] = [
    new Circle(5.0),
    new Rectangle(10.0, 5.0),
];

shapes.forEach(shape => shape.draw());
```

### 静的ディスパッチ vs 動的ディスパッチ

| 種類 | 構文 | パフォーマンス | 柔軟性 |
|-----|------|--------------|-------|
| 静的（ジェネリクス） | `<T: Trait>` | 高速（インライン化可能） | コンパイル時に型が決まる |
| 動的（トレイトオブジェクト） | `dyn Trait` | やや遅い（vtable経由） | 実行時に型を決められる |

---

## 関連型（Associated Types）

トレイトに「関連する型」を定義できます。

```rust
trait Iterator {
    type Item;  // 関連型
    
    fn next(&mut self) -> Option<Self::Item>;
}

struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;  // 具体的な型を指定
    
    fn next(&mut self) -> Option<Self::Item> {
        self.count += 1;
        if self.count <= 5 {
            Some(self.count)
        } else {
            None
        }
    }
}
```

🔄 **比較（ジェネリクスとの違い）**:

```rust
// ジェネリクス: 同じ型に複数の実装が可能
trait Convert<T> {
    fn convert(&self) -> T;
}

impl Convert<String> for i32 { ... }
impl Convert<f64> for i32 { ... }

// 関連型: 1つの型に1つの実装のみ
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
// i32に対してIteratorは1つしか実装できない
```

---

## 実践パターン

### パターン1: ビルダーパターンとトレイト

```rust
trait Builder {
    type Output;
    fn build(self) -> Self::Output;
}

struct UserBuilder {
    name: Option<String>,
    email: Option<String>,
}

impl UserBuilder {
    fn new() -> Self {
        UserBuilder { name: None, email: None }
    }
    
    fn name(mut self, name: impl Into<String>) -> Self {
        self.name = Some(name.into());
        self
    }
    
    fn email(mut self, email: impl Into<String>) -> Self {
        self.email = Some(email.into());
        self
    }
}

impl Builder for UserBuilder {
    type Output = User;
    
    fn build(self) -> User {
        User {
            name: self.name.expect("name is required"),
            email: self.email.expect("email is required"),
        }
    }
}
```

### パターン2: 拡張トレイト

既存の型に機能を追加：

```rust
trait StringExt {
    fn is_blank(&self) -> bool;
}

impl StringExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
}

fn main() {
    let s = "   ";
    println!("{}", s.is_blank());  // true
}
```

---

## まとめ

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| インターフェース | `Protocol`/ABC | `interface` | `trait` |
| ジェネリクス | 型ヒント`T` | `<T>` | `<T>` |
| 型制約 | `TypeVar(bound=)` | `extends` | トレイト境界 |
| 動的型 | デフォルト | `any`/ユニオン | `dyn Trait` |
| 型変換 | `__init__`等 | 手動 | `From`/`Into` |

🎯 **ポイント**:
- **トレイト** = 振る舞いの抽象化（「何ができるか」を定義）
- **ジェネリクス** = 型の抽象化（「どんな型でも」を表現）
- **トレイト境界** = 「このトレイトを実装している型」という制約
- **derive** = よく使うトレイトを自動実装

---

## 次のステップ

[Chapter 06: コレクション](06-collections.md) では、`Vec`、`HashMap`などのコレクション型と、イテレータを使った関数型プログラミングパターンを学びます。
