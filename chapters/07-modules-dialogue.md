# Chapter 07: モジュール（対話形式）

> **この章で学ぶこと**: mod、use、pub、クレート、ワークスペース — コードの構造化と可視性

---

## モジュールシステムって何だろう？

**レイ**: ねえユイちゃん、Rustのプロジェクトが大きくなってきたんだけど、コードをどうやって整理したらいいのかな？今は全部main.rsに詰め込んでるんだけど...

**ユイ**: あー、それはモジュールシステムを使うといいよ！TypeScriptでファイルを分けて`import`するみたいに、Rustでもコードを整理できるんだ。

**レイ**: TypeScriptのimportみたいなものがあるの？

**ユイ**: そう！でも、Rustのモジュールシステムはちょっと独特なんだ。まず、全体像を見てみようか。

| 要素 | 説明 | Python相当 | JS/TS相当 |
|-----|------|-----------|----------|
| クレート（Crate） | コンパイル単位 | パッケージ | パッケージ |
| モジュール（Module） | 名前空間 | モジュール | ファイル/名前空間 |
| パス（Path） | 要素への参照 | `import` | `import` |
| `use` | パスを短縮 | `from ... import` | `import { }` |
| `pub` | 公開 | `__all__` | `export` |

**レイ**: うーん、クレートとモジュールの違いがよくわからないな...

**ユイ**: クレートは「コンパイルの単位」で、npmでいうパッケージみたいなもの。モジュールはその中でコードを整理する「名前空間」だよ。1つのクレートの中に複数のモジュールを作れるんだ。

---

## まずはインラインモジュールから

**ユイ**: 最初は、1つのファイルの中でモジュールを定義する方法を見てみようか。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {
            println!("Added to waitlist");
        }
    }

    mod serving {  // privateモジュール
        fn take_order() {}
    }
}

fn main() {
    // 絶対パス
    crate::front_of_house::hosting::add_to_waitlist();

    // 相対パス
    front_of_house::hosting::add_to_waitlist();
}
```

**レイ**: `mod`っていうキーワードでモジュールを定義するんだね。でも、`pub`が付いてるものと付いてないものがあるけど...？

**ユイ**: いい質問！Rustでは**デフォルトで全部プライベート**なんだ。TypeScriptと真逆だよ。

**レイ**: え、TypeScriptだと何もしなくてもexportできるよね？

**ユイ**: そう。TypeScriptは「exportしないとプライベート」だけど、Rustは「pubを付けないとプライベート」なの。これがRustの大きな特徴の1つ。

```typescript
// TypeScript: デフォルトでプライベート
function privateFunc() {}  // 他のファイルからは見えない
export function publicFunc() {}  // exportで公開

// Rust: デフォルトでプライベート
fn private_func() {}  // 他のモジュールからは見えない
pub fn public_func() {}  // pubで公開
```

**レイ**: なるほど！でも、`serving`モジュールには`pub`が付いてないから、外から使えないってこと？

**ユイ**: その通り！`serving`モジュールは`front_of_house`モジュールの内部でしか使えない。これを「カプセル化」って言うんだけど、Rustは最初から安全性を重視してるんだよね。

---

## ファイルに分けてみよう

**レイ**: でも、全部1つのファイルに書くのは大変そうだよね...

**ユイ**: もちろん、ファイルに分けることもできるよ！こんな感じの構成にしてみようか。

```
src/
├── main.rs
├── front_of_house.rs      # front_of_houseモジュール
└── front_of_house/
    └── hosting.rs         # front_of_house::hostingモジュール
```

**レイ**: これ、TypeScriptのフォルダ構成と似てるね！

**ユイ**: そうだね。でも、Rustにはちょっと「儀式」が必要なんだ。まず**main.rs**を見て。

```rust
mod front_of_house;  // front_of_house.rsまたはfront_of_house/mod.rsを読み込む

fn main() {
    front_of_house::hosting::add_to_waitlist();
}
```

**レイ**: あれ？`import`じゃなくて`mod`なんだ。

**ユイ**: そう！`mod front_of_house;`は「このファイルと同じ階層にある`front_of_house.rs`を読み込んで、モジュールとして宣言してね」っていう意味なの。

次に**front_of_house.rs**:

```rust
pub mod hosting;  // front_of_house/hosting.rsを読み込む
```

**レイ**: ここでも`mod`を使うんだね。

**ユイ**: そう。`mod hosting;`は「`front_of_house`ディレクトリの中の`hosting.rs`を読み込む」という意味。で、最後に**front_of_house/hosting.rs**:

```rust
pub fn add_to_waitlist() {
    println!("Added to waitlist");
}
```

**レイ**: うーん、TypeScriptだったらこうするよね？

```typescript
// TypeScript
// frontOfHouse/hosting.ts
export function addToWaitlist() {
    console.log("Added to waitlist");
}

// index.ts
import { addToWaitlist } from './frontOfHouse/hosting';
```

**ユイ**: そう！TypeScriptは「使う側がimportする」んだけど、Rustは「親モジュールがmod宣言する」んだ。この違いがちょっとトリッキーなんだよね。

**レイ**: なるほど...親モジュールが子を「宣言」するイメージなんだね。

---

## Pythonと比較してみる

**ユイ**: Pythonをやったことある？

**レイ**: うん、ちょっとだけ。

**ユイ**: じゃあ、Pythonとも比較してみようか。Pythonだとこんな感じ:

```
mypackage/
├── __init__.py
├── front_of_house/
│   ├── __init__.py
│   └── hosting.py
```

Pythonの`__init__.py`は「このディレクトリはパッケージですよ」って示すファイルだよね。Rustの`mod.rs`も似た役割だったんだけど、最近は使われなくなってきてる。

**レイ**: え、`mod.rs`って何？

**ユイ**: 古いスタイルだと、こんな構成もあったんだ:

```
src/
├── main.rs
└── front_of_house/
    ├── mod.rs          # 古いスタイル
    └── hosting.rs
```

でも今は、`front_of_house.rs`を使うのが主流だよ。

---

## pubの詳細 — 公開範囲を細かく制御

**レイ**: さっき「デフォルトでプライベート」って言ってたけど、`pub`にも種類があるって本当？

**ユイ**: よく知ってるね！実は`pub`には細かい制御ができるんだ。

```rust
mod outer {
    pub mod inner {
        pub(crate) fn crate_visible() {}     // クレート内で可視
        pub(super) fn parent_visible() {}     // 親モジュールで可視
        pub(in crate::outer) fn path_visible() {} // 指定パスで可視
        pub fn fully_public() {}              // 完全パブリック
    }
}
```

| 可視性 | 説明 |
|-------|------|
| （なし） | プライベート（同一モジュール内のみ） |
| `pub` | 完全パブリック |
| `pub(crate)` | 同一クレート内で可視 |
| `pub(super)` | 親モジュールで可視 |
| `pub(in path)` | 指定パスで可視 |

**レイ**: へー、細かく制御できるんだ。これってTypeScriptにはないよね？

**ユイ**: そう。TypeScriptは「export」か「何もしないか」の2択だけど、Rustは「誰に公開するか」を細かく指定できるの。

**レイ**: `pub(crate)`はよく使うの？

**ユイ**: うん、よく使うよ！「外部には公開したくないけど、プロジェクト内では使いたい」っていう関数やstructに付けるんだ。

```rust
// lib.rs
pub(crate) fn internal_helper() {
    // クレート内でのみ使える内部ヘルパー関数
}

pub fn public_api() {
    internal_helper();  // 内部では使える
}
```

**レイ**: なるほど、内部実装を隠蔽できるんだね。

---

## use — パスを短くする魔法

**レイ**: さっきから`crate::front_of_house::hosting::add_to_waitlist()`みたいな長いパスを使ってるけど、もっと短くできないの？

**ユイ**: できるよ！それが`use`の役割なんだ。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

// useでパスをスコープに持ち込む
use front_of_house::hosting;

fn main() {
    hosting::add_to_waitlist();  // 短くなった！
}
```

**レイ**: あ、これTypeScriptの`import`に似てる！

```typescript
// TypeScript
import { hosting } from './frontOfHouse';

hosting.addToWaitlist();
```

**ユイ**: そうだね。でも、Rustの`use`にはちょっとした「お作法」があるんだ。

### useの慣用的な書き方

**ユイ**: Rustコミュニティでは、`use`の書き方に暗黙のルールがあるんだよ。

```rust
// 関数: 親モジュールまでuseする（推奨）
use std::collections::HashMap;
// HashMap::new() のように使う

// 構造体・列挙型: フルパスでuseする（推奨）
use std::collections::HashMap;
let map = HashMap::new();
```

**レイ**: え、どっちも`HashMap`をuseしてるけど...？

**ユイ**: ごめんごめん、説明が分かりにくかったね。関数の場合はこうするのが推奨なんだ:

```rust
// 関数の場合: 親モジュールまで
use std::collections::hash_map;
hash_map::insert();  // どのモジュールの関数か明確

// 構造体の場合: 型そのもの
use std::collections::HashMap;
let map = HashMap::new();  // 型名だけで十分
```

**レイ**: なんで関数と構造体で書き方が違うの？

**ユイ**: 「どこから来た関数なのか」を明確にするためだよ。例えば、こんな場合を考えてみて。

```rust
use std::fmt::Result;
use std::io::Result;  // エラー！同じ名前

fn function1() -> Result { ... }  // どっちのResult？
```

**レイ**: あ、名前が被っちゃうんだ。

**ユイ**: そう。だから、こうするのが良いスタイル:

```rust
// パターン1: 親モジュールまでuse
use std::fmt;
use std::io;

fn function1() -> fmt::Result { ... }
fn function2() -> io::Result<()> { ... }

// パターン2: asでリネーム
use std::fmt::Result;
use std::io::Result as IoResult;

fn function1() -> Result { ... }
fn function2() -> IoResult<()> { ... }
```

**レイ**: なるほど、明確性を重視してるんだね。

---

## useのテクニック集

**ユイ**: `use`にはいくつか便利な書き方があるから紹介するね。

```rust
// 個別に書く（基本）
use std::cmp::Ordering;
use std::io;

// ネストした書き方（推奨）
use std::{cmp::Ordering, io};

// 同じモジュールから複数
use std::io::{self, Write};  // io と io::Write 両方

// ワイルドカード（非推奨）
use std::collections::*;
```

**レイ**: ワイルドカードって、TypeScriptの`import * as foo`みたいなやつ？

**ユイ**: ちょっと違うかな。

```typescript
// TypeScript
import * as fs from 'fs';  // 名前空間として使う
fs.readFile(...);

// Rust（非推奨）
use std::collections::*;
// HashMap, HashSet など全部がスコープに入る
let map = HashMap::new();  // どこから来たか分かりにくい
```

**レイ**: あー、Rustのワイルドカードは全部ばらまいちゃうんだ。

**ユイ**: そう。だから、基本的には避けるべきなんだ。ただし、テストモジュールでは慣習的に使われるよ。

```rust
#[cfg(test)]
mod tests {
    use super::*;  // 親モジュールの全要素をインポート（テストでは許容）

    #[test]
    fn test_something() {
        // ...
    }
}
```

---

## 再エクスポート — pub useの魔法

**レイ**: ねえ、大規模なライブラリを作るときって、内部構造が複雑になるよね？

**ユイ**: そうそう。そういうときに便利なのが「再エクスポート」、つまり`pub use`なんだ。

```rust
mod internal {
    pub mod deeply {
        pub mod nested {
            pub fn useful_function() {}
        }
    }
}

// 再エクスポート
pub use internal::deeply::nested::useful_function;

// 使う側は短いパスでアクセス可能
// use mycrate::useful_function;
```

**レイ**: あ、これTypeScriptでもよくやるやつだ！

```typescript
// TypeScript
// internal/deeply/nested/useful.ts
export function usefulFunction() {}

// index.ts（再エクスポート）
export { usefulFunction } from './internal/deeply/nested/useful';

// 使う側
import { usefulFunction } from 'mycrate';
```

**ユイ**: そう！でも、Rustの場合は「内部構造を完全に隠蔽する」ことが多いんだ。

### ライブラリのAPI設計例

**ユイ**: 実際のライブラリのAPI設計を見てみようか。

```rust
// lib.rs
mod database;
mod models;
mod utils;

// 公開APIとして再エクスポート
pub use database::Connection;
pub use models::{User, Post};

// utilsは完全に隠蔽（内部でのみ使用）
```

使う側はこうなる:

```rust
use mycrate::{Connection, User, Post};

let conn = Connection::new();
let user = User::new("Alice");
```

**レイ**: おお、内部構造を気にせずにシンプルに使えるんだね。

**ユイ**: そう。これが「公開API設計」の基本なんだ。ライブラリの使いやすさは、この`pub use`の使い方で決まると言っても過言じゃないよ。

---

## クレートの構造を理解しよう

**レイ**: ところで、さっきから「クレート」って言葉が出てくるけど、もう一回詳しく教えてくれない？

**ユイ**: OK！クレートには大きく分けて2種類あるんだ。

### ライブラリクレート vs バイナリクレート

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs     # バイナリクレートのルート
│   └── lib.rs      # ライブラリクレートのルート
```

- **main.rs**: 実行可能ファイルを生成（`cargo run`で実行できる）
- **lib.rs**: ライブラリとして使われる（他のプロジェクトから`use`できる）

**レイ**: 両方あると、どうなるの？

**ユイ**: 両方とも別々のクレートとして扱われるよ。こんな感じ:

```rust
// lib.rs
pub fn greet(name: &str) -> String {
    format!("Hello, {}!", name)
}

// main.rs
use my_project::greet;  // lib.rsの関数を使う

fn main() {
    println!("{}", greet("World"));
}
```

**レイ**: なるほど、ライブラリとして公開しつつ、CLIツールとしても使えるってことか。

**ユイ**: その通り！多くのRustプロジェクトがこの構成を使ってるよ。

### 複数のバイナリを作る

**ユイ**: さらに、複数の実行ファイルを作ることもできるんだ。

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs       # デフォルトのバイナリ
│   └── lib.rs
└── src/bin/
    ├── tool1.rs      # cargo run --bin tool1
    └── tool2.rs      # cargo run --bin tool2
```

**レイ**: これって、npmの`bin`フィールドみたいな感じ？

```json
{
  "bin": {
    "tool1": "./bin/tool1.js",
    "tool2": "./bin/tool2.js"
  }
}
```

**ユイ**: まさにそれ！複数のCLIツールを1つのプロジェクトで管理できるんだよ。

---

## ワークスペース — モノレポを作ろう

**レイ**: ねえ、もっと大規模なプロジェクトで、複数のクレートを一緒に管理したいときはどうするの？

**ユイ**: それが「ワークスペース」だよ！npm workspacesやyarn workspacesに似てるんだ。

```
my_workspace/
├── Cargo.toml          # ワークスペース定義
├── app/
│   ├── Cargo.toml
│   └── src/main.rs
├── core/
│   ├── Cargo.toml
│   └── src/lib.rs
└── utils/
    ├── Cargo.toml
    └── src/lib.rs
```

**ルートのCargo.toml**:
```toml
[workspace]
members = [
    "app",
    "core",
    "utils",
]
```

**app/Cargo.toml**:
```toml
[package]
name = "app"
version = "0.1.0"

[dependencies]
core = { path = "../core" }
utils = { path = "../utils" }
```

**レイ**: これって、monorepo構成だね！

**ユイ**: そう！でも、Rustのワークスペースには便利な機能があるんだ:

1. **依存関係の共有**: 全クレートで同じ`target/`ディレクトリを使う
2. **一括ビルド**: `cargo build`でワークスペース全体をビルド
3. **一括テスト**: `cargo test`でワークスペース全体をテスト

**レイ**: それは便利だね。ビルド時間も節約できそう。

**ユイ**: まさに！大規模プロジェクトでは必須の機能だよ。

---

## 外部クレートを使おう

**レイ**: 他の人が作ったライブラリを使いたいときはどうするの？

**ユイ**: `cargo add`コマンドを使うか、`Cargo.toml`を直接編集するんだ。

```bash
cargo add serde
cargo add serde --features derive
```

または**Cargo.toml**を編集:

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
```

**レイ**: これ、`package.json`みたいなものだね。

```json
{
  "dependencies": {
    "serde": "^1.0.0",
    "serde_json": "^1.0.0"
  }
}
```

**ユイ**: そうだね。で、実際に使うときはこんな感じ:

```rust
use serde::{Deserialize, Serialize};

#[derive(Serialize, Deserialize, Debug)]
struct User {
    name: String,
    age: u32,
}

fn main() {
    let user = User {
        name: String::from("Alice"),
        age: 30,
    };

    let json = serde_json::to_string(&user).unwrap();
    println!("{}", json);  // {"name":"Alice","age":30}
}
```

**レイ**: シンプルだね。でも、`features`って何？

**ユイ**: Rustのクレートは「機能の一部だけ有効にする」ことができるんだ。全部入れるとコンパイル時間が長くなったり、バイナリサイズが大きくなったりするから、必要な機能だけ有効にするんだよ。

**レイ**: なるほど、tree shakingみたいなものか。

---

## プレリュード — 自動インポートの仕組み

**レイ**: ところで、`Vec`とか`String`って、`use`しなくても使えるよね？

**ユイ**: よく気づいたね！それが「プレリュード」なんだ。

```rust
// これらは明示的にuseしなくても使える
fn main() {
    let v: Vec<i32> = Vec::new();        // Vec
    let s: String = String::from("hi");  // String
    let opt: Option<i32> = Some(5);      // Option
    let res: Result<i32, ()> = Ok(10);   // Result
}
```

**レイ**: どうして自動的に使えるの？

**ユイ**: Rustコンパイラが自動的に`std::prelude::v1`をインポートしてくれるんだ。プレリュードには、よく使う型やトレイトが含まれてるよ:

- `Vec`, `String`, `Box`などの型
- `Option`, `Result`などの列挙型
- `Copy`, `Clone`, `Debug`などのトレイト
- `Some`, `None`, `Ok`, `Err`などのバリアント

**レイ**: ライブラリも独自のプレリュードを提供できるの？

**ユイ**: できるよ！例えば、tokioライブラリはこんな感じ:

```rust
use tokio::prelude::*;  // tokioの便利な型を一括インポート
```

でも、最近は「プレリュード」よりも「必要なものを明示的にインポート」する方が好まれてるんだ。

---

## 実践的なプロジェクト構造

**レイ**: じゃあ、実際のWebアプリケーションだと、どんな構造になるの？

**ユイ**: 典型的な構造を見せるね。

```
my_web_app/
├── Cargo.toml
├── src/
│   ├── main.rs           # エントリーポイント
│   ├── lib.rs            # ライブラリルート
│   ├── config.rs         # 設定
│   ├── routes/           # ルーティング
│   │   ├── mod.rs
│   │   ├── users.rs
│   │   └── posts.rs
│   ├── models/           # データモデル
│   │   ├── mod.rs
│   │   ├── user.rs
│   │   └── post.rs
│   ├── services/         # ビジネスロジック
│   │   ├── mod.rs
│   │   └── user_service.rs
│   └── db/               # データベース
│       ├── mod.rs
│       └── connection.rs
└── tests/
    └── integration_tests.rs
```

**レイ**: これって、MVCパターンみたいな感じ？

**ユイ**: そうだね。レイヤーごとにモジュールを分けてるよ。で、**lib.rs**でAPIを公開するんだ:

```rust
// lib.rs
pub mod config;
pub mod routes;
pub mod models;
pub mod services;
pub mod db;

// 公開APIを再エクスポート
pub use config::Config;
pub use db::Connection;
```

**routes/mod.rs**:
```rust
mod users;
mod posts;

pub use users::*;
pub use posts::*;
```

**レイ**: `pub use users::*;`って、ワイルドカードを使ってるけど...？

**ユイ**: そう、これは例外的に許容されるケースだよ。「モジュール内の公開APIを全部上位に持ち上げる」っていう明確な意図があるから。

---

## テストモジュールの書き方

**レイ**: テストコードはどこに書くの？

**ユイ**: Rustでは、テストをモジュールと同じファイルに書くことが多いんだ。

```rust
pub fn add(a: i32, b: i32) -> i32 {
    a + b
}

#[cfg(test)]
mod tests {
    use super::*;  // 親モジュールの全要素をインポート

    #[test]
    fn test_add() {
        assert_eq!(add(2, 3), 5);
    }

    #[test]
    fn test_add_negative() {
        assert_eq!(add(-1, 1), 0);
    }
}
```

**レイ**: `#[cfg(test)]`って何？

**ユイ**: これは「テストビルド時だけコンパイルする」っていう条件付きコンパイルの属性だよ。`cargo build`では無視されて、`cargo test`のときだけコンパイルされるんだ。

**レイ**: なるほど、プロダクションビルドにテストコードが含まれないってことか。

**ユイ**: その通り！JavaScriptだと別ファイルに分けることが多いけど:

```typescript
// add.ts
export function add(a: number, b: number): number {
    return a + b;
}

// add.test.ts
import { add } from './add';

test('add', () => {
    expect(add(2, 3)).toBe(5);
});
```

Rustでは同じファイルに書くのが一般的なんだ。

---

## まとめ — 3言語のモジュールシステム比較

**レイ**: いろいろ学んだけど、頭の中を整理したいな...

**ユイ**: じゃあ、Python・TypeScript・Rustを比較してまとめようか。

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| パッケージ | パッケージ | パッケージ | クレート |
| 名前空間 | モジュール | ファイル | モジュール |
| インポート | `import`/`from` | `import` | `use` |
| 公開 | `__all__` | `export` | `pub` |
| 非公開 | `_prefix` | なし | デフォルト |
| モノレポ | — | workspaces | ワークスペース |

### ポイントの整理

**ユイ**: Rustのモジュールシステムの重要なポイントをまとめるね:

1. **デフォルトはプライベート** — `pub`で明示的に公開する
2. **mod** = モジュール宣言（ファイル読み込み）
3. **use** = パスの短縮（スコープに持ち込み）
4. **pub use** = 再エクスポート（API設計に重要）
5. **ワークスペース** = 複数クレートの管理

**レイ**: TypeScriptと比べると、ちょっと面倒な気もするけど...

**ユイ**: そうだね。でも、この「厳格さ」がRustの安全性につながってるんだ。どの関数が公開されてるか、どのモジュールから来たのか、全部明確になるからね。

**レイ**: 確かに、大規模プロジェクトだと「どこから来た関数か分からない」って問題が起きにくそうだね。

**ユイ**: その通り！最初は面倒に感じるかもしれないけど、慣れてくると「この厳格さが心地いい」って思えるようになるよ。

---

## 実践演習

**レイ**: じゃあ、練習してみようかな。

**ユイ**: いいね！こんな演習をやってみて:

### 演習1: 基本的なモジュール構造

以下の構造でプロジェクトを作ってみよう:

```
calculator/
├── Cargo.toml
└── src/
    ├── main.rs
    ├── lib.rs
    ├── operations/
    │   ├── mod.rs
    │   ├── add.rs
    │   └── multiply.rs
```

**目標**:
- `lib.rs`で公開APIを定義
- `operations`モジュールで加算・乗算を実装
- `main.rs`から使ってみる

### 演習2: 再エクスポートを使う

`lib.rs`で内部構造を隠蔽して、シンプルなAPIを提供してみよう:

```rust
// こう使えるようにする
use calculator::{add, multiply};

fn main() {
    println!("{}", add(2, 3));
    println!("{}", multiply(4, 5));
}
```

**レイ**: おお、やってみる！

**ユイ**: 分からなくなったら、いつでも聞いてね。

---

## トラブルシューティング

**レイ**: あ、エラーが出た...

```
error[E0603]: module `hosting` is private
```

**ユイ**: よくあるエラーだね。これは「hostingモジュールがprivateだよ」って意味。`pub`を付け忘れてない？

**レイ**: あ、本当だ！`pub mod hosting;`にしたら直った。

**ユイ**: 他によくあるエラーも紹介しておくね:

### エラー1: ファイルが見つからない

```
error[E0583]: file not found for module `foo`
```

**原因**: `mod foo;`と書いたのに、`foo.rs`または`foo/mod.rs`が存在しない

**解決**: ファイルを作るか、モジュール名を確認する

### エラー2: 循環参照

```
error[E0369]: cyclic module dependency
```

**原因**: モジュールAがBをインポートして、BがAをインポートしてる

**解決**: モジュール構造を見直す（共通の親モジュールを作るなど）

### エラー3: プライベートなフィールドへのアクセス

```
error[E0451]: field `x` of struct `Point` is private
```

**原因**: structのフィールドがprivate

**解決**: フィールドに`pub`を付けるか、メソッド経由でアクセスする

**レイ**: なるほど、エラーメッセージが親切だね。

**ユイ**: そう、Rustのコンパイラはとても親切なんだ。エラーメッセージをよく読めば、たいてい解決策が分かるよ。

---

## 次のステップ

**レイ**: モジュールシステム、だいぶ分かってきた気がする！

**ユイ**: すごいね！次は非同期処理を学ぼうか。

**レイ**: 非同期処理？JavaScriptの`async`/`await`みたいなやつ？

**ユイ**: そう！Rustにも`async`/`await`があるんだけど、JavaScriptとはちょっと違うんだ。次の[Chapter 08: 非同期処理](08-async.md)で詳しく見ていこう。

**レイ**: 楽しみ！

---

**🎯 この章で学んだこと**:

- ✅ Rustのモジュールシステムは「デフォルトでプライベート」
- ✅ `mod`でモジュール宣言、`use`でパス短縮、`pub`で公開
- ✅ `pub use`で再エクスポート（API設計の要）
- ✅ クレート = パッケージ、モジュール = 名前空間
- ✅ ワークスペースで複数クレートを管理
- ✅ TypeScript/Pythonとの違いを理解

**次回**: [Chapter 08: 非同期処理](08-async.md) — `async`/`await`とtokioランタイムを使った非同期プログラミング
