# Chapter 05: トレイト・ジェネリクス 〜対話編〜

> **この章で学ぶこと**: トレイト（振る舞いの抽象化）、ジェネリクス（型の抽象化）、トレイト境界、静的ディスパッチと動的ディスパッチ

> 📖 **記号リファレンス**: この章では `<T>`, `&self`, `dyn`, `impl` など多くの記号が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## レイ「トレイトって、インターフェースみたいなもの?」

ユイがホワイトボードに`trait Summary`と書き始めたとき、レイは既視感を覚えた。

**レイ**: 「ねえユイさん、これTypeScriptのinterfaceっぽくない?」

**ユイ**: 「お、いいところに気づいたね。確かに似てる。でも決定的な違いが1つある」

ユイはホワイトボードに大きく図を描いた:

```
┌─────────────────────────────────────────────────────────────────┐
│                    設計思想の違い                                │
├─────────────────────────────────────────────────────────────────┤
│  Python（ダックタイピング）                                      │
│    → 「実行時にメソッドがあればOK」                               │
│    → 柔軟だが、バグが本番環境まで潜む可能性                        │
│                                                                 │
│  TypeScript（interface）                                        │
│    → 「コンパイル時にチェック」                                   │
│    → でも構造的部分型なので、明示的な宣言不要                      │
│                                                                 │
│  Rust（トレイト境界）                                            │
│    → 「コンパイル時にメソッドの存在を保証」                        │
│    → 厳格で、明示的な実装宣言が必要                               │
│    → バグをリリース前に発見できる                                │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**: 「Pythonで痛い目にあったことない? 本番で突然`AttributeError`が出るやつ」

**レイ**: 「あー、あるある。`summarize()`メソッドだと思って呼んだら、実は`summrize()`ってタイポしてたとか...」

**ユイ**: 「そう! Rustではそういうバグは**絶対に本番まで届かない**。コンパイルの段階で弾かれるから」

---

## トレイトの基本 — 契約書として理解する

**ユイ**: 「トレイトを『契約書』だと思って。こんな風に書く:」

```rust
trait Summary {
    fn summarize(&self) -> String;
}
```

**レイ**: 「`&self`って何? Pythonの`self`とは違うの?」

**ユイ**: 「いい質問! Rustは所有権があるから、selfの扱いが3種類あるんだ:」

ユイがホワイトボードに書き加える:

```
┌─────────────────────────────────────────────────────────────────┐
│  なぜ &self なのか?                                             │
├─────────────────────────────────────────────────────────────────┤
│  self      → 所有権を奪う。メソッド呼び出し後、元の変数は使えない │
│  &self     → 不変借用。読み取りのみ。元の変数はその後も使える     │
│  &mut self → 可変借用。読み書き可能。元の変数はその後も使える     │
├─────────────────────────────────────────────────────────────────┤
│  summarize()は値を読むだけなので、&self（不変借用）で十分       │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: 「なるほど。Pythonは全部参照だから気にしないけど、Rustは明示的にどう扱うか宣言するんだ」

**ユイ**: 「そういうこと。じゃあ、この契約を実装してみよう:」

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
```

**レイ**: 「`impl Summary for Article`の読み方は...『ArticleにSummaryを実装する』?」

**ユイ**: 「完璧! この一文で、コンパイラは『Article型は必ず`summarize()`メソッドを持っている』ことを保証できる」

---

## レイ「Pythonとの決定的な違いを実感した...」

**レイ**: 「ちょっと試してみたいんだけど、わざとタイポしたらどうなる?」

```rust
impl Summary for Article {
    fn summrize(&self) -> String {  // typo: summrize
        format!("{}", self.title)
    }
}
```

**ユイ**: 「コンパイラが即座に怒ってくれる:」

```
error[E0407]: method `summrize` is not a member of trait `Summary`
 --> src/main.rs:15:5
  |
15|     fn summrize(&self) -> String {
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ not a member of trait `Summary`
```

**レイ**: 「これ、すごくない? Pythonだったら本番で初めて気づくやつじゃん」

**ユイ**: 「そう。特にチーム開発だと威力を発揮する。誰かがAPIを変更したら、その瞬間に影響を受けるすべてのコードがコンパイルエラーになる」

ユイはTypeScriptのコードも横に書いた:

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

**レイ**: 「TypeScriptでもチェックはされるよね。何が違うの?」

**ユイ**: 「TypeScriptは**構造的部分型**だから、`implements Summary`を書かなくても、`summarize()`メソッドがあれば通る」

```typescript
// TypeScript: implements を書かなくてもOK
class Article {
    summarize(): string { return "..."; }
}

function print(item: Summary) {
    console.log(item.summarize());
}

print(new Article());  // ✅ 構造が一致するのでOK
```

**ユイ**: 「でもRustは**名義型**。明示的に`impl Summary for Article`と宣言しないと、トレイト境界を満たさない」

**レイ**: 「明示的な契約が必要ってことか...」

---

## デフォルト実装 — DRYの原則を守る

**ユイ**: 「トレイトには『デフォルト実装』を持たせられる。これ便利なんだよ」

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

**レイ**: 「中身が空の`impl`...? これで動くの?」

**ユイ**: 「動く。トレイトにデフォルト実装があるから。もちろん、上書きもできる:」

```rust
impl Summary for Article {
    fn summarize(&self) -> String {
        format!("Title: {}", self.title)  // 上書き
    }
}
```

**レイ**: 「10個の型が同じ実装を使うなら、コードを10回書かなくて済むわけか」

**ユイ**: 「そう。DRY原則（Don't Repeat Yourself）を守れる。しかも型安全に」

---

## トレイト境界 — 型に制約をつける

ユイが新しいセクションのタイトルをホワイトボードに書いた: **「トレイト境界 — Rustの真骨頂」**

**ユイ**: 「ここからが本番。Pythonでこんな関数、書いたことあるでしょ?」

```python
def notify(item):
    print(f"速報！ {item.summarize()}")

notify(Article())        # ✅ OK
notify("文字列です")      # ❌ 実行時エラー: AttributeError
notify(42)               # ❌ 実行時エラー: AttributeError
```

**レイ**: 「あー、ダックタイピングのアレね。実行するまでわからないやつ」

**ユイ**: 「Rustではこう書く:」

```rust
// Summaryトレイトを実装している型のみ受け入れる
fn notify(item: &impl Summary) {
    println!("速報！ {}", item.summarize());
}
```

**レイ**: 「`&impl Summary`って何?」

**ユイ**: 「分解すると:」

```
&impl Summary
 │   └───────── Summaryトレイトを実装している何らかの型
 └─ 参照（借用）
```

**ユイ**: 「これで、コンパイル時にチェックが入る:」

```rust
notify(&article);  // ✅ ArticleはSummaryを実装している
notify(&tweet);    // ✅ TweetもSummaryを実装している
notify(&42);       // ❌ コンパイルエラー！i32はSummaryを実装していない
```

**レイ**: 「エラーメッセージは?」

**ユイ**: 「こんな感じ:」

```
error[E0277]: the trait bound `i32: Summary` is not satisfied
 --> src/main.rs:5:12
  |
5 |     notify(&42);
  |            ^^^ the trait `Summary` is not implemented for `i32`
```

**レイ**: 「親切すぎる...『i32はSummaryを実装してない』って明示的に教えてくれるんだ」

---

## impl Trait vs ジェネリクス構文

**ユイ**: 「実は`&impl Summary`って、糖衣構文なんだ。正式にはこう:」

```rust
// 糖衣構文
fn notify(item: &impl Summary) { ... }

// 明示的なジェネリクス構文（完全に同じ意味）
fn notify<T: Summary>(item: &T) { ... }
```

**レイ**: 「`<T: Summary>`...これTypeScriptの`<T extends Summary>`に似てる」

**ユイ**: 「そう! コンセプトは同じ。分解すると:」

ユイがホワイトボードに書く:

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

**レイ**: 「じゃあ、どっちを使えばいいの?」

**ユイ**: 「シンプルな場合は`impl Trait`の方が読みやすい。でも複数の引数で同じ型を強制したいときはジェネリクスじゃないと無理」

```rust
// これだと、item1とitem2は異なる型でもOK
fn notify(item1: &impl Summary, item2: &impl Summary) { ... }

// これだと、item1とitem2は必ず同じ型
fn notify<T: Summary>(item1: &T, item2: &T) { ... }
```

**レイ**: 「なるほど! 型の一致を強制したいならジェネリクスね」

---

## 複数のトレイト境界 — AND条件

**ユイ**: 「DisplayとDebug、両方実装してる型だけ受け入れたいときは:」

```rust
use std::fmt::{Debug, Display};

// DisplayとDebugの両方を実装している型のみ受け入れる
fn print_info<T: Display + Debug>(item: &T) {
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
}
```

**レイ**: 「`+`がAND条件なんだ」

**ユイ**: 「そう。でもこれが長くなると読みにくいから、`where`句を使う:」

```rust
// 読みにくい...
fn some_function<T: Display + Clone + Debug, U: Clone + Debug>(t: &T, u: &U) -> i32 {
    // ...
}

// where句で見やすく
fn some_function<T, U>(t: &T, u: &U) -> i32
where
    T: Display + Clone + Debug,
    U: Clone + Debug,
{
    // ...
}
```

**レイ**: 「確かに見やすい! 関数シグネチャと制約が分離されてる」

**ユイ**: 「特に関連型への制約とか、複雑な条件になるとwhereじゃないと書けないケースもある:」

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

**レイ**: 「ジェネリクスの話がちょくちょく出てくるけど、そもそもなんでジェネリクスが必要なの?」

**ユイ**: 「例えばこんな関数を書いたとしよう:」

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

**ユイ**: 「次に、f64でも同じことしたくなったら?」

**レイ**: 「あー...同じコードをコピペして、型だけ変える?」

```rust
fn largest_f64(list: &[f64]) -> &f64 { ... }  // ほぼ同じコード
fn largest_char(list: &[char]) -> &char { ... }  // ほぼ同じコード
```

**ユイ**: 「そう。これがDRY原則違反。しかもバグの温床」

**レイ**: 「じゃあジェネリクスで...」

**ユイ**: 「そう!」

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

**レイ**: 「1つの関数で全部対応できる...! でも`PartialOrd`って何?」

---

## なぜPartialOrdが必要なのか

ユイがホワイトボードに大きく書いた:

```
┌─────────────────────────────────────────────────────────────────┐
│  なぜ PartialOrd なのか?                                        │
├─────────────────────────────────────────────────────────────────┤
│  Pythonでは a > b と書けば、Pythonが実行時に比較を試みる          │
│  比較できない型なら実行時エラー                                   │
│                                                                 │
│  Rustでは「比較可能な型のみ」という制約をコンパイル時にチェック     │
│  PartialOrd を実装していない型を渡すとコンパイルエラー             │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**: 「関数内で`item > largest`って比較してるでしょ? すべての型が比較できるわけじゃない」

**レイ**: 「例えば?」

**ユイ**: 「関数型は比較できない。だから、比較可能な型だけを受け入れる制約が必要」

```rust
fn largest<T>(list: &[T]) -> &T {  // ❌ T: PartialOrd がない!
    let mut largest = &list[0];
    for item in list {
        if item > largest {  // エラー: Tが比較可能か不明
            largest = item;
        }
    }
    largest
}
```

**ユイ**: 「コンパイラは教えてくれる:」

```
error[E0369]: binary operation `>` cannot be applied to type `&T`
  |
6 |         if item > largest {
  |            ---- ^ ------- &T
  |            |
  |            &T
  |
help: consider restricting type parameter `T`
  |
1 | fn largest<T: std::cmp::PartialOrd>(list: &[T]) -> &T {
  |             +++++++++++++++++++++++
```

**レイ**: 「親切! 『PartialOrd制約を付けたら?』って提案してくれてる」

**ユイ**: 「TypeScriptだとこうはいかない:」

```typescript
function largest<T>(list: T[]): T {
    // TypeScriptは比較演算子の制約を型で表現しにくい
    return list.reduce((a, b) => a > b ? a : b);
}

// オブジェクト同士の比較も通ってしまう（意味ないのに）
largest([{name: "Alice"}, {name: "Bob"}]);  // ❌ 実行時に変な結果
```

**レイ**: 「Rustのトレイト境界、かなり強力だね...」

---

## ジェネリック構造体 — データ構造も抽象化

**ユイ**: 「関数だけじゃなくて、構造体にもジェネリクス使える:」

```rust
struct Point<T> {
    x: T,
    y: T,
}

fn main() {
    let int_point = Point { x: 5, y: 10 };       // Point<i32>
    let float_point = Point { x: 1.0, y: 4.0 };  // Point<f64>
}
```

**レイ**: 「`Point<T>`...Tは使用時に決まるんだ」

**ユイ**: 「そう。Rustの型推論が強力だから、明示的に書かなくても推論してくれる」

```rust
// 明示的に書くなら
let int_point: Point<i32> = Point { x: 5, y: 10 };

// でも型推論で十分
let int_point = Point { x: 5, y: 10 };  // xとyがi32なので、Point<i32>と推論
```

**レイ**: 「これxとyで型を変えたかったら?」

**ユイ**: 「複数の型パラメータを使う:」

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

## ジェネリック構造体へのメソッド実装

**ユイ**: 「メソッドを実装するときもジェネリクスを使う:」

```rust
impl<T> Point<T> {
    fn x(&self) -> &T {
        &self.x
    }
}
```

**レイ**: 「`impl<T> Point<T>`...読み方は?」

**ユイ**: 「『型パラメータTを使って、Point<T>という型にメソッドを実装する』って意味」

ユイが図を描く:

```
impl<T> Point<T>
 │   │   └─────── Point<T>という型に対して
 │   └─ 型パラメータTを導入
 └─ 実装ブロック
```

**ユイ**: 「これは『どんな型Tに対しても、Point<T>はこのメソッドを持つ』って宣言」

**レイ**: 「じゃあ、f64専用のメソッドとかも作れる?」

**ユイ**: 「もちろん!」

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

**レイ**: 「型システムで『f64のときだけこのメソッドが使える』って表現できるんだ...すごい」

---

## よく使う標準トレイト — deriveの魔法

**ユイ**: 「Rustの標準ライブラリには、頻繁に使うトレイトが用意されてる。多くは`#[derive(...)]`で自動実装できる」

**レイ**: 「`#[derive(...)]`って何? 見たことあるけど」

**ユイ**: 「コンパイル時にコードを自動生成するマクロ。こう書くと:」

```rust
#[derive(Debug)]
struct Point {
    x: i32,
    y: i32,
}
```

**ユイ**: 「コンパイラがこういうコードを自動生成する（イメージ）:」

```rust
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

**レイ**: 「ボイラープレートを書かなくて済むわけか!」

---

## Debug — デバッグ出力

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

**レイ**: 「Pythonの`__repr__`みたいなもんだね」

**ユイ**: 「そう。JavaScriptでオブジェクトをconsole.logするのにも似てる」

---

## Clone vs Copy — 値のコピーの2つの流儀

**ユイ**: 「これ、初学者が混乱しがちなポイント」

ユイがホワイトボードに書く:

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

**Clone の例**:

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

**Copy の例**:

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

**レイ**: 「Copyがあると、所有権のムーブが起きないんだ」

**ユイ**: 「そう。i32とかプリミティブ型がCopyを実装してるから、普通に代入できるんだよね」

---

## PartialEq, Eq, PartialOrd, Ord — 比較のトレイト

**ユイ**: 「等価比較と順序比較のトレイト:」

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

**レイ**: 「PartialEqとEqの違いは?」

**ユイ**: 「PartialEqは『等しいかどうかを比較できる』だけ。Eqは『常に反射律が成り立つ』」

```
PartialEq: == が使える
Eq:        a == a が常にtrue（反射律）

f64はNaN != NaNなのでPartialEqのみ実装
i32はEqも実装できる
```

**順序比較**:

```rust
#[derive(PartialEq, Eq, PartialOrd, Ord)]
struct Score(i32);

fn main() {
    let mut scores = vec![Score(80), Score(95), Score(70)];
    scores.sort();  // Ordがあるとソート可能
    let max = scores.iter().max();  // Some(&Score(95))
}
```

**レイ**: 「Ordがあると、ソートとかmax/minが使えるのか」

---

## Default — デフォルト値を賢く扱う

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

**レイ**: 「`..Default::default()`って何?」

**ユイ**: 「構造体更新構文。『残りのフィールドはこれから取ってきて』って意味」

**カスタムデフォルト値**:

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

**レイ**: 「手動で実装すれば、デフォルト値をカスタマイズできるんだ」

---

## From と Into — 型変換の標準パターン

**ユイ**: 「型変換を定義するトレイト:」

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

**レイ**: 「Fromを実装すると、Intoが勝手に使えるの?」

**ユイ**: 「そう。これがブランケット実装。標準ライブラリにこんな定義がある:」

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

**レイ**: 「だから、Fromだけ実装すればIntoもついてくる!」

---

## トレイトオブジェクト — 動的ディスパッチの世界

**ユイ**: 「ここまでのジェネリクスは『静的ディスパッチ』。コンパイル時に型が決まる」

**レイ**: 「じゃあ、異なる型を同じコレクションに入れたいときは?」

**ユイ**: 「それができないのがジェネリクスの限界:」

```rust
// これはコンパイルエラー！
let shapes: Vec<???> = vec![
    Circle { radius: 5.0 },      // Circle型
    Rectangle { width: 10.0, height: 5.0 },  // Rectangle型 ← 型が違う！
];
```

**ユイ**: 「`Vec<T>`のTには1つの型しか入れられない。ここで**トレイトオブジェクト**の出番」

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

---

## Box<dyn Draw> を徹底解剖

**レイ**: 「`Box<dyn Draw>`...初見殺しすぎる。分解して教えて」

**ユイ**: 「OK、ホワイトボードに書く:」

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

**レイ**: 「なんでBoxが必要なの?」

**ユイ**: 「スタックに値を置くには、コンパイル時にサイズがわかってないとダメ。でも`dyn Draw`は『Drawを実装した何か』だから、Circleかもしれないし、Rectangleかもしれない」

**レイ**: 「サイズが違うから、スタックには置けない...」

**ユイ**: 「そう。Boxを使うと、値はヒープに置いて、スタックにはポインタ（固定サイズ）だけを置く」

ユイが図を描く:

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

---

## vtable — 動的ディスパッチの仕組み

**レイ**: 「実行時に適切なdraw()が呼ばれるって、どういう仕組み?」

**ユイ**: 「vtable（仮想関数テーブル）を使う。`Box<dyn Draw>`は実際には『ファットポインタ』— 2つのポインタを持ってる」

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

**レイ**: 「C++のvirtual関数テーブルと同じ仕組みか」

**ユイ**: 「そう。TypeScriptだとこんな感じ:」

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

**ユイ**: 「TypeScript/JavaScriptはすべてのオブジェクトがヒープにあって、変数は常に参照。だから`Box`みたいな明示的なヒープ割り当ては不要」

**レイ**: 「Rustは『スタック vs ヒープ』を明示的に管理するから、この記法が必要なんだね」

---

## impl Trait vs dyn Trait — 使い分けフローチャート

**レイ**: 「で、結局いつ`impl Trait`を使って、いつ`dyn Trait`を使うの?」

**ユイ**: 「これ、初学者が最も混乱するポイント。表にまとめるとこう:」

| 種類 | 構文 | パフォーマンス | 柔軟性 | サイズ |
|-----|------|--------------|-------|-------|
| 静的（ジェネリクス） | `impl Trait` / `<T: Trait>` | 高速（インライン化可能） | コンパイル時に型が決まる | 型ごとにコード生成 |
| 動的（トレイトオブジェクト） | `dyn Trait` | やや遅い（vtable経由） | 実行時に型を決められる | コードは1つ |

**ユイ**: 「フローチャートで考えると:」

ユイがホワイトボードに大きく描く:

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

---

## 具体例で理解する使い分け

**例1: 単純な関数引数 → impl Trait**

```rust
// ✅ 良い例: 引数が1つの型に固定される場合
fn print_summary(item: &impl Summary) {
    println!("{}", item.summarize());
}
```

**例2: 異なる型を返す可能性 → Box<dyn Trait>**

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

**レイ**: 「なるほど。ほとんどの場合は`impl Trait`で、異なる型を混在させたいときだけ`dyn Trait`なんだ」

---

## 関連型（Associated Types） — Iteratorの秘密

**ユイ**: 「トレイトには『関連型』を定義できる。これ、Iteratorで使われてる」

```rust
trait Iterator {
    type Item;  // 関連型: 「このイテレータが返す要素の型」

    fn next(&mut self) -> Option<Self::Item>;
}
```

**レイ**: 「`Self::Item`って何?」

**ユイ**: 「分解すると:」

```
Self::Item
 │    └─── 関連型Item
 └─ このトレイトを実装している型自身
```

**実装例**:

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

**レイ**: 「なんでジェネリクスで`Iterator<T>`じゃダメなの?」

---

## 関連型 vs ジェネリクス — 使い分けの本質

**ユイ**: 「いい質問! 違いはこう:」

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

**ユイ**: 「使い分けは:」

```
┌─────────────────────────────────────────────────────────────────┐
│  関連型 vs ジェネリクス                                          │
├─────────────────────────────────────────────────────────────────┤
│  関連型:      「この型に対して実装は1つ」（例: Iterator）          │
│  ジェネリクス: 「同じ型に対して複数の実装が欲しい」（例: From, Into）│
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: 「Iteratorは『1つの型は1種類の要素を返す』から、関連型なんだ」

**ユイ**: 「その通り!」

---

## 実践パターン1: ビルダーパターン

**ユイ**: 「実際の開発でよく使うパターンを見せる。まずはビルダーパターン」

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

**レイ**: 「`impl Into<String>`...これ賢い! `&str`でも`String`でも受け入れられる」

**ユイ**: 「そう。呼び出し側は`.name("Alice")`でも`.name(String::from("Alice"))`でもOK」

---

## 実践パターン2: 拡張トレイト

**ユイ**: 「既存の型に自分で定義したメソッドを追加できる。これ強力だよ」

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

**レイ**: 「標準ライブラリのstr型を変更せずに、機能を追加してる...!」

**ユイ**: 「これが**オープン・クローズド原則**（拡張に開いており、修正に閉じている）の実現。Rustの強力な機能の1つ」

---

## まとめ — 3言語比較表

**ユイ**: 「最後に、3言語の対応表を見せる:」

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| インターフェース | `Protocol`/ABC | `interface` | `trait` |
| ジェネリクス | 型ヒント`T` | `<T>` | `<T>` |
| 型制約 | `TypeVar(bound=)` | `extends` | トレイト境界 `T: Trait` |
| 動的ディスパッチ | デフォルト | ユニオン型等 | `dyn Trait` |
| 型変換 | `__init__`等 | 手動 | `From`/`Into` |
| チェック時点 | 実行時 | コンパイル時（部分的） | コンパイル時（厳格） |

**レイ**: 「Rustはコンパイル時に厳格にチェックする、と」

**ユイ**: 「そう。この章のポイントをまとめると:」

```
🎯 この章のポイント:
- トレイト = 振る舞いの契約（「何ができるか」を定義）
- ジェネリクス = 型の抽象化（「どんな型でも」を表現）
- トレイト境界 = コンパイル時の型チェック（実行時エラーを防ぐ）
- 静的ディスパッチ（impl Trait） = 高速、コンパイル時に型決定
- 動的ディスパッチ（dyn Trait） = 柔軟、実行時に型決定
- derive = よく使うトレイトの自動実装
```

---

## TypeScript/Pythonエンジニアのための移行ガイド

**レイ**: 「ユイさん、最後に移行時のマインドシフトを教えて」

**ユイ**: 「OK。一番重要なのはこれ:」

### Pythonのダックタイピングとの根本的な違い

```python
# Python: 実行時にメソッドの存在をチェック
def process(item):
    return item.process()  # どんなオブジェクトでも渡せる

process(ValidObject())   # ✅ 動く
process("invalid")       # ❌ 実行時エラー（process()メソッドがない）
```

```rust
// Rust: コンパイル時にトレイトの実装をチェック
fn process<T: Processable>(item: T) {
    item.process()
}

process(ValidObject);  // ✅ コンパイル通る
process("invalid");    // ❌ コンパイルエラー（&strはProcessableを実装していない）
```

**ユイ**: 「マインドシフトはこう:」

```
1. 「とりあえず動かしてみる」ではなく「型で契約を明示する」
2. トレイト境界は「制限」ではなく「保証」
3. コンパイラのエラーは「敵」ではなく「バグを事前に防ぐ味方」
```

---

## よくあるコンパイルエラーと対処法

**エラー1: `the trait bound 'X: Y' is not satisfied`**

```
error[E0277]: the trait bound `MyType: Debug` is not satisfied
```

**ユイ**: 「これは`MyType`が`Debug`トレイトを実装してないってこと」

**対処法**:

```rust
#[derive(Debug)]  // これを追加
struct MyType { ... }
```

---

**エラー2: `the size for values of type 'dyn Trait' cannot be known`**

```
error[E0277]: the size for values of type `dyn Draw` cannot be known at compilation time
```

**ユイ**: 「`dyn Trait`はサイズ不明だから直接使えない」

**対処法**:

```rust
// ❌ fn foo() -> dyn Draw
// ✅ fn foo() -> Box<dyn Draw>
```

---

**エラー3: `impl Trait` not allowed outside of function return types**

```
error[E0562]: `impl Trait` not allowed outside of function return types
```

**ユイ**: 「`impl Trait`は構造体のフィールドとかでは使えない」

**対処法**:

```rust
// ❌ struct Foo { handler: impl Handler }
// ✅ struct Foo { handler: Box<dyn Handler> }
// ✅ struct Foo<H: Handler> { handler: H }  // ジェネリクスで解決
```

---

## レイ「トレイトとジェネリクス、奥が深いね...」

**レイ**: 「最初は難しく感じたけど、『契約書』『型パラメータ』って考えると理解しやすくなった」

**ユイ**: 「良い理解だね。特にトレイト境界は、Rustの型システムの核心部分。ここを理解すると、Rustの設計思想が見えてくる」

**レイ**: 「PythonやTypeScriptと比較しながら学べたから、すごく腑に落ちた」

**ユイ**: 「次の章では、コレクション（VecとかHashMapとか）とイテレータを学ぶ。Iteratorトレイトがバンバン出てくるから、この章の知識が活きてくるよ」

**レイ**: 「楽しみ!」

---

## 次のステップ

[Chapter 06: コレクション](06-collections.md) では、`Vec`、`HashMap`などのコレクション型と、イテレータを使った関数型プログラミングパターンを学びます。イテレータは`Iterator`トレイトを実装しており、この章で学んだ知識が活きてきます。

---

**ユイ**: 「お疲れさま。今日はここまで」

**レイ**: 「ありがとう! トレイトとジェネリクス、完全に理解した...はず!」

**ユイ**: 「（笑）使っていくうちに、さらに深く理解できるようになるよ」
