# Chapter 00: 導入・環境構築

> **この章で学ぶこと**: Rustとは何か、なぜ学ぶ価値があるのか、開発環境の構築方法

---

## なぜRustを学ぶのか

あなたはPythonやTypeScriptで開発してきた経験があります。これらは素晴らしい言語ですが、Rustを学ぶ理由は何でしょうか？

### Rustが選ばれる3つの理由

**1. パフォーマンス**

PythonやJavaScriptはインタプリタ言語であり、実行時にコードを解釈します。一方、Rustはコンパイル言語であり、ネイティブコードに直接変換されます。

```
実行速度の目安（相対値）:
C/C++/Rust : 1x（最速）
Go/Java    : 2-5x
JavaScript : 10-50x
Python     : 50-100x
```

**2. メモリ安全性（ガベージコレクタなし）**

PythonやJavaScriptはGC（ガベージコレクタ）がメモリを自動管理します。便利ですが、GCが動作するタイミングで一瞬プログラムが止まることがあります（Stop-the-World）。

Rustは「所有権システム」という仕組みで、**コンパイル時に**メモリの解放タイミングを決定します。つまり：
- GCによる予期しない停止がない
- メモリリークが起きにくい
- C/C++のような手動メモリ管理のバグ（ダングリングポインタ、二重解放）もコンパイラが防ぐ

**3. 実用的なユースケース**

| 分野 | なぜRust？ | 代表例 |
|-----|----------|-------|
| Web開発 | 高速なAPI、低レイテンシ | Axum, Actix-web |
| CLI開発 | シングルバイナリ配布、起動が速い | ripgrep, bat, exa |
| WebAssembly | ブラウザでネイティブ級の速度 | Figma, Cloudflare Workers |
| **Web3/ブロックチェーン** | **安全性とパフォーマンスが必須** | **Solana, Polkadot** |
| システム開発 | OS、ブラウザエンジン | Linux kernel, Firefox |

🎯 **ポイント**: 特にWeb3（Solana）では、スマートコントラクトの実行コストが直接手数料に影響するため、Rustの効率性が重要になります。

---

## 🔄 開発ツールの比較

### パッケージマネージャ・ビルドツール

| 役割 | Python | Node.js/TS | Rust |
|-----|--------|------------|------|
| 言語バージョン管理 | pyenv | nvm | **rustup** |
| パッケージマネージャ | pip | npm/yarn | **Cargo** |
| プロジェクト定義 | pyproject.toml | package.json | **Cargo.toml** |
| ロックファイル | requirements.txt / poetry.lock | package-lock.json | **Cargo.lock** |
| ビルド・実行 | python / pytest | npm run | **cargo build / cargo run** |
| フォーマッタ | black / ruff | prettier | **rustfmt** |
| リンター | ruff / pylint | eslint | **clippy** |

💡 **Tips**: Rustでは`Cargo`が「パッケージマネージャ」「ビルドツール」「テストランナー」「ドキュメント生成」をすべて担当します。npm + webpack + jest を1つにまとめたようなものです。

---

## 環境構築

### Step 1: rustupのインストール

```bash
# macOS / Linux
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# インストール後、シェルを再起動するか以下を実行
source $HOME/.cargo/env
```

rustupは「Rustのバージョンマネージャ」です。pyenvやnvmと同じ役割を持ちます。

```bash
# バージョン確認
rustc --version   # コンパイラ
cargo --version   # パッケージマネージャ/ビルドツール

# 最新版に更新
rustup update
```

### Step 2: エディタの設定（VS Code）

VS Codeを使う場合、以下の拡張機能をインストールしてください：

1. **rust-analyzer** — LSP（コード補完、エラー表示、リファクタリング）
2. **Even Better TOML** — Cargo.tomlのシンタックスハイライト
3. **CodeLLDB** — デバッガ（オプション）

```
💡 Tips: rust-analyzerはPylanceやTypeScriptのtsserverに相当します。
   非常に強力で、型の推論結果をインラインで表示してくれます。
```

---

## 最初のプロジェクトを作る

### プロジェクトの作成

```bash
# 新しいプロジェクトを作成
cargo new hello_rust
cd hello_rust
```

🔄 **比較**:
- Python: `mkdir project && cd project && python -m venv .venv`
- Node.js: `mkdir project && cd project && npm init -y`
- Rust: `cargo new project`（一発で完了）

### プロジェクト構造

```
hello_rust/
├── Cargo.toml    # package.json / pyproject.toml に相当
├── Cargo.lock    # 自動生成（コミット対象）
└── src/
    └── main.rs   # エントリーポイント
```

**Cargo.toml の中身**:

```toml
[package]
name = "hello_rust"
version = "0.1.0"
edition = "2021"        # Rustのエディション（言語仕様のバージョン）

[dependencies]
# ここに依存クレート（ライブラリ）を追加
```

**src/main.rs の中身**:

```rust
fn main() {
    println!("Hello, world!");
}
```

### ビルドと実行

```bash
# コンパイル + 実行（開発用、最適化なし）
cargo run

# コンパイルのみ
cargo build

# リリースビルド（最適化あり、本番用）
cargo build --release

# テスト実行
cargo test

# コードフォーマット
cargo fmt

# 静的解析（リンター）
cargo clippy
```

🔄 **比較**:
```bash
# Python
python main.py          # 実行
pytest                   # テスト
black .                  # フォーマット
ruff check .             # リンター

# Node.js/TypeScript  
npm run dev              # 実行
npm test                 # テスト
npx prettier --write .   # フォーマット
npx eslint .             # リンター

# Rust（Cargoがすべて統一）
cargo run                # 実行
cargo test               # テスト
cargo fmt                # フォーマット
cargo clippy             # リンター
```

---

## 依存クレートの追加

Rustでは外部ライブラリを「クレート（crate）」と呼びます。

```bash
# 依存クレートを追加
cargo add serde          # npm install serde と同じ

# 開発時のみの依存
cargo add --dev tokio-test
```

または `Cargo.toml` を直接編集：

```toml
[dependencies]
serde = "1.0"
serde_json = "1.0"

[dev-dependencies]
tokio-test = "0.4"
```

クレートの検索は [crates.io](https://crates.io/) で行います（npmjs.com、PyPIに相当）。

---

## Hello, World! を理解する

```rust
fn main() {
    println!("Hello, world!");
}
```

この短いコードにもRustの特徴が現れています：

| 要素 | 説明 | Python/TSとの違い |
|-----|------|------------------|
| `fn` | 関数定義キーワード | `def`（Python）、`function`（JS） |
| `main` | エントリーポイント（必須） | Pythonは `if __name__ == "__main__":`、Node.jsは任意 |
| `println!` | マクロ（`!`が目印） | 関数ではなくマクロ。コンパイル時に展開される |
| `;` | 文の終端（必須） | Pythonは不要、JSは省略可能 |

⚠️ **注意**: `println!`の`!`はマクロを表します。Rustでは関数とマクロを明確に区別します。マクロはコンパイル時にコードを生成する強力な機能です。

---

## コンパイルエラーを恐れない

Rustのコンパイラは非常に厳格ですが、それは「実行時エラーを事前に防ぐ」ためです。

```rust
fn main() {
    let x = 5;
    x = 10;  // エラー！
}
```

```
error[E0384]: cannot assign twice to immutable variable `x`
 --> src/main.rs:3:5
  |
2 |     let x = 5;
  |         - first assignment to `x`
3 |     x = 10;
  |     ^^^^^^ cannot assign twice to immutable variable
  |
help: consider making this binding mutable
  |
2 |     let mut x = 5;
  |         +++
```

🎯 **ポイント**: Rustのエラーメッセージは非常に親切です。何が問題で、どう直せばいいかまで教えてくれます。「コンパイラと対話しながらコードを書く」のがRust流です。

---

## まとめ

| 項目 | 内容 |
|-----|------|
| Rustの強み | パフォーマンス、メモリ安全性、モダンなツールチェーン |
| rustup | Rustのバージョンマネージャ（pyenv/nvmに相当） |
| Cargo | パッケージマネージャ + ビルドツール + テストランナー |
| Cargo.toml | プロジェクト定義ファイル（package.json/pyproject.tomlに相当） |
| crates.io | クレート（ライブラリ）のレジストリ |
| rust-analyzer | VS Code拡張（LSP） |

---

## 次のステップ

[Chapter 01: 型と変数](01-basics.md) では、Rustの型システムと変数について学びます。TypeScriptの型システムとの共通点・相違点を中心に解説します。
