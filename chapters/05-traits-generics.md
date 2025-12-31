# Chapter 05: トレイト・ジェネリクス

> **この章で学ぶこと**: トレイト（振る舞いの抽象化）、ジェネリクス（型の抽象化）、トレイト境界、静的ディスパッチと動的ディスパッチ

> 📖 **記号リファレンス**: この章では `<T>`, `&self`, `dyn`, `impl` など多くの記号が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## この章を学ぶ前に：なぜトレイトとジェネリクスが必要なのか？

プログラミングでは「似たような処理を、異なる型に対して行いたい」場面が頻繁にあります。

例えば「何かを文字列として要約する」機能を考えてみましょう：
- 記事（Article）なら「タイトル, by 著者」
- ツイート（Tweet）なら「@ユーザー名: 内容」
- 商品（Product）なら「商品名 - ¥価格」

Pythonではどうするでしょうか？

```python
# Python: ダックタイピングで実現
def print_summary(item):
    print(item.summarize())  # summarize()メソッドがあれば何でもOK
```

Pythonは**ダックタイピング**（"アヒルのように歩き、アヒルのように鳴くなら、それはアヒルだ"）を採用しています。`summarize()`メソッドさえあれば、どんなオブジェクトでも渡せます。

**しかし、これには問題があります：**

```python
# 実行時まで問題がわからない
class Article:
    def summarize(self):
        return f"{self.title}, by {self.author}"

class BrokenClass:
    pass  # summarize()メソッドがない！

print_summary(Article())      # ✅ OK
print_summary(BrokenClass())  # ❌ 実行時にAttributeError
```

Pythonでは、間違ったオブジェクトを渡しても**実行するまでエラーがわかりません**。本番環境でクラッシュして初めて気づく、という事態が起こりえます。

### Rustのアプローチ：コンパイル時に保証する

Rustは**コンパイル時に「この型は必要なメソッドを持っているか？」をチェック**します。これを実現するのが**トレイト（Trait）**です。

```
┌─────────────────────────────────────────────────────────────────┐
│                    設計思想の違い                                │
├─────────────────────────────────────────────────────────────────┤
│  Python（ダックタイピング）                                      │
│    → 「実行時にメソッドがあればOK」                               │
│    → 柔軟だが、バグが本番環境まで潜む可能性                        │
│                                                                 │
│  Rust（トレイト境界）                                            │
│    → 「コンパイル時にメソッドの存在を保証」                        │
│    → 厳格だが、バグをリリース前に発見できる                        │
└─────────────────────────────────────────────────────────────────┘
```

この違いは特に大規模なチーム開発や、長期間メンテナンスするプロジェクトで重要になります。

---

## トレイトとは

**トレイト（Trait）**は「型が実装すべき振る舞い」を定義する契約書です。TypeScriptの`interface`やPythonの`Protocol`（またはABC）に似ていますが、**コンパイル時に強制される**点が大きく異なります。

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

### このコードを行ごとに読み解く

| 行 | コード | 意味 |
|----|--------|------|
| 1 | `trait Summary` | 「Summary」という名前の振る舞いの契約を定義 |
| 2 | `fn summarize(&self) -> String;` | この契約を実装する型は、`summarize`メソッドを持つ必要がある |

**`&self`の意味**:
- `self` = このメソッドを呼び出しているインスタンス自身
- `&` = 借用（所有権を奪わず参照だけ借りる）
- つまり「自分自身を読み取り専用で参照する」という意味

これはPythonの`self`やTypeScriptの`this`に相当しますが、`&`があるのはRustならではです。なぜ`&`が必要なのかは、所有権の観点から理解できます：

```
┌─────────────────────────────────────────────────────────────────┐
│  なぜ &self なのか？                                             │
├─────────────────────────────────────────────────────────────────┤
│  self    → 所有権を奪う。メソッド呼び出し後、元の変数は使えない    │
│  &self   → 不変借用。読み取りのみ。元の変数はその後も使える        │
│  &mut self → 可変借用。読み書き可能。元の変数はその後も使える      │
├─────────────────────────────────────────────────────────────────┤
│  summarize()は値を読むだけなので、&self（不変借用）で十分          │
└─────────────────────────────────────────────────────────────────┘
```

### トレイトの実装

トレイトを定義したら、具体的な型に対して「この型はこの契約を満たします」と宣言します。

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

// 「ArticleはSummaryトレイトを実装する」という宣言
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{}, by {}", self.title, self.author)
    }
}

// 「TweetもSummaryトレイトを実装する」
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

### コードの重要ポイント

**`impl Summary for Article`の読み方**:
- `impl` = 実装する（implement）
- `Summary` = このトレイトを
- `for Article` = Article型に対して

これにより、コンパイラは「Article型は`summarize()`メソッドを持っている」ことを知ります。

**Pythonとの決定的な違い**:

```python
# Python: 実行時チェック
class Article:
    def summarize(self):  # typo: summrize() と書いても気づかない
        return f"{self.title}"
```

```rust
// Rust: コンパイル時チェック
impl Summary for Article {
    fn summrize(&self) -> String {  // ❌ コンパイルエラー！
        // "method `summrize` is not a member of trait `Summary`"
        format!("{}", self.title)
    }
}
```

Rustでは、トレイトで定義されていないメソッド名を書くと**コンパイルエラー**になります。タイプミスも即座に発見できます。

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

🔄 **比較（Python - Protocol）**:
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

トレイトにはデフォルトの実装を持たせることができます。これは「ほとんどの型で同じ実装になる」場合に便利です。

```rust
trait Summary {
    fn summarize(&self) -> String {
        String::from("(続きを読む...)")  // デフォルト実装
    }
}

struct Article {
    title: String,
}

// デフォルト実装をそのまま使う（中身を書かなくてよい）
impl Summary for Article {}

fn main() {
    let article = Article { title: String::from("Rust") };
    println!("{}", article.summarize());  // "(続きを読む...)"
}
```

### なぜデフォルト実装が便利なのか？

例えば、10個の型がすべて同じ実装を持つ場合、デフォルト実装がなければ同じコードを10回書く必要があります。デフォルト実装があれば`impl Trait for Type {}`と書くだけで済みます。

もちろん、デフォルト実装を上書きすることもできます：

```rust
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("Title: {}", self.title)  // 上書き
    }
}
```

---

## トレイト境界（Trait Bounds）— 型に制約をつける

ここからがRustの真骨頂です。「このトレイトを実装している型のみ受け入れる」という制約を関数に付けられます。

### 問題：Pythonのダックタイピングの危険性

```python
def notify(item):
    print(f"速報！ {item.summarize()}")

notify(Article())        # ✅ OK
notify("文字列です")      # ❌ 実行時エラー: AttributeError
notify(42)               # ❌ 実行時エラー: AttributeError
```

Pythonでは、`summarize()`メソッドを持たないオブジェクトを渡しても、**実行するまでエラーがわかりません**。

### 解決：Rustのトレイト境界

```rust
// Summaryトレイトを実装している型のみ受け入れる
fn notify(item: &impl Summary) {
    println!("速報！ {}", item.summarize());
}
```

**`&impl Summary`の読み方**:
- `&` = 参照（借用）
- `impl Summary` = Summaryトレイトを実装している何らかの型

これにより：

```rust
notify(&article);  // ✅ ArticleはSummaryを実装している
notify(&tweet);    // ✅ TweetもSummaryを実装している
notify(&42);       // ❌ コンパイルエラー！i32はSummaryを実装していない
```

**コンパイルエラーの例**:
```
error[E0277]: the trait bound `i32: Summary` is not satisfied
 --> src/main.rs:5:12
  |
5 |     notify(&42);
  |            ^^^ the trait `Summary` is not implemented for `i32`
```

バグが**本番環境ではなく、コンパイル時に**発見されます。

### より明示的な書き方：ジェネリクス構文

`&impl Trait`は実は糖衣構文（シンタックスシュガー）で、より明示的に書くこともできます：

```rust
// 糖衣構文
fn notify(item: &impl Summary) { ... }

// 明示的なジェネリクス構文（完全に同じ意味）
fn notify<T: Summary>(item: &T) { ... }
```

**`<T: Summary>`の読み方**:
- `<T>` = 型パラメータTを導入
- `T: Summary` = TはSummaryトレイトを実装していなければならない

```
┌─────────────────────────────────────────────────────────────────┐
│  <T: Summary> の分解                                            │
├─────────────────────────────────────────────────────────────────┤
│    <T>          型パラメータの宣言。「Tという名前の型を使う」      │
│                                                                 │
│    T: Summary   トレイト境界。「TはSummaryを実装している」         │
│                                                                 │
│    item: &T     引数itemの型は「Tへの参照」                       │
└─────────────────────────────────────────────────────────────────┘
```

🔄 **比較（TypeScript）**:
```typescript
function notify(item: Summary): void {
    console.log(`速報！ ${item.summarize()}`);
}
// TypeScriptのinterfaceは構造的部分型なので、
// summarize()メソッドがあればどんな型でも渡せる
```

### 複数のトレイト境界

「DisplayとDebug、両方を実装している型」のように、複数の制約を付けることもできます。

```rust
use std::fmt::{Debug, Display};

// DisplayとDebugの両方を実装している型のみ受け入れる
fn print_info<T: Display + Debug>(item: &T) {
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
}
```

**`T: Display + Debug`の読み方**:
- `+` = AND条件
- 「TはDisplayを実装していて、かつ、Debugも実装している」

### where句 — 長いトレイト境界を見やすく

トレイト境界が長くなると、関数シグネチャが読みにくくなります：

```rust
// 読みにくい...
fn some_function<T: Display + Clone + Debug, U: Clone + Debug>(t: &T, u: &U) -> i32 {
    // ...
}
```

`where`句を使うと、トレイト境界を関数シグネチャの後に書けます：

```rust
// where句で見やすく
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone + Debug,
    U: Clone + Debug,
{
    // ...
}
```

**where句が必須になる場面**:

特に、関連型やより複雑な制約を書く場合、`where`句でないと表現できないケースがあります：

```rust
fn process<I>(iter: I)
where
    I: Iterator,
    I::Item: Display,  // 関連型への制約はwhereが必要
{
    for item in iter {
        println!("{}", item);
    }
}
```

---

## ジェネリクス — 型をパラメータ化する

**ジェネリクス**は「具体的な型を後から決められる」機能です。「どんな型でも動く関数やデータ構造」を作れます。

### なぜジェネリクスが必要か？

i32のリストから最大値を見つける関数を書いたとします：

```rust
fn largest_i32(list: &[i32]) -> &i32 {
    let mut largest = &list[0];
    for item in list {
        if item > largest {
            largest = item;
        }
    }
    largest
}
```

次に、f64のリストでも同じことをしたくなったら？char でも？String でも？

```rust
fn largest_f64(list: &[f64]) -> &f64 { ... }  // ほぼ同じコード
fn largest_char(list: &[char]) -> &char { ... }  // ほぼ同じコード
```

**同じロジックを何度も書くのは非効率で、バグの温床**です。

### ジェネリクスで解決

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
    println!("最大: {}", largest(&numbers));  // i32で動く
    
    let chars = vec!['y', 'm', 'a', 'q'];
    println!("最大: {}", largest(&chars));    // charでも動く
}
```

### コードを行ごとに読み解く

```rust
fn largest<T: PartialOrd>(list: &[T]) -> &T
```

| 部分 | 意味 |
|------|------|
| `<T: PartialOrd>` | 型パラメータTを導入。TはPartialOrd（比較可能）を実装している必要あり |
| `list: &[T]` | 引数はT型のスライスへの参照 |
| `-> &T` | 戻り値はT型への参照 |

**なぜ`PartialOrd`が必要か？**

関数内で`item > largest`と比較しています。すべての型が比較できるわけではないので（例：関数型は比較できない）、「比較可能な型」という制約を付ける必要があります。`PartialOrd`はまさに「比較演算子が使える」ことを保証するトレイトです。

```
┌─────────────────────────────────────────────────────────────────┐
│  なぜ PartialOrd なのか？                                        │
├─────────────────────────────────────────────────────────────────┤
│  Pythonでは a > b と書けば、Pythonが実行時に比較を試みる          │
│  比較できない型なら実行時エラー                                   │
│                                                                 │
│  Rustでは「比較可能な型のみ」という制約をコンパイル時にチェック     │
│  PartialOrd を実装していない型を渡すとコンパイルエラー             │
└─────────────────────────────────────────────────────────────────┘
```

🔄 **比較（TypeScript）**:
```typescript
function largest<T>(list: T[]): T {
    // TypeScriptは比較演算子の制約を型で表現しにくい
    return list.reduce((a, b) => a > b ? a : b);
}
```

TypeScriptでは`a > b`が常に許されてしまい、意味のない比較（オブジェクト同士など）もコンパイルが通ってしまいます。Rustはトレイト境界でこれを防ぎます。

### ジェネリック構造体

関数だけでなく、構造体にもジェネリクスを使えます。

```rust
struct Point<T> {
    x: T,
    y: T,
}
```

**`Point<T>`の意味**:
- Pointという構造体を定義
- `<T>` = 型パラメータTを持つ（具体的な型は使用時に決まる）
- xとyは両方ともT型

```rust
fn main() {
    let int_point = Point { x: 5, y: 10 };       // Point<i32>
    let float_point = Point { x: 1.0, y: 4.0 };  // Point<f64>
}
```

### ジェネリック構造体へのメソッド実装

```rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

**`impl<T> Point<T>`の読み方**:
- `impl<T>` = 型パラメータTを使う実装ブロック
- `Point<T>` = Point<T>という型に対して

これは「どんな型Tに対しても、Point<T>はこのメソッドを持つ」という意味です。

### 特定の型にのみメソッドを実装

ジェネリック構造体でも、特定の型に対してのみメソッドを追加できます：

```rust
// f64専用のメソッド
impl Point<f64> {
    fn distance_from_origin(&self) -> f64 {
        (self.x.powi(2) + self.y.powi(2)).sqrt()
    }
}

fn main() {
    let float_point = Point { x: 3.0, y: 4.0 };
    println!("距離 = {}", float_point.distance_from_origin());  // 5.0
    
    let int_point = Point { x: 3, y: 4 };
    // int_point.distance_from_origin();  // ❌ コンパイルエラー！f64専用
}
```

これは「f64の場合のみ、原点からの距離を計算できる」という制限を型システムで表現しています。

### 複数の型パラメータ

xとyで異なる型を持たせたい場合は、複数の型パラメータを使います：

```rust
struct Point<T, U> {
    x: T,
    y: U,
}

fn main() {
    let mixed = Point { x: 5, y: 4.0 };  // Point<i32, f64>
}
```

---

## よく使う標準トレイト

Rustの標準ライブラリには、頻繁に使うトレイトが用意されています。多くは`#[derive(...)]`で自動実装できます。

### `#[derive(...)]`とは何か？

`#[derive(...)]`は**コンパイル時にコードを自動生成するマクロ**です。手動で書くべきボイラープレートコードを、コンパイラが代わりに書いてくれます。

```rust
// これを書くと...
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

// コンパイラがこれを自動生成する（イメージ）
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

### Debug — デバッグ出力

`{:?}`でデバッグ出力できるようになります。

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);   // Point { x: 1, y: 2 }
    println!("{:#?}", p);  // 整形出力（Pretty print）
}
```

🔄 **比較**: Pythonの`__repr__`やTypeScriptでオブジェクトをconsole.logするのに相当します。

### Clone と Copy — 値のコピー

**Clone**: 明示的なコピー（深いコピー）

```rust
#[derive(Clone)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();  // 明示的にコピーを作成
    // p1もp2も両方使える
}
```

**Copy**: 暗黙のコピー（スタック上の単純な型のみ）

```rust
#[derive(Copy, Clone)]  // CopyにはCloneが必要
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1;  // ムーブではなくコピーされる
    println!("{:?}", p1);  // ✅ p1もまだ使える
}
```

```
┌─────────────────────────────────────────────────────────────────┐
│  Clone vs Copy                                                  │
├─────────────────────────────────────────────────────────────────┤
│  Clone  明示的（.clone()が必要）、ヒープデータもコピー可能       │
│  Copy   暗黙的（代入時に自動コピー）、スタックのみの型限定        │
├─────────────────────────────────────────────────────────────────┤
│  Stringはヒープを使うのでCopyを実装できない → Cloneのみ          │
│  i32, f64, bool等はスタックのみ → Copy実装可能                   │
└─────────────────────────────────────────────────────────────────┘
```

### PartialEq と Eq — 等価比較

`==`と`!=`で比較できるようになります。

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

**PartialEqとEqの違い**:
- `PartialEq`: 「等しいかどうかを比較できる」（`==`が使える）
- `Eq`: 「常に反射律が成り立つ」（`a == a`が常にtrue）

`f64`は`NaN != NaN`なので`PartialEq`のみ実装。`i32`は`Eq`も実装できます。

### PartialOrd と Ord — 順序比較

`<`, `>`, `<=`, `>=`で比較できるようになります。

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Score(i32);

fn main() {
    let mut scores = vec![Score(80), Score(95), Score(70)];
    scores.sort();  // Ordがあるとソート可能
    let max = scores.iter().max();  // Some(&Score(95))
}
```

### Default — デフォルト値

`Default::default()`でデフォルト値を生成できるようになります。

```rust
#[derive(Default)]
struct Config {
    debug: bool,    // false
    port: u16,      // 0
    name: String,   // ""
}

fn main() {
    let config = Config::default();
    
    // 一部だけ変更したい場合
    let custom = Config {
        debug: true,
        ..Default::default()  // 残りはデフォルト
    };
}
```

カスタムのデフォルト値を設定したい場合は、手動で実装します：

```rust
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            port: 8080,  // 0ではなく8080
            name: String::from("App"),
        }
    }
}
```

### From と Into — 型変換

型から型への変換を定義します。

```rust
struct Millimeters(u32);
struct Meters(u32);

// 「MetersからMillimetersへの変換」を定義
impl From<Meters> for Millimeters {
    fn from(m: Meters) -> Self {
        Millimeters(m.0 * 1000)
    }
}

fn main() {
    let m = Meters(5);
    
    // 2つの書き方（どちらも同じ結果）
    let mm1 = Millimeters::from(Meters(5));  // From
    let mm2: Millimeters = Meters(5).into(); // Into（FromがあればIntoは自動実装）
}
```

**なぜFromを実装するとIntoが自動で使えるのか？**

Rustの標準ライブラリで、`From<T>`を実装すると自動的に`Into<T>`も実装されるブランケット実装があるためです。

```rust
// 標準ライブラリ内の定義（イメージ）
impl<T, U> Into<U> for T
where
    U: From<T>,
{
    fn into(self) -> U {
        U::from(self)
    }
}
```

---

## トレイトオブジェクト — 動的ディスパッチ

ここまで学んだジェネリクス（`<T: Trait>`）は**静的ディスパッチ**です。コンパイル時に具体的な型が決まります。

しかし「異なる型を同じコレクションに入れたい」場合はどうでしょう？

```rust
// これはコンパイルエラー！
let shapes: Vec<???> = vec![
    Circle { radius: 5.0 },      // Circle型
    Rectangle { width: 10.0, height: 5.0 },  // Rectangle型 ← 型が違う！
];
```

`Vec<T>`のTには1つの型しか入れられません。ここで**トレイトオブジェクト**の出番です。

### トレイトオブジェクトの基本

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
    // dyn Draw = 「Drawトレイトを実装した何らかの型」
    let shapes: Vec<Box<dyn Draw>> = vec![
        Box::new(Circle { radius: 5.0 }),
        Box::new(Rectangle { width: 10.0, height: 5.0 }),
    ];
    
    for shape in shapes {
        shape.draw();  // 実行時に適切なdraw()が呼ばれる
    }
}
```

### `Box<dyn Draw>`を分解して理解する

この記法は初見だと難解なので、一つずつ分解しましょう。

```
┌─────────────────────────────────────────────────────────────────┐
│  Box<dyn Draw> の分解                                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  dyn Draw                                                       │
│    └─ dyn = dynamic（動的）                                     │
│    └─ Draw = トレイト名                                         │
│    └─ 意味: 「Drawトレイトを実装した、何らかの型」               │
│                                                                 │
│  Box<...>                                                       │
│    └─ ヒープ上にデータを置くスマートポインタ                     │
│    └─ なぜ必要？→ 「何らかの型」はサイズ不明なのでスタックに置けない│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

**なぜBoxが必要なのか？**

スタックに値を置くには、コンパイル時にサイズがわかっている必要があります。しかし`dyn Draw`は「Drawを実装した何か」なので、Circleかもしれないし、Rectangleかもしれません。サイズが異なるので、スタックには直接置けません。

`Box`を使うと、値をヒープに置き、スタックにはポインタ（固定サイズ）だけを置きます。

```
スタック                    ヒープ
┌────────────────┐         ┌────────────────┐
│ shapes (Vec)   │         │                │
│  ├─ ptr ──────────────→  │ Circle         │
│  ├─ ptr ──────────────→  │ { radius: 5.0 }│
│  └─ len: 2     │         ├────────────────┤
└────────────────┘         │ Rectangle      │
                           │ { w: 10, h: 5 }│
                           └────────────────┘
```

### vtable（仮想関数テーブル）— 動的ディスパッチの仕組み

`dyn Trait`を使うと、**実行時に**正しいメソッドが呼び出されます。これは**vtable（仮想関数テーブル）**という仕組みで実現されます。

```
┌─────────────────────────────────────────────────────────────────┐
│  Box<dyn Draw> のメモリレイアウト                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Box<dyn Draw>は実際には「ファットポインタ」（2つのポインタ）     │
│                                                                 │
│  ┌──────────────────┐                                           │
│  │ data_ptr ────────│──→ 実際のデータ（Circle等）               │
│  │ vtable_ptr ──────│──→ メソッドテーブル                        │
│  └──────────────────┘                                           │
│                                                                 │
│  vtable（Circle用）           vtable（Rectangle用）              │
│  ┌─────────────────────┐     ┌─────────────────────┐            │
│  │ draw → Circle::draw │     │ draw → Rectangle::draw│          │
│  │ drop → ...          │     │ drop → ...           │           │
│  └─────────────────────┘     └─────────────────────┘            │
│                                                                 │
│  shape.draw() が呼ばれると：                                     │
│  1. vtable_ptrからvtableを参照                                  │
│  2. vtable内のdrawのアドレスを取得                               │
│  3. そのアドレスにジャンプして実行                                │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

🔄 **比較（TypeScript）**:
```typescript
interface Draw {
    draw(): void;
}

// TypeScriptではそのまま配列に入れられる
const shapes: Draw[] = [
    new Circle(5.0),
    new Rectangle(10.0, 5.0),
];

shapes.forEach(shape => shape.draw());
```

TypeScript/JavaScriptではすべてのオブジェクトがヒープにあり、変数は常に参照なので、`Box`のような明示的なヒープ割り当ては不要です。Rustは「スタック vs ヒープ」を明示的に管理するため、この記法が必要になります。

---

## impl Trait vs dyn Trait — 使い分けフローチャート

「いつ`impl Trait`を使い、いつ`dyn Trait`を使うか？」これはRust初学者が最も混乱するポイントです。

### 静的ディスパッチ vs 動的ディスパッチ

| 種類 | 構文 | パフォーマンス | 柔軟性 | サイズ |
|-----|------|--------------|-------|-------|
| 静的（ジェネリクス） | `impl Trait` / `<T: Trait>` | 高速（インライン化可能） | コンパイル時に型が決まる | 型ごとにコード生成 |
| 動的（トレイトオブジェクト） | `dyn Trait` | やや遅い（vtable経由） | 実行時に型を決められる | コードは1つ |

### 使い分けフローチャート

```
                   ┌─────────────────────────────────────┐
                   │ 関数の引数/戻り値でトレイトを使いたい │
                   └────────────────┬────────────────────┘
                                    │
                   ┌────────────────▼────────────────┐
                   │ 異なる型を同じコレクションに入れる？ │
                   └────────────────┬────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │ YES                 │                     │ NO
              ▼                     │                     ▼
    ┌─────────────────┐             │         ┌─────────────────┐
    │ dyn Trait を使う │             │         │ impl Trait を使う│
    │ (動的ディスパッチ) │             │         │ (静的ディスパッチ) │
    └─────────────────┘             │         └─────────────────┘
              │                     │                     │
              ▼                     │                     ▼
    Box<dyn Trait> 等で              │         fn foo(x: impl Trait)
    ヒープ割り当て必要               │         fn foo<T: Trait>(x: T)
                                    │
                   ┌────────────────▼────────────────┐
                   │   戻り値の型を隠蔽したい？         │
                   │   （実装の詳細を公開したくない）    │
                   └────────────────┬────────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              │ YES                 │                     │ NO
              ▼                     ▼                     ▼
    ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
    │ 戻り値に impl Trait│   │ 条件分岐で異なる  │   │ 具体的な型を返す │
    │ fn foo() -> impl X│   │ 型を返す？       │   │ fn foo() -> Vec │
    └─────────────────┘   └────────┬────────┘   └─────────────────┘
                                   │
              ┌────────────────────┼────────────────────┐
              │ YES                │                    │ NO
              ▼                    ▼                    ▼
    ┌─────────────────┐   ┌─────────────────┐
    │ Box<dyn Trait>  │   │ impl Trait でOK │
    │ を返す          │   │                 │
    └─────────────────┘   └─────────────────┘
```

### 具体例で理解する

**例1: 単純な関数引数 → impl Trait**

```rust
// ✅ 良い例: 引数が1つの型に固定される場合
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

**例2: 異なる型を返す可能性 → Box&lt;dyn Trait&gt;**

```rust
// ✅ 良い例: 条件によって異なる型を返す場合
fn create_shape(is_circle: bool) -> Box<dyn Draw> {
    if is_circle {
        Box::new(Circle { radius: 5.0 })
    } else {
        Box::new(Rectangle { width: 10.0, height: 5.0 })
    }
}
```

**例3: 戻り値の型を隠蔽 → impl Trait**

```rust
// ✅ 良い例: イテレータの具体的な型を隠す
fn create_iter() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter().map(|x| x * 2)
    // 実際の型は std::iter::Map<std::vec::IntoIter<i32>, ...> 
    // という複雑な型だが、呼び出し側は知る必要がない
}
```

---

## 関連型（Associated Types）

トレイトには「このトレイトに関連する型」を定義できます。

### なぜ関連型が必要か？

`Iterator`トレイトを考えてみましょう。イテレータは「次の要素を返す」機能を持ちますが、**何の型の要素を返すか**はイテレータによって異なります。

```rust
trait Iterator {
    type Item;  // 関連型: 「このイテレータが返す要素の型」
    
    fn next(&mut self) -> Option<Self::Item>;
}
```

**`Self::Item`の読み方**:
- `Self` = このトレイトを実装している型自身
- `Item` = その型が持つ関連型

### 関連型の実装

```rust
struct Counter {
    count: u32,
}

impl Iterator for Counter {
    type Item = u32;  // 「Counterのイテレータはu32を返す」と宣言
    
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

### 関連型 vs 型パラメータ（ジェネリクス）

「なぜ関連型を使うのか？ジェネリクスで`Iterator<T>`としても同じでは？」

```rust
// ジェネリクスの場合: 同じ型に複数の実装が可能
trait Convert<T> {
    fn convert(&self) -> T;
}

impl Convert<String> for i32 {
    fn convert(&self) -> String { self.to_string() }
}

impl Convert<f64> for i32 {
    fn convert(&self) -> f64 { *self as f64 }
}

// i32はConvert<String>もConvert<f64>も実装できる
```

```rust
// 関連型の場合: 1つの型に1つの実装のみ
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

// CounterのIterator実装は1つだけ
// type Item = u32 と type Item = String の両方は持てない
```

**使い分け**:
- **関連型**: 「この型に対して実装は1つ」（例: Iterator）
- **ジェネリクス**: 「同じ型に対して複数の実装が欲しい」（例: From, Into）

---

## 実践パターン

### パターン1: ビルダーパターン

複雑なオブジェクトを段階的に構築するパターンです。

```rust
struct User {
    name: String,
    email: String,
    age: Option<u32>,
}

struct UserBuilder {
    name: Option<String>,
    email: Option<String>,
    age: Option<u32>,
}

impl UserBuilder {
    fn new() -> Self {
        UserBuilder {
            name: None,
            email: None,
            age: None,
        }
    }
    
    // メソッドチェーンを可能にするため self を返す
    fn name(mut self, name: impl Into<String>) -> Self {
        self.name = Some(name.into());
        self
    }
    
    fn email(mut self, email: impl Into<String>) -> Self {
        self.email = Some(email.into());
        self
    }
    
    fn age(mut self, age: u32) -> Self {
        self.age = Some(age);
        self
    }
    
    fn build(self) -> Result<User, String> {
        Ok(User {
            name: self.name.ok_or("name is required")?,
            email: self.email.ok_or("email is required")?,
            age: self.age,
        })
    }
}

fn main() {
    let user = UserBuilder::new()
        .name("Alice")
        .email("alice@example.com")
        .age(30)
        .build()
        .unwrap();
}
```

**`impl Into<String>`の意味**:
- 「`Into<String>`を実装している任意の型」を受け入れる
- `&str`も`String`も渡せる（どちらもInto<String>を実装）
- 呼び出し側は`.name("Alice")`でも`.name(String::from("Alice"))`でもOK

### パターン2: 拡張トレイト

既存の型に自分で定義したメソッドを追加できます。

```rust
trait StringExt {
    fn is_blank(&self) -> bool;
    fn truncate_with_ellipsis(&self, max_len: usize) -> String;
}

impl StringExt for str {
    fn is_blank(&self) -> bool {
        self.trim().is_empty()
    }
    
    fn truncate_with_ellipsis(&self, max_len: usize) -> String {
        if self.len() <= max_len {
            self.to_string()
        } else {
            format!("{}...", &self[..max_len])
        }
    }
}

fn main() {
    let s = "   ";
    println!("{}", s.is_blank());  // true
    
    let long = "This is a very long text";
    println!("{}", long.truncate_with_ellipsis(10));  // "This is a ..."
}
```

これは**オープン・クローズド原則**（拡張に開いており、修正に閉じている）を実現する強力なパターンです。標準ライブラリの型を変更せずに、機能を追加できます。

---

## まとめ

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| インターフェース | `Protocol`/ABC | `interface` | `trait` |
| ジェネリクス | 型ヒント`T` | `<T>` | `<T>` |
| 型制約 | `TypeVar(bound=)` | `extends` | トレイト境界 `T: Trait` |
| 動的ディスパッチ | デフォルト | ユニオン型等 | `dyn Trait` |
| 型変換 | `__init__`等 | 手動 | `From`/`Into` |
| チェック時点 | 実行時 | コンパイル時（部分的） | コンパイル時（厳格） |

🎯 **この章のポイント**:
- **トレイト** = 振る舞いの契約（「何ができるか」を定義）
- **ジェネリクス** = 型の抽象化（「どんな型でも」を表現）
- **トレイト境界** = コンパイル時の型チェック（実行時エラーを防ぐ）
- **静的ディスパッチ（impl Trait）** = 高速、コンパイル時に型決定
- **動的ディスパッチ（dyn Trait）** = 柔軟、実行時に型決定
- **derive** = よく使うトレイトの自動実装

---

## TypeScript/Pythonエンジニアのための移行コラム

### Pythonのダックタイピングとの根本的な違い

Pythonでは「メソッドがあれば呼べる」というダックタイピングがデフォルトです：

```python
# Python: 実行時にメソッドの存在をチェック
def process(item):
    return item.process()  # どんなオブジェクトでも渡せる

process(ValidObject())   # ✅ 動く
process("invalid")       # ❌ 実行時エラー（process()メソッドがない）
```

Rustでは「コンパイル時にメソッドの存在を保証」します：

```rust
// Rust: コンパイル時にトレイトの実装をチェック
fn process<T: Processable>(item: T) {
    item.process()
}

process(ValidObject);  // ✅ コンパイル通る
process("invalid");    // ❌ コンパイルエラー（&strはProcessableを実装していない）
```

**移行時のマインドシフト**:
1. 「とりあえず動かしてみる」ではなく「型で契約を明示する」
2. トレイト境界は「制限」ではなく「保証」
3. コンパイラのエラーは「敵」ではなく「バグを事前に防ぐ味方」

### TypeScriptのinterfaceとの違い

TypeScriptの`interface`は**構造的部分型**です：

```typescript
interface Summary {
    summarize(): string;
}

// Article は Summary を明示的に implements していないが...
class Article {
    summarize(): string { return "..."; }
}

function print(item: Summary) {
    console.log(item.summarize());
}

print(new Article());  // ✅ 構造が一致するのでOK
```

Rustのトレイトは**名義型**です：

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article;

impl Article {
    fn summarize(&self) -> String { "...".to_string() }
}

fn print(item: &impl Summary) {
    println!("{}", item.summarize());
}

// print(&Article);  // ❌ ArticleはSummaryを明示的に実装していない！
```

Rustでは`impl Summary for Article`と**明示的に宣言**しない限り、トレイト境界を満たしません。

### よくあるコンパイルエラーと対処法

**エラー1: `the trait bound 'X: Y' is not satisfied`**

```
error[E0277]: the trait bound `MyType: Debug` is not satisfied
```

原因: `MyType`が`Debug`トレイトを実装していない
対処: `#[derive(Debug)]`を追加するか、手動で実装

```rust
#[derive(Debug)]  // これを追加
struct MyType { ... }
```

**エラー2: `the size for values of type 'dyn Trait' cannot be known`**

```
error[E0277]: the size for values of type `dyn Draw` cannot be known at compilation time
```

原因: `dyn Trait`はサイズ不明なので直接使えない
対処: `Box<dyn Trait>`や`&dyn Trait`を使う

```rust
// ❌ fn foo() -> dyn Draw
// ✅ fn foo() -> Box<dyn Draw>
```

**エラー3: `impl Trait` not allowed outside of function and inherent method return types**

```
error[E0562]: `impl Trait` not allowed outside of function return types
```

原因: `impl Trait`は構造体のフィールドなど一部の場所で使えない
対処: 具体的な型か`Box<dyn Trait>`を使う

```rust
// ❌ struct Foo { handler: impl Handler }
// ✅ struct Foo { handler: Box<dyn Handler> }
// ✅ struct Foo<H: Handler> { handler: H }  // ジェネリクスで解決
```

---

## 次のステップ

[Chapter 06: コレクション](06-collections.md) では、`Vec`、`HashMap`などのコレクション型と、イテレータを使った関数型プログラミングパターンを学びます。イテレータは`Iterator`トレイトを実装しており、この章で学んだ知識が活きてきます。
