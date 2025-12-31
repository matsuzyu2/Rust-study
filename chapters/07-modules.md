# Chapter 07: モジュール

> **この章で学ぶこと**: mod、use、pub、クレート、ワークスペース — コードの構造化と可視性

---

## モジュールシステム概要

Rustのモジュールシステムは以下の要素で構成されます：

| 要素 | 説明 | Python相当 | JS/TS相当 |
|-----|------|-----------|----------|
| クレート（Crate） | コンパイル単位 | パッケージ | パッケージ |
| モジュール（Module） | 名前空間 | モジュール | ファイル/名前空間 |
| パス（Path） | 要素への参照 | `import` | `import` |
| `use` | パスを短縮 | `from ... import` | `import { }` |
| `pub` | 公開 | `__all__` | `export` |

---

## モジュールの基本

### インラインモジュール

1つのファイル内でモジュールを定義：

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

### ファイルによるモジュール分割

```
src/
├── main.rs
├── front_of_house.rs      # front_of_houseモジュール
└── front_of_house/
    └── hosting.rs         # front_of_house::hostingモジュール
```

**main.rs**:
```rust
mod front_of_house;  // front_of_house.rsまたはfront_of_house/mod.rsを読み込む

fn main() {
    front_of_house::hosting::add_to_waitlist();
}
```

**front_of_house.rs**:
```rust
pub mod hosting;  // front_of_house/hosting.rsを読み込む
```

**front_of_house/hosting.rs**:
```rust
pub fn add_to_waitlist() {
    println!("Added to waitlist");
}
```

🔄 **比較（Python）**:
```
mypackage/
├── __init__.py
├── front_of_house/
│   ├── __init__.py
│   └── hosting.py
```

🔄 **比較（TypeScript）**:
```
src/
├── index.ts
├── frontOfHouse/
│   ├── index.ts
│   └── hosting.ts
```

---

## 可視性（pub）

Rustでは**デフォルトでプライベート**です。公開するには`pub`を付けます。

```rust
mod my_module {
    // プライベート（モジュール外からアクセス不可）
    fn private_function() {}
    
    // パブリック（モジュール外からアクセス可能）
    pub fn public_function() {}
    
    pub struct PublicStruct {
        pub public_field: i32,     // パブリックフィールド
        private_field: i32,        // プライベートフィールド
    }
}
```

### 可視性の種類

```rust
mod outer {
    pub mod inner {
        pub(crate) fn crate_visible() {}     // クレート内で可視
        pub(super) fn parent_visible() {}     // 親モジュールで可視
        pub(in crate::outer) fn path_visible() {} // 指定パスで可視
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

🔄 **比較（TypeScript）**:
- デフォルト: プライベート（exportなし）
- `export`: パブリック

🔄 **比較（Python）**:
- デフォルト: パブリック
- `_prefix`: 慣習的プライベート
- `__prefix`: 名前マングリング

---

## use — パスの短縮

長いパスを短くするために`use`を使います。

```rust
mod front_of_house {
    pub mod hosting {
        pub fn add_to_waitlist() {}
    }
}

// useでパスをスコープに持ち込む
use front_of_house::hosting;

fn main() {
    hosting::add_to_waitlist();  // 短くなった
}
```

### 慣用的なuseの書き方

```rust
// 関数: 親モジュールまでuseする
use std::collections::HashMap;
// HashMap::new() のように使う

// 構造体・列挙型: フルパスでuseする
use std::collections::HashMap;
let map = HashMap::new();

// 同名の型がある場合: 親モジュールを使う
use std::fmt;
use std::io;

fn function1() -> fmt::Result { ... }
fn function2() -> io::Result<()> { ... }

// または as でリネーム
use std::fmt::Result;
use std::io::Result as IoResult;
```

### 複数のインポート

```rust
// 個別に書く
use std::cmp::Ordering;
use std::io;

// ネストした書き方
use std::{cmp::Ordering, io};

// 同じモジュールから複数
use std::io::{self, Write};  // io と io::Write

// ワイルドカード（非推奨、テスト以外では避ける）
use std::collections::*;
```

🔄 **比較（Python）**:
```python
from std.collections import HashMap
from std import cmp, io
from std.io import *  # 非推奨
```

🔄 **比較（TypeScript）**:
```typescript
import { HashMap } from 'std/collections';
import * as io from 'std/io';
```

---

## 再エクスポート（pub use）

モジュールの内部構造を隠しつつ、特定の要素を公開できます。

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

**ライブラリの公開API設計で重要**：
```rust
// lib.rs
mod database;
mod models;
mod utils;

// 公開APIとして再エクスポート
pub use database::Connection;
pub use models::{User, Post};
```

---

## クレート構造

### ライブラリクレート vs バイナリクレート

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs     # バイナリクレートのルート
│   └── lib.rs      # ライブラリクレートのルート
```

- **main.rs**: 実行可能ファイルを生成
- **lib.rs**: ライブラリとして使われる

両方存在する場合、それぞれ別のクレートとして扱われます。

### 複数のバイナリ

```
my_project/
├── Cargo.toml
├── src/
│   ├── main.rs
│   └── lib.rs
└── src/bin/
    ├── tool1.rs    # cargo run --bin tool1
    └── tool2.rs    # cargo run --bin tool2
```

---

## ワークスペース

複数のクレートを1つのプロジェクトで管理：

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

🔄 **比較（npm）**: モノレポ構成（npm workspaces, yarn workspaces）

---

## 外部クレートの使用

### 依存関係の追加

```bash
cargo add serde
cargo add serde --features derive
```

または**Cargo.toml**を直接編集：

```toml
[dependencies]
serde = { version = "1.0", features = ["derive"] }
serde_json = "1.0"
tokio = { version = "1", features = ["full"] }
```

### 使用例

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

---

## プレリュード

Rustには**プレリュード**という自動的にインポートされるモジュールがあります。

```rust
// これらは明示的にuseしなくても使える
// Option, Result, Vec, String, Box, etc.
// Copy, Clone, Debug, etc.

fn main() {
    let v: Vec<i32> = Vec::new();  // useなしでOK
    let s: String = String::from("hello");
    let opt: Option<i32> = Some(5);
}
```

ライブラリも独自のプレリュードを提供することがあります：

```rust
use tokio::prelude::*;  // tokioの便利な型を一括インポート
```

---

## 実践的なプロジェクト構造

### Webアプリケーション例

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

**lib.rs**:
```rust
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

---

## テストモジュール

テストは`#[cfg(test)]`属性付きのモジュールに書きます。

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

```bash
cargo test
```

---

## まとめ

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| パッケージ | パッケージ | パッケージ | クレート |
| 名前空間 | モジュール | ファイル | モジュール |
| インポート | `import`/`from` | `import` | `use` |
| 公開 | `__all__` | `export` | `pub` |
| 非公開 | `_prefix` | なし | デフォルト |
| モノレポ | — | workspaces | ワークスペース |

🎯 **ポイント**:
- **デフォルトはプライベート** — `pub`で明示的に公開
- **mod** = モジュール宣言（ファイル読み込み）
- **use** = パスの短縮（スコープに持ち込み）
- **pub use** = 再エクスポート（API設計に重要）
- **ワークスペース** = 複数クレートの管理

---

## 次のステップ

[Chapter 08: 非同期処理](08-async.md) では、`async`/`await`とtokioランタイムを使った非同期プログラミングを学びます。JavaScript/Pythonの非同期処理との比較を交えて解説します。
