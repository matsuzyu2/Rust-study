# Rust学習教材 — Python/TypeScript経験者のための実践ガイド

> **対象読者**: Python・TypeScript経験者（約1年）、CS基礎知識あり  
> **ゴール**: Rust基礎 → Web開発（Axum） → Solana/Anchor開発  
> **形式**: 読み物中心、各章3000〜4000字

---

## 🎯 この教材の特徴

1. **🆕 対話形式で学ぶ**: 新人エンジニア「ユイ」と先輩エンジニア「レイ」の掛け合いを通じて、Rustの概念をストーリー仕立てで学べます。現場の空気感を感じながら、"なぜ？" を深掘りできます
2. **比較で学ぶ**: Python/TypeScriptとの対比で「なぜRustはこうなっているのか」を理解
3. **段階的難易度**: 基礎 → 中級 → 応用（Web/Web3）の順で構成
4. **実務指向**: 最終的にWeb APIとSolanaスマートコントラクトが書けるレベルを目指す

---

## 📚 目次

### Part 1: Rust基礎（Ch.00〜04）

| 章 | タイトル | 概要 |
|:--:|---------|------|
| [00](chapters/00-introduction-dialogue.md) | **導入・環境構築**<br><sub>[📄解説モード](chapters/00-introduction.md)</sub> | rustup, cargo, プロジェクト構造。npm/pipとの比較 |
| [01](chapters/01-basics-dialogue.md) | **型と変数**<br><sub>[📄解説モード](chapters/01-basics.md)</sub> | 静的型付け, 型推論, mut, シャドーイング |
| [02](chapters/02-ownership-dialogue.md) | **所有権**<br><sub>[📄解説モード](chapters/02-ownership.md)</sub> | 所有権, 借用, ライフタイム。Rust最大の特徴 |
| [03](chapters/03-structs-enums-dialogue.md) | **構造体・列挙型**<br><sub>[📄解説モード](chapters/03-structs-enums.md)</sub> | struct, enum, impl, パターンマッチ |
| [04](chapters/04-error-handling-dialogue.md) | **エラーハンドリング**<br><sub>[📄解説モード](chapters/04-error-handling.md)</sub> | Result, Option, `?`演算子 |

### Part 2: Rust中級（Ch.05〜08）

| 章 | タイトル | 概要 |
|:--:|---------|------|
| [05](chapters/05-traits-generics-dialogue.md) | **トレイト・ジェネリクス**<br><sub>[📄解説モード](chapters/05-traits-generics.md)</sub> | trait, impl, ジェネリクス, トレイト境界 |
| [06](chapters/06-collections-dialogue.md) | **コレクション**<br><sub>[📄解説モード](chapters/06-collections.md)</sub> | Vec, HashMap, イテレータ, クロージャ |
| [07](chapters/07-modules-dialogue.md) | **モジュール**<br><sub>[📄解説モード](chapters/07-modules.md)</sub> | mod, use, pub, crate, ワークスペース |
| [08](chapters/08-async-dialogue.md) | **非同期処理**<br><sub>[📄解説モード](chapters/08-async.md)</sub> | async/await, tokio, Future |

### Part 3: 実践応用（Ch.09〜10）

| 章 | タイトル | 概要 |
|:--:|---------|------|
| [09](chapters/09-web-development-dialogue.md) | **Web開発**<br><sub>[📄解説モード](chapters/09-web-development.md)</sub> | Axum, ルーティング, ミドルウェア, JSON API |
| [10](chapters/10-solana-anchor-dialogue.md) | **Solana/Anchor**<br><sub>[📄解説モード](chapters/10-solana-anchor.md)</sub> | Anchor, Program構造, PDA, CPI |

### 付録

| 章 | タイトル | 概要 |
|:--:|---------|------|
| [11](chapters/11-syntax-reference-dialogue.md) | **記号・構文リファレンス**<br><sub>[📄解説モード](chapters/11-syntax-reference.md)</sub> | Rustの記号や構文の早見表。わからない時にサッと確認 |

---

## 🛠 学習の進め方

```
推奨順序: Ch.00 → Ch.01 → Ch.02（重要！）→ Ch.03〜08 → Ch.09 or Ch.10
```

- **Chapter 02（所有権）は最重要**: ここを理解できればRustの8割は理解したも同然
- **Part 1を終えたら**: 簡単なCLIツールが書けるレベル
- **Part 2を終えたら**: ライブラリのコードが読めるレベル
- **Part 3を終えたら**: Web API・スマートコントラクトの入口に立てる

### 📖 対話形式 vs 解説モード

- **対話形式（デフォルト）**: ストーリー仕立てで読みやすい。初めて学ぶ時におすすめ
- **解説モード**: 辞書的にサッと参照したい時、復習する時に便利

両方のリンクを用意しているので、用途に応じて使い分けてください

---

## 🔗 関連リソース

- [The Rust Programming Language（公式Book）](https://doc.rust-lang.org/book/)
- [Rust by Example](https://doc.rust-lang.org/rust-by-example/)
- [Axum公式ドキュメント](https://docs.rs/axum/latest/axum/)
- [Anchor公式ドキュメント](https://www.anchor-lang.com/)
- [Solana Cookbook](https://solanacookbook.com/)

---

## 📝 凡例

本教材では以下の記法を使用します：

```
💡 Tips: 知っておくと便利な情報
⚠️ 注意: よくあるミスや落とし穴
🔄 比較: Python/TypeScriptとの比較
🎯 ポイント: 特に重要な概念
```

---

**最終更新**: 2026年1月（対話形式版リリース）
**Author**: GitHub Copilot（Claude Sonnet 4.5）
