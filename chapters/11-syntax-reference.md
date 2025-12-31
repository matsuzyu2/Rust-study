# Chapter 11: 記号・構文リファレンス

> **この章の目的**: Rustのコードを読んでいて「この記号は何？」となった時に、すぐに確認できるリファレンスです。各章で学ぶ概念の早見表として活用してください。

---

## 記号一覧（アルファベット順）

### `&` — 参照（借用）

**読み方**: 「〜への参照」「〜を借用」

**意味**: 値の所有権を移動させずに、値を参照（借用）する

```rust
let s = String::from("hello");
let r = &s;  // sへの参照を作成（sの所有権はそのまま）
println!("{}", r);  // 参照経由で値にアクセス
println!("{}", s);  // sもまだ使える
```

**Python/TypeScriptとの違い**:
- Python/TypeScriptでは変数は常に参照（のようなもの）
- Rustでは「所有」と「借用（参照）」を明示的に区別

**関連**: [Chapter 02: 所有権](02-ownership.md)

---

### `&mut` — 可変参照（可変借用）

**読み方**: 「〜への可変参照」「〜を可変で借用」

**意味**: 値を変更可能な状態で借用する

```rust
let mut s = String::from("hello");
let r = &mut s;  // 可変参照を作成
r.push_str(", world");  // 参照経由で値を変更
println!("{}", s);  // "hello, world"
```

**重要なルール**:
```
┌─────────────────────────────────────────────────────────────────┐
│  借用のルール                                                    │
├─────────────────────────────────────────────────────────────────┤
│  1. 不変参照（&T）は同時にいくつでも作れる                        │
│  2. 可変参照（&mut T）は同時に1つだけ                            │
│  3. 不変参照と可変参照は同時に存在できない                        │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 02: 所有権](02-ownership.md)

---

### `'a` — ライフタイム注釈

**読み方**: 「ライフタイム・エー」「ライフタイム・ア」

**意味**: 参照が有効な期間（スコープ）を明示する

```rust
fn longest<'a>(x: &'a str, y: &'a str) -> &'a str {
    if x.len() > y.len() { x } else { y }
}
```

**なぜ必要か**:
```
┌─────────────────────────────────────────────────────────────────┐
│  ライフタイムの目的                                              │
├─────────────────────────────────────────────────────────────────┤
│  「戻り値の参照が、引数のどちらかと同じ期間有効」ということを      │
│  コンパイラに伝える。これにより、ダングリング参照を防止。         │
├─────────────────────────────────────────────────────────────────┤
│  Python/TSでは不要 → GCが参照の有効性を管理するから              │
│  Rustでは必要 → GCなしで参照の安全性をコンパイル時に保証          │
└─────────────────────────────────────────────────────────────────┘
```

**よく見るライフタイム**:
- `'a`, `'b` — 一般的なライフタイムパラメータ
- `'static` — プログラム全体で有効（文字列リテラルなど）
- `'_` — ライフタイムの省略（コンパイラが推論）

**関連**: [Chapter 02: 所有権](02-ownership.md)

---

### `<T>` — 型パラメータ（ジェネリクス）

**読み方**: 「型パラメータT」「ジェネリック型T」

**意味**: 具体的な型を後から決められるパラメータ

```rust
fn identity<T>(x: T) -> T {
    x
}

let num = identity(42);        // T = i32
let text = identity("hello");  // T = &str
```

**よく見るパターン**:
```rust
Vec<T>           // T型の要素を持つベクタ
HashMap<K, V>    // K型のキー、V型の値を持つハッシュマップ
Option<T>        // T型の値があるかもしれない
Result<T, E>     // 成功時はT、失敗時はE
```

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `<T: Trait>` — トレイト境界

**読み方**: 「TはTraitを実装している」

**意味**: 型パラメータTに制約を付ける（特定のトレイトを実装している型のみ許可）

```rust
fn print_debug<T: Debug>(x: T) {
    println!("{:?}", x);
}

// Debug を実装している型のみ渡せる
print_debug(42);      // ✅ i32 は Debug を実装
print_debug(vec![1]); // ✅ Vec<i32> も Debug を実装
```

**複数のトレイト境界**:
```rust
fn foo<T: Display + Debug>(x: T) { ... }  // DisplayとDebugの両方が必要
```

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `::` — パスセパレータ / 関連関数呼び出し

**読み方**: 「ダブルコロン」「パスセパレータ」

**2つの用途**:

**1. モジュールパス**:
```rust
use std::collections::HashMap;  // 標準ライブラリ→collections→HashMap
```

**2. 関連関数（静的メソッド）呼び出し**:
```rust
let s = String::from("hello");  // String型のfrom関連関数を呼び出し
let v = Vec::new();             // Vec型のnew関連関数を呼び出し
```

**`.`との違い**:
```rust
// :: はインスタンスなしで呼び出し（関連関数）
String::from("hello")

// . はインスタンス経由で呼び出し（メソッド）
"hello".to_string()
```

**関連**: [Chapter 07: モジュール](07-modules.md)

---

### `::<Type>` — ターボフィッシュ

**読み方**: 「ターボフィッシュ」

**意味**: 型を明示的に指定する（型推論が効かない場合に使用）

```rust
// 型推論が効かない場合
let numbers: Vec<i32> = vec![1, 2, 3]
    .into_iter()
    .collect();  // collectは何に集めるか分からない

// ターボフィッシュで明示
let numbers = vec![1, 2, 3]
    .into_iter()
    .collect::<Vec<i32>>();

// より短い書き方
let numbers = vec![1, 2, 3]
    .into_iter()
    .collect::<Vec<_>>();  // 要素の型は推論
```

**名前の由来**: `::<>` が魚（🐟）に見えるから

**関連**: [Chapter 06: コレクション](06-collections.md)

---

### `?` — エラー伝播演算子

**読み方**: 「クエスチョンマーク演算子」「try演算子」

**意味**: `Result`または`Option`がエラー/Noneの場合、即座に関数から返す

```rust
fn read_file() -> Result<String, io::Error> {
    let content = fs::read_to_string("file.txt")?;
    // ↑ エラーなら即座にreturn Err(...)
    // 成功ならcontentにString値が入る
    Ok(content.to_uppercase())
}
```

**展開するとこうなる**:
```rust
// file?  は以下と同じ
match file {
    Ok(value) => value,
    Err(e) => return Err(e.into()),
}
```

**関連**: [Chapter 04: エラーハンドリング](04-error-handling.md)

---

### `!` — Never型 / マクロ呼び出し

**2つの用途**:

**1. マクロ呼び出し**:
```rust
println!("Hello");  // println はマクロ
vec![1, 2, 3];      // vec! もマクロ
panic!("Error!");   // panic! もマクロ
```

**2. Never型（関数が返らないことを示す）**:
```rust
fn infinite_loop() -> ! {
    loop {}  // 永遠にループ
}

fn always_panic() -> ! {
    panic!("This never returns")
}
```

---

### `_` — ワイルドカード / プレースホルダ

**3つの用途**:

**1. パターンマッチで「その他」**:
```rust
match value {
    1 => println!("one"),
    2 => println!("two"),
    _ => println!("other"),  // 上記以外すべて
}
```

**2. 使わない変数**:
```rust
let _unused = calculate();  // 警告を抑制

for _ in 0..5 {  // ループ変数を使わない
    println!("Hello");
}
```

**3. 型推論のヒント**:
```rust
let numbers: Vec<_> = vec![1, 2, 3];  // 要素型は推論に任せる
```

---

### `..` — 範囲 / 残りすべて

**3つの用途**:

**1. 範囲**:
```rust
for i in 0..5 { }     // 0, 1, 2, 3, 4（5を含まない）
for i in 0..=5 { }    // 0, 1, 2, 3, 4, 5（5を含む）
```

**2. 構造体更新構文**:
```rust
let user2 = User {
    email: String::from("new@example.com"),
    ..user1  // 残りのフィールドはuser1からコピー
};
```

**3. パターンマッチで「残り」**:
```rust
let (first, .., last) = (1, 2, 3, 4, 5);  // 最初と最後だけ取得
```

---

### `|` — クロージャ引数 / パターンOR

**2つの用途**:

**1. クロージャの引数リスト**:
```rust
let add = |a, b| a + b;       // 引数2つ
let print = || println!("Hi"); // 引数なし
let double = |x| x * 2;        // 引数1つ
```

**2. パターンマッチでOR**:
```rust
match value {
    1 | 2 | 3 => println!("small"),  // 1または2または3
    _ => println!("large"),
}
```

**関連**: [Chapter 06: コレクション](06-collections.md)

---

### `->` — 関数の戻り値型

**読み方**: 「returns」「戻り値は」

**意味**: 関数やクロージャの戻り値の型を指定

```rust
fn add(a: i32, b: i32) -> i32 {
    a + b
}

let closure = |x: i32| -> i32 { x * 2 };
```

---

### `=>` — マッチアーム

**読み方**: 「ならば」「の場合は」

**意味**: パターンマッチで「このパターンなら、この処理」を示す

```rust
match value {
    1 => println!("one"),
    2 => {
        println!("two");
        do_something();
    },
    _ => println!("other"),
}
```

---

## キーワード

### `mut` — 可変

**意味**: 変数を変更可能にする

```rust
let x = 5;      // 不変（変更不可）
let mut y = 5;  // 可変（変更可能）
y = 10;         // ✅ OK
// x = 10;      // ❌ エラー
```

**なぜデフォルトが不変か**:
```
┌─────────────────────────────────────────────────────────────────┐
│  Rustの設計思想                                                  │
├─────────────────────────────────────────────────────────────────┤
│  「変更されない」がデフォルト = バグを防ぐ                        │
│  変更が必要な場合のみ明示的に mut を付ける                        │
│  → 「この変数は変更される可能性がある」と読み手に伝わる           │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 01: 基本構文](01-basics.md)

---

### `impl` — 実装

**2つの用途**:

**1. 型にメソッドを実装**:
```rust
impl Point {
    fn new(x: i32, y: i32) -> Self {
        Point { x, y }
    }
}
```

**2. トレイトを型に実装**:
```rust
impl Display for Point {
    fn fmt(&self, f: &mut Formatter) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
```

**関連**: [Chapter 03: 構造体・列挙型](03-structs-enums.md), [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `impl Trait` — トレイトを実装する型（引数/戻り値）

**意味**: 「Traitを実装している何らかの型」を表す

```rust
// 引数: Summaryを実装している任意の型を受け取る
fn notify(item: &impl Summary) {
    println!("{}", item.summarize());
}

// 戻り値: Iteratorを実装している何かを返す（具体的な型は隠蔽）
fn make_iter() -> impl Iterator<Item = i32> {
    vec![1, 2, 3].into_iter()
}
```

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `dyn Trait` — トレイトオブジェクト

**読み方**: 「ダイナミック・トレイト」

**意味**: 実行時に型が決まる、トレイトを実装した型への参照

```rust
let shapes: Vec<Box<dyn Draw>> = vec![
    Box::new(Circle { radius: 5.0 }),
    Box::new(Rectangle { width: 10.0, height: 5.0 }),
];
```

**impl Trait との違い**:
```
┌─────────────────────────────────────────────────────────────────┐
│  impl Trait vs dyn Trait                                        │
├─────────────────────────────────────────────────────────────────┤
│  impl Trait  コンパイル時に型決定（静的ディスパッチ）、高速       │
│  dyn Trait   実行時に型決定（動的ディスパッチ）、柔軟             │
├─────────────────────────────────────────────────────────────────┤
│  異なる型を同じコレクションに入れる → dyn Trait                  │
│  それ以外 → impl Trait（パフォーマンスが良い）                   │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `move` — 所有権の強制移動

**意味**: クロージャに変数の所有権を移動させる

```rust
let s = String::from("hello");

// moveなし: sを借用
let closure1 = || println!("{}", s);

// moveあり: sの所有権を移動
let closure2 = move || println!("{}", s);
// ここでsは使えなくなる
```

**いつ必要か**:
```
┌─────────────────────────────────────────────────────────────────┐
│  move が必要な場面                                               │
├─────────────────────────────────────────────────────────────────┤
│  1. クロージャを別スレッドに渡す                                  │
│  2. クロージャが元のスコープより長く生存する                      │
│  3. 非同期タスクにクロージャを渡す                                │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 06: コレクション](06-collections.md), [Chapter 08: 非同期](08-async.md)

---

### `async` / `await` — 非同期

**意味**: 非同期関数の定義と、非同期処理の待機

```rust
async fn fetch_data() -> String {
    // 非同期処理
    "data".to_string()
}

async fn process() {
    let data = fetch_data().await;  // 完了を待つ
    println!("{}", data);
}
```

**Python/TypeScriptとの違い**:
```
┌─────────────────────────────────────────────────────────────────┐
│  非同期の違い                                                    │
├─────────────────────────────────────────────────────────────────┤
│  Python/TS: async関数を呼ぶと自動的に実行開始                    │
│  Rust: async関数を呼ぶだけではFutureを作るだけ（遅延評価）        │
│        .await して初めて実行される                               │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 08: 非同期](08-async.md)

---

### `self` / `Self` — インスタンス / 型自身

**意味**:
- `self` — メソッドを呼び出しているインスタンス（小文字）
- `Self` — そのメソッドが実装されている型自身（大文字）

```rust
impl Point {
    // Self = Point型
    fn new(x: i32, y: i32) -> Self {
        Self { x, y }  // Point { x, y } と同じ
    }
    
    // self = このメソッドを呼んでいるPointインスタンス
    fn distance(&self) -> f64 {
        ((self.x.pow(2) + self.y.pow(2)) as f64).sqrt()
    }
}
```

**selfの種類**:
```
┌─────────────────────────────────────────────────────────────────┐
│  self の種類                                                     │
├─────────────────────────────────────────────────────────────────┤
│  self      所有権を奪う。呼び出し後、元の変数は使えない           │
│  &self     不変借用。読み取りのみ。                              │
│  &mut self 可変借用。読み書き可能。                              │
└─────────────────────────────────────────────────────────────────┘
```

**関連**: [Chapter 03: 構造体・列挙型](03-structs-enums.md)

---

### `where` — トレイト境界の分離

**意味**: 複雑なトレイト境界を見やすく書く

```rust
// 読みにくい
fn foo<T: Clone + Debug, U: Clone + Debug>(t: T, u: U) {}

// where句で見やすく
fn foo<T, U>(t: T, u: U)
where
    T: Clone + Debug,
    U: Clone + Debug,
{}
```

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

## 属性（アトリビュート）

### `#[derive(...)]`

**意味**: トレイトの実装を自動生成

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}
```

**よく使うderive**:
| トレイト | 効果 |
|---------|------|
| `Debug` | `{:?}`でデバッグ出力可能 |
| `Clone` | `.clone()`で複製可能 |
| `Copy` | 暗黙のコピー（スタック型のみ） |
| `PartialEq` | `==`で比較可能 |
| `Eq` | 反射律を保証（`a == a`が常にtrue） |
| `Default` | `::default()`でデフォルト値生成 |

**関連**: [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md)

---

### `#[cfg(...)]`

**意味**: 条件付きコンパイル

```rust
#[cfg(test)]
mod tests {
    // cargo test の時だけコンパイル
}

#[cfg(target_os = "windows")]
fn windows_only() {
    // Windowsでのみコンパイル
}
```

---

### `#[allow(...)]` / `#[warn(...)]` / `#[deny(...)]`

**意味**: コンパイラ警告の制御

```rust
#[allow(dead_code)]  // 未使用コードの警告を抑制
fn unused_function() {}

#[deny(unused_variables)]  // 未使用変数をエラーに
fn foo() {
    let x = 5;  // エラー！
}
```

---

## マクロ

### `println!` / `print!`

**意味**: 標準出力に出力

```rust
println!("Hello, {}!", name);  // 改行あり
print!("No newline");           // 改行なし
println!("{:?}", object);       // Debug出力
println!("{:#?}", object);      // Pretty Debug出力
```

---

### `format!`

**意味**: フォーマットした文字列を返す（出力しない）

```rust
let s = format!("Hello, {}!", name);
```

---

### `vec!`

**意味**: ベクタを簡単に作成

```rust
let v = vec![1, 2, 3];  // Vec::from([1, 2, 3]) と同等
let zeros = vec![0; 10];  // 0が10個のベクタ
```

---

### `panic!`

**意味**: プログラムを異常終了させる

```rust
panic!("Something went wrong!");
panic!("Error: {}", error_message);
```

---

### `todo!` / `unimplemented!`

**意味**: 未実装のプレースホルダ

```rust
fn not_yet() -> i32 {
    todo!("後で実装する")
}

fn never_will() {
    unimplemented!("この関数は実装しない")
}
```

---

### `assert!` / `assert_eq!`

**意味**: 条件が満たされなければpanic

```rust
assert!(x > 0);
assert_eq!(a, b);  // a == b でなければpanic
assert_ne!(a, b);  // a != b でなければpanic
```

---

## スマートポインタ

### `Box<T>`

**意味**: ヒープ上に値を置く

```rust
let b = Box::new(5);  // 5をヒープに置く

// 再帰的なデータ構造に必要
enum List {
    Cons(i32, Box<List>),
    Nil,
}
```

---

### `Rc<T>` / `Arc<T>`

**意味**: 参照カウント（複数の所有者）

```rust
use std::rc::Rc;

let a = Rc::new(5);
let b = Rc::clone(&a);  // 参照カウント+1
let c = Rc::clone(&a);  // 参照カウント+1
// a, b, c すべてが同じデータを共有
```

```
┌─────────────────────────────────────────────────────────────────┐
│  Rc vs Arc                                                      │
├─────────────────────────────────────────────────────────────────┤
│  Rc   シングルスレッド用。軽量。                                 │
│  Arc  マルチスレッド用（Atomic Reference Count）。やや重い。     │
└─────────────────────────────────────────────────────────────────┘
```

---

### `RefCell<T>` / `Mutex<T>` / `RwLock<T>`

**意味**: 内部可変性（不変参照経由で値を変更）

```rust
use std::cell::RefCell;

let data = RefCell::new(5);
*data.borrow_mut() = 10;  // 不変参照経由で変更

// マルチスレッド版
use std::sync::Mutex;
let data = Mutex::new(5);
*data.lock().unwrap() = 10;
```

---

## Option と Result

### `Option<T>`

**意味**: 値があるかもしれないし、ないかもしれない

```rust
let some_number: Option<i32> = Some(5);
let no_number: Option<i32> = None;

// 値を取り出す方法
if let Some(n) = some_number {
    println!("{}", n);
}

let n = some_number.unwrap();        // Noneならpanic
let n = some_number.unwrap_or(0);    // Noneなら0
let n = some_number.expect("必要");   // Noneならメッセージ付きpanic
```

---

### `Result<T, E>`

**意味**: 成功か失敗か

```rust
let ok: Result<i32, String> = Ok(5);
let err: Result<i32, String> = Err("error".to_string());

// 値を取り出す方法
match result {
    Ok(value) => println!("Success: {}", value),
    Err(e) => println!("Error: {}", e),
}

let value = result?;  // エラーなら早期リターン
```

---

## よくあるパターン

### if let / while let

**意味**: パターンマッチの簡略版

```rust
// if let
if let Some(x) = option {
    println!("{}", x);
}

// while let
while let Some(x) = iterator.next() {
    println!("{}", x);
}
```

---

### match

**意味**: パターンマッチング

```rust
match value {
    1 => println!("one"),
    2 | 3 => println!("two or three"),
    4..=10 => println!("four to ten"),
    n if n > 10 => println!("big: {}", n),  // ガード条件
    _ => println!("other"),
}
```

---

## 次のステップ

この記号リファレンスは、各章で学ぶ概念を補完するものです。詳しい説明は各章を参照してください：

- [Chapter 01: 基本構文](01-basics.md) — 変数、関数、制御フロー
- [Chapter 02: 所有権](02-ownership.md) — `&`, `&mut`, ライフタイム
- [Chapter 03: 構造体・列挙型](03-structs-enums.md) — `impl`, `self`, `match`
- [Chapter 04: エラーハンドリング](04-error-handling.md) — `?`, `Result`, `Option`
- [Chapter 05: トレイト・ジェネリクス](05-traits-generics.md) — `<T>`, `impl Trait`, `dyn Trait`
- [Chapter 06: コレクション](06-collections.md) — イテレータ、クロージャ
- [Chapter 08: 非同期](08-async.md) — `async`, `await`, `move`
