# Chapter 04: エラーハンドリング

> **この章で学ぶこと**: Result型、`?`演算子、panic — Rustのエラー処理パターン

> 📖 **記号リファレンス**: この章では `?`, `Ok()`, `Err()`, `unwrap()` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## プロローグ：try-catchがない！？

> 👩‍💻 **ユイ**: 「先輩、ファイル読み込みのコード書いてたんですけど…try-catchってどう書くんですか？」
>
> 👩‍🏫 **レイ**: 「あぁ…ユイちゃん、Rustには**try-catchがない**のよ。」
>
> 👩‍💻 **ユイ**: 「えっ！？例外処理ないんですか！？じゃあエラーってどう処理するんですか！？」
>
> 👩‍🏫 **レイ**: （クスッと笑って）「Rustは『例外を投げる』んじゃなくて、『エラーを値として返す』の。これが**めちゃくちゃ優れてる**のよ。」
>
> 👩‍💻 **ユイ**: 「エラーを…値として…？うーん、正直ピンと来ないです…」
>
> 👩‍🏫 **レイ**: 「分かるわ。まず、なぜRustがこのアプローチを選んだのか、プログラミング言語の歴史から話すわね。」

---

## エラー処理の歴史 — なぜ例外は問題なのか

> 👩‍🏫 **レイ**: （ホワイトボードに年表を描き始める）「エラー処理の歴史を振り返ってみましょう。」

```
┌─────────────────────────────────────────────────────────────────┐
│  エラー処理の歴史                                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1970年代: C言語 — 戻り値でエラーを返す                          │
│     int result = open("file.txt");                              │
│     if (result < 0) { /* エラー処理 */ }                        │
│     問題: エラーチェックを忘れる、成功値とエラー値が混在          │
│                                                                 │
│  1990年代: Java — チェック例外（Checked Exception）              │
│     void read() throws IOException { ... }                      │
│     問題: throws宣言の伝播が面倒、例外を握りつぶす人続出         │
│                                                                 │
│  2000年代: C++/Python/JavaScript — 非検査例外                    │
│     どこからでも例外が飛んでくる、catch忘れで本番クラッシュ       │
│                                                                 │
│  2009年: Go言語 — 多値返却 (value, error)                        │
│     value, err := doSomething()                                 │
│     if err != nil { return err }                                │
│     問題: errを無視できてしまう（_ に捨てる人多数）               │
│                                                                 │
│  2015年: Rust 1.0 — Result<T, E>型 + ?演算子                    │
│     型システムでエラーを表現、処理を忘れたらコンパイルエラー      │
│     → これまでの教訓を活かした『集大成』                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「へぇ…！Goの多値返却も『無視できちゃう』のが問題だったんですね。」
>
> 👩‍🏫 **レイ**: 「そうなの。Goでは `_, err := doSomething()` の `_` でエラーを握りつぶせちゃうのよ。Rustの`Result`は**使わないとコンパイラに怒られる**から、この問題がない。」
>
> 👩‍💻 **ユイ**: 「言語が強制してくれるんですね…安心感ある…」

---

## Rustのエラー処理哲学 — なぜ例外がないのか

> 👩‍🏫 **レイ**: （ホワイトボードに向かう）「Pythonとかのtry-catchって、実は問題があるのよ。」

### 例外の問題点

> 👩‍🏫 **レイ**: 「ホワイトボードに書くわね。」

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

> 👩‍💻 **ユイ**: 「確かに…Pythonだと、どの関数がどんなエラー投げるか分からないですよね…。」
>
> 👩‍🏫 **レイ**: 「そうでしょ？ドキュメント読むか、実際にエラーが起きないと分からないのよ。」
>
> 👩‍💻 **ユイ**: 「前にそれで本番環境でエラー出ちゃって…冷や汗かきました…。」

### Rustの解決策

> 👩‍🏫 **レイ**: 「Rustはこうするの。」

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

> 👩‍💻 **ユイ**: 「戻り値の型に`Result`って書いてある…！」
>
> 👩‍🏫 **レイ**: 「そう！関数のシグネチャを見れば、『この関数はエラーを返すかもしれない』って一目で分かるの。」
>
> 👩‍💻 **ユイ**: 「それは安心ですね…！」

**比較:**

| 言語 | エラー処理方法 | 問題点 |
|-----|--------------|-------|
| Python | `try`/`except` | どの関数が例外を投げるか分からない |
| JavaScript | `try`/`catch` | 同上 + 非同期エラーが複雑 |
| Java | チェック例外 | 冗長、例外が伝播 |
| Go | 多値返却 | `err`の無視が可能 |
| Rust | `Result<T, E>` | **エラーの可能性が型に現れる** |

> 👩‍💻 **ユイ**: 「型で表現するから、コンパイラがチェックしてくれるんですね！」

---

## Result型 — 成功と失敗を表す列挙型

> 👩‍🏫 **レイ**: 「`Result<T, E>`は、前に学んだ`Option`と似た列挙型よ。」

```rust
enum Result<T, E> {
    Ok(T),   // 成功: 値Tを持つ
    Err(E),  // 失敗: エラーEを持つ
}
```

> 👩‍💻 **ユイ**: 「あっ、`Option`の`Some`/`None`みたいに、`Ok`/`Err`があるんですね！」
>
> 👩‍🏫 **レイ**: 「完璧！実際に使ってみましょう。」

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

> 👩‍💻 **ユイ**: 「`match`でパターンマッチするんですね。」
>
> 👩‍🏫 **レイ**: 「そう。`Option`と同じパターンよ。型を分解してみるわね。」

### 各部分を読み解く

> 👩‍🏫 **レイ**: （ホワイトボードに書く）

```
Result<File, std::io::Error>
   ↓      ↓           ↓
Result<成功時の型, 失敗時の型>

Ok(file)  → 成功時、File型の値が入っている
Err(error) → 失敗時、io::Error型の値が入っている
```

> 👩‍💻 **ユイ**: 「成功と失敗の両方の型を指定するんですね。」
>
> 👩‍🏫 **レイ**: 「そう。TypeScriptで同じことをやろうとすると…」

**TypeScript:**

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

> 👩‍💻 **ユイ**: 「TypeScriptだと自分で型を定義しないといけないんですね。」
>
> 👩‍🏫 **レイ**: 「そう。Rustは標準で用意されてるから楽なのよ。」

---

## `?`演算子 — エラーの伝播を1文字で

> 👩‍🏫 **レイ**: 「さて、ここからが本番よ。**`?`演算子**。これがRustのエラー処理の真骨頂なの。」
>
> 👩‍💻 **ユイ**: 「クエスチョンマーク…？」

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

> 👩‍💻 **ユイ**: 「`?`って何してるんですか？」
>
> 👩‍🏫 **レイ**: （ホワイトボードに図を描く）「詳しく説明するわね。実はこの`?`、最初からあったわけじゃないのよ。」

### `?`演算子の歴史 — try!マクロからの進化

> 👩‍🏫 **レイ**: 「2016年まで、Rustには`try!`マクロっていうのがあったの。」

```rust
// Rust 1.13以前（2016年まで）
fn read_username() -> Result<String, io::Error> {
    let mut file = try!(File::open("username.txt"));  // try!マクロ
    let mut username = String::new();
    try!(file.read_to_string(&mut username));
    Ok(username)
}

// Rust 1.13以降（RFC 243で導入）
fn read_username() -> Result<String, io::Error> {
    let mut file = File::open("username.txt")?;  // ?演算子
    let mut username = String::new();
    file.read_to_string(&mut username)?;
    Ok(username)
}
```

> 👩‍💻 **ユイ**: 「`try!(...)`から`?`に変わったんですね！確かに`?`の方が圧倒的にスッキリ…」
>
> 👩‍🏫 **レイ**: 「そう。RFC 243で『`?`演算子を導入しよう』って提案があって、コミュニティで議論の末に採用されたの。**人間工学（ergonomics）**を重視した結果よ。」

### `?`の動作の詳細

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

> 👩‍💻 **ユイ**: 「えっ、`match`を省略してるんですか！？」
>
> 👩‍🏫 **レイ**: 「そう！`Ok`なら中身を取り出して、`Err`なら即座に関数から`return`するの。**たった1文字で**。」
>
> 👩‍💻 **ユイ**: 「すごい…！めっちゃ便利じゃないですか！」
>
> 👩‍🏫 **レイ**: 「ちなみに、この`?`はSwiftの`try?`やKotlinの`?:`（Elvis演算子）にも影響を与えたと言われてるわ。Rustのエラー処理は、後発の言語のお手本になってるのよ。」

### `?`なしで書くとどうなるか

> 👩‍🏫 **レイ**: 「比較してみましょう。」

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

> 👩‍💻 **ユイ**: 「全然違う…！`?`の方が圧倒的に読みやすいですね。」
>
> 👩‍🏫 **レイ**: 「でしょ？さらに短くもできるわよ。」

### `?`のチェーン

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

> 👩‍💻 **ユイ**: 「えっ、最後のやつ、1行じゃないですか！」
>
> 👩‍🏫 **レイ**: 「標準ライブラリがよく使うパターンを用意してくれてるのよ。」

### `?`が使える条件

> 👩‍💻 **ユイ**: 「じゃあ、どこでも`?`使えるんですか？」
>
> 👩‍🏫 **レイ**: 「いいえ。`?`は**Resultを返す関数内でのみ**使えるの。」

```rust
fn main() {
    let file = File::open("hello.txt")?;  // ❌ エラー！
    // main()はデフォルトでResultを返さない
}
```

> 👩‍💻 **ユイ**: 「あっ、`main`関数では使えないんですね…。」
>
> 👩‍🏫 **レイ**: 「でも、解決策があるわ。」

**解決策:**

```rust
// main関数もResultを返せる
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let file = File::open("hello.txt")?;  // ✅ OK
    println!("ファイルを開きました: {:?}", file);
    Ok(())
}
```

> 👩‍💻 **ユイ**: 「`main`の戻り値を`Result`にすればいいんですね！」
>
> 👩‍🏫 **レイ**: 「そう。`Ok(())`は『成功、でも返す値はない』って意味よ。」

---

## 複数のエラー型を扱う

> 👩‍💻 **ユイ**: 「ところで、違う種類のエラーが出るときはどうするんですか？」
>
> 👩‍🏫 **レイ**: 「いい質問ね。実際のコードだと、いろんなエラーが混ざるのよ。」

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

> 👩‍💻 **ユイ**: 「あっ、`io::Error`と`ParseIntError`が混ざってる…！」
>
> 👩‍🏫 **レイ**: 「そうなのよ。解決策がいくつかあるわ。」

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

> 👩‍💻 **ユイ**: 「`Box<dyn Error>`…？」
>
> 👩‍🏫 **レイ**: 「『Errorトレイトを実装するあらゆる型』を受け入れる、っていう型よ。詳しくはトレイトの章で説明するけど、今は『どんなエラーでも入れられる箱』って思っておいて。」

### 解決策2: `thiserror`クレート（実務で推奨）

> 👩‍🏫 **レイ**: 「実務では`thiserror`クレートを使うのが一般的よ。」

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

> 👩‍💻 **ユイ**: 「`#[from]`で自動変換してくれるんですね！便利…！」
>
> 👩‍🏫 **レイ**: 「でしょ？エラーメッセージもカスタマイズできるし、実務では必須のクレートよ。」

### 解決策3: `anyhow`クレート（アプリケーション向け）

> 👩‍🏫 **レイ**: 「アプリケーション開発なら、`anyhow`も便利よ。」

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

> 👩‍💻 **ユイ**: 「`.context()`でエラーメッセージを追加できるんですね。」
>
> 👩‍🏫 **レイ**: 「そう。エラーが起きたとき、『どこで何が起きたか』が分かりやすくなるの。」

**thiserror vs anyhow の使い分け:**

| 用途 | 推奨クレート | 理由 |
|-----|-------------|------|
| ライブラリ開発 | `thiserror` | 利用者が具体的なエラー型を知れる |
| アプリケーション開発 | `anyhow` | エラー処理が簡潔、コンテキスト追加が容易 |

---

## panic! — 回復不能なエラー

> 👩‍🏫 **レイ**: 「最後に、`panic!`について。どうしても回復できないエラーはこれを使うわ。」

```rust
fn main() {
    panic!("Something went terribly wrong!");
}
```

> 👩‍💻 **ユイ**: 「これ実行したら…？」
>
> 👩‍🏫 **レイ**: 「プログラムがクラッシュするわ。」

```
thread 'main' panicked at 'Something went terribly wrong!', src/main.rs:2:5
```

### いつpanicを使うか

> 👩‍🏫 **レイ**: （ホワイトボードに書く）「使い分けのルールはこうよ。」

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

> 👩‍💻 **ユイ**: 「『回復できるか』で判断するんですね。」
>
> 👩‍🏫 **レイ**: 「そう。基本は`Result`を使って、本当にバグのときだけ`panic`よ。」

---

## unwrapとexpect — 手軽だが危険

> 👩‍💻 **ユイ**: 「あの、`unwrap()`ってよく見るんですけど、あれって何ですか？」
>
> 👩‍🏫 **レイ**: 「あぁ、`unwrap()`ね。手軽だけど**危険**なメソッドよ。」

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

> 👩‍💻 **ユイ**: 「`unwrap()`したら、`None`や`Err`のときにpanicするんですね…。」
>
> 👩‍🏫 **レイ**: 「そう。だから**本番コードでは基本的に避けるべき**よ。」

### unwrapを使っても良い場面

> 👩‍🏫 **レイ**: 「でも、使ってもいい場面もあるわ。」

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

> 👩‍💻 **ユイ**: 「テストとか、確実に成功する場合ならいいんですね。」

### より安全な代替手段

```rust
fn main() {
    let x: Option<i32> = None;

    // unwrap_or: デフォルト値を指定
    let value = x.unwrap_or(0);  // 0

    // unwrap_or_default: 型のデフォルト値を使用
    let value = x.unwrap_or_default();  // 0 (i32のデフォルト)

    // if let / match: パターンマッチ
    if let Some(v) = x {
        println!("値: {}", v);
    } else {
        println!("値がありません");
    }
}
```

> 👩‍💻 **ユイ**: 「`unwrap_or()`なら安全ですね！」

---

## まとめ

> 👩‍🏫 **レイ**: 「じゃあ、今日のまとめよ。」

| 概念 | Python/JS | Rust |
|-----|-----------|------|
| エラー表現 | 例外（Exception） | `Result<T, E>`型 |
| エラー伝播 | `raise`/`throw` | `?`演算子 |
| 値がない | `None`/`null` | `Option<T>` |
| 回復不能エラー | — | `panic!` |
| エラーキャッチ | `try`/`except`/`catch` | `match`/`if let` |

> 👩‍💻 **ユイ**: 「最初は『try-catchがない』って聞いて不安でしたけど、`Result`と`?`で十分ですね！」
>
> 👩‍🏫 **レイ**: 「でしょ？むしろ、型でエラーの可能性が分かるから、Pythonより**安全**なのよ。」
>
> 👩‍💻 **ユイ**: 「`?`演算子、めっちゃ便利ですね。たった1文字でエラー伝播できるなんて…。」
>
> 👩‍🏫 **レイ**: 「そう。Rustのエラー処理は、最初は慣れないかもしれないけど、慣れたら**手放せなくなる**わよ。コンパイラが『エラー処理忘れてるわよ』って教えてくれるから、実行時エラーが激減するの。」
>
> 👩‍💻 **ユイ**: 「それは安心ですね…！」

---

## ポイント

- **エラーは型で表現** — 関数のシグネチャを見ればエラーの可能性が分かる
- **`?`演算子** — エラー伝播を1文字で表現、最重要
- **Result<T, E>** — 成功(Ok)と失敗(Err)を持つ列挙型
- **網羅的な処理** — コンパイラがエラー処理忘れを検出
- **panic vs Result** — 回復可能かどうかで使い分け
- **thiserror/anyhow** — 実務では必須のクレート
- **`unwrap()`は避ける** — テストやプロトタイプ以外では使わない

---

## 次のステップ

> 👩‍🏫 **レイ**: 「次は**トレイト・ジェネリクス**よ。Rustの抽象化メカニズムを学ぶわ。」
>
> 👩‍💻 **ユイ**: 「さっきの`Error`トレイトとか、`From`トレイトの仕組みが分かるんですね！」
>
> 👩‍🏫 **レイ**: 「その通り！楽しみにしててね。」

[Chapter 05: トレイト・ジェネリクス](05-traits-generics-dialogue.md) では、Rustの抽象化メカニズムを学びます。`Error`トレイトの実装で使った`fmt::Display`や`From`トレイトの仕組みを深く理解できます。
