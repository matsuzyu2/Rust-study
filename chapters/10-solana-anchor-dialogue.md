# Chapter 10: Solana/Anchor — 対話編

> **この章で学ぶこと**: レイとユイが、Solanaブロックチェーン上でのRust開発に挑戦。Ethereumとの違い、Anchorフレームワークの魅力、そして実際にスマートコントラクトを作りながら、Web3開発の世界を体験します。

---

## プロローグ：ユイのWeb3への目覚め

ユイが開発室のドアを開けると、レイは複数のターミナルウィンドウを開いて、いつもより真剣な表情でコードを書いていた。

**ユイ**「レイ先輩、今日は何を作ってるんですか？」

**レイ**「お、ユイちゃん。ちょうどいいところに来た。今Solanaでスマートコントラクト書いてるんだけど、これがめちゃくちゃ面白いんだよ」

**ユイ**「Solana...ブロックチェーンのやつですよね。私、実はちょっとWeb3に興味があって」

**レイ**「へえ、意外。なんでまた？」

**ユイ**「最近、友達がNFTプロジェクトに参加してて楽しそうで。でも、スマートコントラクトってSolidityっていう専用の言語で書くって聞いたんです。また新しい言語を覚えないといけないのかなって...」

**レイ**「それが違うんだよ。SolanaならRustで書けるんだ」

**ユイ**「えっ！　Rustで？　私たち、今まで散々Rust勉強してきたじゃないですか」

**レイ**「そう。だから実はユイちゃん、もうWeb3開発の準備はできてるんだよ。Rustの所有権システムとか、ライフタイムとか、あれ全部スマートコントラクトのセキュリティに直結する概念なの」

**ユイ**「マジですか...！　じゃあ、教えてください！」

**レイ**「よし、今日はSolanaとAnchorフレームワークを使って、実際にスマートコントラクトを作ってみようか」

---

## なぜSolanaなのか？

**レイ**「まず、なぜSolanaを選ぶのか。簡単に比較表を見てみよう」

```
| ブロックチェーン | 主要言語 | 特徴 |
|----------------|---------|------|
| Ethereum | Solidity | 最大のエコシステム、EVM |
| Solana | Rust | 高速（65,000+ TPS）、低手数料 |
| Polkadot | Rust | クロスチェーン、Substrate |
| Near | Rust | シャーディング |
```

**ユイ**「Ethereumが一番有名ですよね。でもSolanaってそんなに速いんですか？」

**レイ**「そう。Ethereumが秒間15トランザクションくらいなのに対して、Solanaは65,000以上。そして手数料が0.00025ドルくらい。Ethereumだと混雑時は数千円とかになることもある」

**ユイ**「えっ、それは全然違いますね...」

**レイ**「しかも、Rustネイティブだから、Rustの型システムや所有権の概念がそのまま使える。これがセキュリティ面で大きなアドバンテージなんだ」

---

## Solanaのアーキテクチャ：Ethereumとの違い

**レイ**「じゃあ、Solanaの基本概念から。まず『アカウントモデル』っていうのを理解する必要がある」

```
┌─────────────────────────────────────────────────────┐
│                    Account                          │
├─────────────────────────────────────────────────────┤
│ lamports: u64        // 残高（SOLの最小単位）          │
│ data: Vec<u8>        // 任意のデータ                  │
│ owner: Pubkey        // このアカウントを所有するProgram │
│ executable: bool     // Programかどうか              │
│ rent_epoch: u64      // レント支払い                  │
└─────────────────────────────────────────────────────┘
```

**ユイ**「アカウント...ウォレットみたいなもの？」

**レイ**「それだけじゃない。Solanaでは『すべてがアカウント』なんだ。ウォレットも、データも、スマートコントラクト自体も全部アカウント」

**ユイ**「うーん、ちょっと抽象的でイメージしにくいです」

**レイ**「じゃあ、従来のWeb開発と比較してみよう」

```
従来のWebアプリ:
┌─────────────┐     ┌─────────────┐
│   サーバー   │────→│  データベース │
│  (Node.js)  │     │  (PostgreSQL)│
└─────────────┘     └─────────────┘
サーバーがDBへの読み書きを制御

Solana:
┌─────────────┐     ┌─────────────┐
│   Program   │────→│   Account   │
│  (Rust)     │     │   (状態)    │
└─────────────┘     └─────────────┘
Programがアカウントデータを変更
※ 誰でもアカウントを「読める」が、「書ける」のは所有者のみ
```

**ユイ**「あ、なるほど！　Node.jsサーバーがProgramで、PostgreSQLがAccountみたいな感じですね」

**レイ**「いい例え。でも大きな違いが一つある。Solanaでは、Programとデータが完全に分離されてるんだ」

**ユイ**「分離...？」

**レイ**「そう。Ethereumと比較すると分かりやすい」

```
┌─────────────────────────────────────────────────────────────────┐
│  EthereumとSolanaのアーキテクチャ比較                            │
├─────────────────────────────────────────────────────────────────┤
│  Ethereum:                                                      │
│  ┌─────────────────────────────────────┐                       │
│  │ Contract                            │                       │
│  │  ├─ コード（バイトコード）            │  ← 一体化             │
│  │  └─ ストレージ（状態変数）            │                       │
│  └─────────────────────────────────────┘                       │
│                                                                 │
│  Solana:                                                        │
│  ┌─────────────────┐     ┌─────────────────┐                   │
│  │ Program         │     │ Account A        │                   │
│  │  └─ コードのみ   │────→│  └─ データ       │  ← 分離            │
│  └─────────────────┘     └─────────────────┘                   │
│           │              ┌─────────────────┐                   │
│           └─────────────→│ Account B        │                   │
│                          │  └─ データ       │                   │
│                          └─────────────────┘                   │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「Ethereumはコードとデータがセットなんですね。でもSolanaは分かれてる」

**レイ**「そう！　この分離が超重要。例えば、100人のユーザーがいたら、Programは1つだけど、アカウントは100個作れる。そして、それぞれのアカウントが独立してるから、並列処理できるんだ」

**ユイ**「あっ、だから速いんだ！　アカウントAとアカウントBを同時に処理できるってことですね」

**レイ**「正解。Ethereumだと基本的にシングルスレッドだけど、Solanaは並列実行可能。これがスループットの違いを生んでる」

---

## 開発環境のセットアップ

**レイ**「じゃあ、実際に手を動かしてみようか。まずは開発環境のセットアップから」

**ユイ**「はい！　何をインストールすればいいですか？」

**レイ**「Rustはもう入ってるから、Solana CLIとAnchor CLIだけ」

```bash
# Rustを最新に
rustup update

# Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# Anchor CLI
cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked

# 確認
solana --version
anchor --version
```

**ユイ**「インストールできました！」

**レイ**「次に、ローカルでテストできるようにネットワークを設定しよう」

```bash
# ローカルネットワークに設定
solana config set --url localhost

# ウォレットを作成
solana-keygen new

# ローカルバリデータを起動（別ターミナルで）
solana-test-validator
```

**ユイ**「バリデータって、ブロックチェーンのノードみたいなものですか？」

**レイ**「そうそう。これで自分のPCにSolanaのブロックチェーンが立ち上がる。ローカルだから好き放題テストできるよ」

---

## Anchorとは何か？

**ユイ**「Anchorっていうのは何なんですか？」

**レイ**「Anchorは、Solana開発のフレームワーク。Next.jsとReactの関係みたいなもの」

**ユイ**「あ、フレームワークなんですね。使うとどう便利になるんですか？」

**レイ**「これを見て」

```
┌─────────────────────────────────────────────────────────────────┐
│  素のSolana開発 vs Anchor                                        │
├─────────────────────────────────────────────────────────────────┤
│  素のSolana（約100行のボイラープレート）:                         │
│  - アカウントの所有者検証を手動で実装                              │
│  - Borshシリアライズ/デシリアライズを手動で実装                    │
│  - 署名者検証を手動で実装                                         │
│  - discriminatorを手動で管理                                     │
│                                                                 │
│  Anchor（マクロで自動生成）:                                      │
│  - #[account] → 所有者検証 + シリアライズ                         │
│  - #[derive(Accounts)] → 必要なアカウント検証                     │
│  - Signer<'info> → 署名者検証                                    │
│  - 8バイトのdiscriminatorを自動管理                               │
│                                                                 │
│  結果: 同じ機能を1/3以下のコードで安全に実装できる                 │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「わあ...手動だと100行も書かないといけないんですか」

**レイ**「しかも、それ全部セキュリティに関わる重要なチェック。一つでも抜けると脆弱性になる。Anchorはマクロでそれを全部自動生成してくれるんだ」

**ユイ**「Chapter 05で学んだマクロが、ここで活きるんですね」

**レイ**「そういうこと。じゃあ、実際にプロジェクトを作ってみよう」

---

## 最初のProgram：Hello Solana

**レイ**「まずはプロジェクト作成から」

```bash
anchor init my_program
cd my_program
```

**ユイ**「できました。ディレクトリ構造はどうなってるんですか？」

```
my_program/
├── Anchor.toml         # プロジェクト設定
├── Cargo.toml
├── programs/
│   └── my_program/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs  # ← Programのコード（ここを編集）
├── tests/
│   └── my_program.ts   # TypeScriptテスト
└── migrations/
    └── deploy.ts
```

**レイ**「メインのコードは`programs/my_program/src/lib.rs`に書く。テストはTypeScriptで書けるようになってるんだ」

**ユイ**「TypeScriptでテスト...？　でもコードはRustですよね？」

**レイ**「そう。フロントエンドはTypeScriptで書くことが多いから、同じ言語でテストできると便利でしょ。Anchorが自動でTypeScriptのクライアントコードも生成してくれるんだ」

**ユイ**「すごい...至れり尽くせり」

**レイ**「じゃあ、`lib.rs`を開いて、Hello Worldを書いてみよう」

```rust
use anchor_lang::prelude::*;

declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");

#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        msg!("Hello, Solana!");
        Ok(())
    }
}

#[derive(Accounts)]
pub struct Initialize {}
```

**ユイ**「これだけですか？　めっちゃシンプルですね」

**レイ**「各部分を解説するね」

---

## コードを読み解く：Anchorの基本構造

**レイ**「まず最初の行」

```rust
use anchor_lang::prelude::*;
```

**レイ**「これはAnchorの基本的な型やマクロを全部インポートしてる。`Context`、`Result`、`Account`とか、よく使うやつ全部」

**ユイ**「`prelude`って、Chapter 08で出てきましたよね」

**レイ**「よく覚えてるね。次はこれ」

```rust
declare_id!("Fg6PaFpoGXkYsidMpWTK6W2BeZ7FEfcYkg476zPFsLnS");
```

**レイ**「これは、このProgramのID、つまりブロックチェーン上のアドレスを宣言してる。`anchor build`すると自動で生成される」

**ユイ**「なんか長いランダムな文字列...」

**レイ**「これで一意に識別できるんだ。じゃあ、メインの部分」

```rust
#[program]
pub mod my_program {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        msg!("Hello, Solana!");
        Ok(())
    }
}
```

**レイ**「`#[program]`っていう属性マクロで、『このモジュールはSolana Programです』って宣言してる。その中に、命令（Instruction）を関数として定義するんだ」

**ユイ**「`initialize`が命令なんですね」

**レイ**「そう。クライアントから『initialize命令を実行して』って送信されると、この関数が呼ばれる」

**ユイ**「引数の`ctx: Context<Initialize>`って何ですか？」

**レイ**「`ctx`は、この命令に必要なアカウント群が入ってるコンテキスト。`Context<Initialize>`の`Initialize`は、必要なアカウントを定義した構造体の名前」

```rust
#[derive(Accounts)]
pub struct Initialize {}
```

**レイ**「今回は空だけど、本来はここに『この命令に必要なアカウント』をリストアップする」

**ユイ**「あ、だからアカウントを事前に宣言するって言ってたんですね」

**レイ**「正解。最後に`msg!`マクロ」

```rust
msg!("Hello, Solana!");
```

**レイ**「これは`println!`みたいなもの。ブロックチェーンのトランザクションログに出力される」

---

## `<'info>`の謎を解く

**ユイ**「そういえば、さっきから気になってたんですけど...」

**レイ**「ん？」

**ユイ**「Anchorのコード見てると、`<'info>`っていうのがよく出てくるじゃないですか。あれ、ライフタイムですよね？」

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(mut)]
    pub user: Signer<'info>,
    pub system_program: Program<'info, System>,
}
```

**レイ**「おお、いいところに気づいた！　Chapter 02で所有権とライフタイムやったよね」

**ユイ**「はい。でも、なんで`'info`って名前なんですか？　`'a`じゃなくて」

**レイ**「それは、この`'info`が特別な意味を持ってるから。図で説明するね」

```
┌─────────────────────────────────────────────────────────────────┐
│  'info ライフタイムの意味                                        │
├─────────────────────────────────────────────────────────────────┤
│  トランザクション処理の流れ:                                     │
│                                                                 │
│  1. クライアントがトランザクションを送信                          │
│     └─ 「これらのアカウントを使って、この命令を実行して」          │
│                                                                 │
│  2. Solanaランタイムがアカウント情報をロード                      │
│     └─ AccountInfo構造体の配列を用意                             │
│     └─ この配列が「'info」の期間だけ有効                          │
│                                                                 │
│  3. Programが実行される                                          │
│     └─ AccountInfo への参照を使ってデータを読み書き                │
│                                                                 │
│  4. トランザクション完了                                          │
│     └─ AccountInfo配列が解放される                                │
│     └─ 'infoライフタイム終了                                      │
│                                                                 │
│  'info = 「1回のトランザクション処理中」という期間を表す           │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「あ！　つまり、`'info`は『トランザクション処理中だけ有効』っていう意味のライフタイムなんですね」

**レイ**「そういうこと！　Anchorの内部実装を見ると」

```rust
// Anchorの内部型（簡略化）
pub struct Account<'info, T> {
    // AccountInfoへの参照を保持
    info: &'info AccountInfo<'info>,
    // デシリアライズされたデータ
    account: T,
}

pub struct Signer<'info> {
    info: &'info AccountInfo<'info>,
}
```

**レイ**「`Account<'info, T>`は、`AccountInfo`への参照を持ってる。この参照は、トランザクション処理中だけ有効」

**ユイ**「だから`'info`で、その有効期間を明示してるんですね」

**レイ**「そう。これによって、コンパイラが『トランザクション外でアカウントを使おうとするミス』を防いでくれるんだ」

**ユイ**「Chapter 02で学んだライフタイムが、こんなところで役立つなんて...」

**レイ**「Rustの基礎がしっかりしてると、Anchorのコードも理解しやすいでしょ」

---

## 実践：Counterプログラムを作る

**レイ**「じゃあ、実用的な例を作ってみよう。カウンターアプリ」

**ユイ**「カウンター...？」

**レイ**「クリックすると数字が増えるやつ。シンプルだけど、Anchorの基本が全部詰まってる」

```rust
use anchor_lang::prelude::*;

declare_id!("...");

#[program]
pub mod counter {
    use super::*;

    pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count = 0;
        counter.authority = ctx.accounts.authority.key();
        Ok(())
    }

    pub fn increment(ctx: Context<Increment>) -> Result<()> {
        let counter = &mut ctx.accounts.counter;
        counter.count += 1;
        Ok(())
    }
}

#[account]
pub struct Counter {
    pub count: u64,
    pub authority: Pubkey,
}

#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,
        payer = authority,
        space = 8 + 8 + 32
    )]
    pub counter: Account<'info, Counter>,

    #[account(mut)]
    pub authority: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        has_one = authority
    )]
    pub counter: Account<'info, Counter>,

    pub authority: Signer<'info>,
}
```

**ユイ**「うわ、急に長くなりましたね...」

**レイ**「大丈夫、順番に見ていこう」

---

## `#[account]`：データ構造の定義

**レイ**「まずは、カウンターのデータ構造から」

```rust
#[account]
pub struct Counter {
    pub count: u64,      // 8バイト
    pub authority: Pubkey, // 32バイト
}
```

**ユイ**「普通の構造体に、`#[account]`って属性がついてるだけですね」

**レイ**「この`#[account]`マクロが、めちゃくちゃ仕事してくれるんだ」

```
┌─────────────────────────────────────────────────────────────────┐
│  #[account] マクロの展開結果                                     │
├─────────────────────────────────────────────────────────────────┤
│  1. Borshシリアライズ/デシリアライズの実装                       │
│     → アカウントデータ(Vec<u8>)と構造体の相互変換                 │
│                                                                 │
│  2. 所有者チェックの実装                                         │
│     → このアカウントの所有者がこのProgramであることを検証          │
│                                                                 │
│  3. Discriminator（識別子）の付与                                │
│     → 8バイトのユニークなIDでアカウント種別を識別                  │
│                                                                 │
│  4. AccountSerialize / AccountDeserialize トレイトの実装         │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「シリアライズって、データをバイト列に変換するやつですよね」

**レイ**「そう。ブロックチェーンのアカウントには、バイト列（`Vec<u8>`）しか保存できない。だから構造体↔バイト列の変換が必要なんだけど、それを全部自動でやってくれる」

**ユイ**「discriminatorって何ですか？」

**レイ**「いい質問。これは超重要」

```
┌─────────────────────────────────────────────────────────────────┐
│  Discriminator の仕組み                                          │
├─────────────────────────────────────────────────────────────────┤
│  アカウントのdata領域:                                           │
│  ┌──────────────────────────────────────────────────┐           │
│  │ 8 bytes    │ 8 bytes │ 32 bytes                 │           │
│  │discriminator│ count   │ authority                │           │
│  │ (識別子)    │ (u64)   │ (Pubkey)                 │           │
│  └──────────────────────────────────────────────────┘           │
│                                                                 │
│  discriminator = sha256("account:Counter")[0..8]                │
│  → "Counter"構造体専用の8バイトID                                │
│                                                                 │
│  なぜ必要？                                                      │
│  - 同じProgramが複数のアカウント型を持つ場合                      │
│  - アカウントのdata(バイト列)だけでは区別できない                  │
│  - discriminatorで「このデータはCounter型だ」と識別                │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「あー、型の識別子なんですね。バイト列だけだと、何の型か分からないから」

**レイ**「正解。Anchorは保存時に先頭8バイトにdiscriminatorを自動で付与して、読み込み時にチェックしてくれる」

---

## `space`の計算：なぜ8 + 8 + 32？

**ユイ**「次に気になったんですけど...」

```rust
#[account(
    init,
    payer = authority,
    space = 8 + 8 + 32  // ← これ
)]
pub counter: Account<'info, Counter>,
```

**ユイ**「この`space`って何ですか？　なんで8 + 8 + 32なんですか？」

**レイ**「これは、アカウントに必要なバイト数を指定してる」

```
space計算の内訳:
┌─────────────────────────────────────────────────────────────────┐
│  8   = discriminator (Anchorが自動付与)                          │
│  8   = count: u64 のサイズ                                       │
│  32  = authority: Pubkey のサイズ                                │
│ ───────────────────────────────────                             │
│  48  = 合計バイト数                                              │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「あ、構造体のサイズを計算してるんですね」

**レイ**「そう。Solanaでは、アカウントを作る時に、必要なスペースを事前に確保する必要があるんだ。スペースに応じて『レント』っていう保管料金も計算される」

**ユイ**「保管料金...？」

**レイ**「Solanaは、ストレージを使う対価として少額のSOLを預ける必要がある。でも一定額以上預けておけば、レントが免除されるから、実質的には無料みたいなもの」

**レイ**「よく使う型のサイズは、これを覚えておくといいよ」

```
| 型         | サイズ (bytes) |
|-----------|---------------|
| bool      | 1             |
| u8 / i8   | 1             |
| u16 / i16 | 2             |
| u32 / i32 | 4             |
| u64 / i64 | 8             |
| u128/i128 | 16            |
| Pubkey    | 32            |
| String    | 4 + len       |  ← 長さプレフィックス + 文字数
| Vec<T>    | 4 + n * sizeof(T) |
```

**ユイ**「メモメモ...」

---

## `#[derive(Accounts)]`と制約

**レイ**「次は、必要なアカウントを定義する構造体」

```rust
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,                    // 新規作成
        payer = authority,       // 作成費用の支払い者
        space = 8 + 8 + 32       // 必要なバイト数
    )]
    pub counter: Account<'info, Counter>,

    #[account(mut)]              // このアカウントは変更される
    pub authority: Signer<'info>, // 署名が必要

    pub system_program: Program<'info, System>, // システムプログラム
}
```

**ユイ**「`#[account(...)]`の中に、いろんな制約が書かれてますね」

**レイ**「そう。この制約によって、Anchorが自動で検証コードを生成してくれる」

```
| 制約 | 意味 | 自動生成される検証 |
|-----|------|------------------|
| init | アカウント新規作成 | 空であること、レント料金の計算 |
| mut | 変更可能 | トランザクション後に変更を保存 |
| payer = X | 作成費用支払者 | Xから必要SOLを引き落とし |
| has_one = X | フィールド検証 | account.X == X.key() |
```

**ユイ**「`payer = authority`って、authorityがお金払うってことですか？」

**レイ**「そう。アカウントを作るには、レント用のSOLが必要。それを`authority`（つまりトランザクションに署名した人）から引き落とすってこと」

**ユイ**「なるほど...じゃあ、`Signer<'info>`って？」

**レイ**「これは、『このトランザクションに署名した人』を表す型。つまり、このアカウントのプライベートキーを持ってる人ってこと」

**ユイ**「あ、認証みたいなものですね」

**レイ**「正解。Anchorのアカウント型には、いくつか種類があるんだ」

```rust
// 1. Account<'info, T> — データを持つアカウント
pub counter: Account<'info, Counter>,

// 2. Signer<'info> — 署名者（トランザクションに署名した人）
pub authority: Signer<'info>,

// 3. Program<'info, T> — 他のProgram
pub system_program: Program<'info, System>,

// 4. SystemAccount<'info> — 通常のウォレット（データなし）
pub user_wallet: SystemAccount<'info>,
```

**ユイ**「`system_program`って何ですか？」

**レイ**「Solanaのシステムプログラム。アカウントの作成とか、SOLの転送とか、基本的な操作を担当してる。`init`でアカウントを作る時は、このシステムプログラムを呼び出す必要があるから、必須なんだ」

---

## 命令の実装を読み解く

**レイ**「じゃあ、命令の中身を見てみよう」

```rust
pub fn initialize(ctx: Context<Initialize>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    counter.count = 0;
    counter.authority = ctx.accounts.authority.key();
    Ok(())
}
```

**ユイ**「`ctx.accounts.counter`で、さっき定義した`counter`アカウントにアクセスしてるんですね」

**レイ**「そう。`&mut`で可変借用してる。なぜなら、カウンターの値を変更するから」

**ユイ**「Chapter 02で学んだ可変借用だ！」

**レイ**「`.key()`は、アカウントの公開鍵（Pubkey）を取得するメソッド。これで『誰がこのカウンターのオーナーか』を記録してる」

**ユイ**「最後の`Ok(())`は？」

**レイ**「成功を返してる。Anchorは、この関数が成功したら、変更されたアカウントデータを自動的にシリアライズして、ブロックチェーンに保存してくれるんだ」

**ユイ**「え、自動で保存してくれるんですか？」

**レイ**「そう。素のSolanaだと、手動でシリアライズして書き込む必要があるけど、Anchorはそれも全部やってくれる」

---

## `has_one`制約の秘密

**レイ**「次は`increment`命令」

```rust
pub fn increment(ctx: Context<Increment>) -> Result<()> {
    let counter = &mut ctx.accounts.counter;
    counter.count += 1;
    Ok(())
}
```

**ユイ**「シンプルですね。カウンターを1増やすだけ」

**レイ**「でも、重要なのはアカウント構造体の方」

```rust
#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        has_one = authority  // ← これ！
    )]
    pub counter: Account<'info, Counter>,

    pub authority: Signer<'info>,
}
```

**ユイ**「`has_one = authority`...？」

**レイ**「これは、セキュリティチェック。こう動く」

```
has_one = authority が行う検証:

1. counter.authority (Counter構造体のauthorityフィールド)
2. authority.key() (Increment構造体のauthorityアカウントのキー)
3. これらが一致することを検証

つまり:
「このトランザクションに署名した人が、
 カウンターの所有者(authority)と同一人物か？」
を自動チェック
```

**ユイ**「あ！　他人が勝手にカウンターをインクリメントできないようにしてるんですね」

**レイ**「正解。これがないと、誰でも誰のカウンターでも変更できちゃう。`has_one`制約一つで、このチェックが自動化される」

**ユイ**「Anchorすごい...手動だと、このチェック書き忘れそう」

**レイ**「そうなんだよ。実際、素のSolanaで開発してた初期の頃は、このチェック漏れで脆弱性があるコントラクトが結構あった」

---

## PDA：Programが所有できる特殊なアカウント

**ユイ**「レイ先輩、一つ気になったことがあって...」

**レイ**「何？」

**ユイ**「今のカウンター、ユーザーごとに別々のカウンターを作ることってできますか？」

**レイ**「お、いい質問。それにはPDAっていう仕組みを使う」

**ユイ**「PDA...？」

**レイ**「Program Derived Address。Programが『所有』できる特殊なアドレスなんだ」

```
┌─────────────────────────────────────────────────────────────────┐
│  通常のアカウント vs PDA                                         │
├─────────────────────────────────────────────────────────────────┤
│  通常のアカウント:                                               │
│  - 秘密鍵から生成されたアドレス                                   │
│  - 署名には秘密鍵が必要                                          │
│  - 問題: Programは秘密鍵を持てない！                              │
│                                                                 │
│  PDA (Program Derived Address):                                 │
│  - Programのアドレス + シードから計算で生成                       │
│  - 対応する秘密鍵が「存在しない」ように設計                        │
│  - Programが直接署名できる（CPIで使用）                           │
│                                                                 │
│  用途:                                                           │
│  - ユーザーごとのデータ保存（シード = ユーザーPubkey）            │
│  - Programが管理するトークン口座                                  │
│  - 決定的なアドレス生成（同じシード → 同じアドレス）              │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「秘密鍵が存在しない...？　どういうことですか？」

**レイ**「通常のアカウントは、秘密鍵があって、それに対応する公開鍵がアドレスになる。でも、PDAは計算だけで生成されたアドレスで、対応する秘密鍵が存在しないように設計されてるんだ」

**ユイ**「秘密鍵がないのに、どうやって使うんですか？」

**レイ**「Programが『代理で署名』できる。だからProgramが管理するアカウントとして使えるんだ」

**レイ**「実際のコードを見てみよう」

```rust
#[derive(Accounts)]
pub struct CreateUserProfile<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + UserProfile::INIT_SPACE,
        seeds = [b"user_profile", user.key().as_ref()],
        bump
    )]
    pub user_profile: Account<'info, UserProfile>,

    #[account(mut)]
    pub user: Signer<'info>,

    pub system_program: Program<'info, System>,
}

#[account]
#[derive(InitSpace)]
pub struct UserProfile {
    pub authority: Pubkey,    // 32
    #[max_len(50)]
    pub name: String,         // 4 + 50
    pub created_at: i64,      // 8
}
```

**ユイ**「`seeds`と`bump`...？」

**レイ**「`seeds`は、アドレスを生成するための材料」

```
seeds = [b"user_profile", user.key().as_ref()]
         ↓               ↓
         固定文字列      ユーザーの公開鍵

アドレス計算:
PDA = hash(seeds + program_id + bump)

例:
- ユーザーA: seeds = ["user_profile", A_pubkey] → PDA_A
- ユーザーB: seeds = ["user_profile", B_pubkey] → PDA_B

同じシードからは常に同じPDAが生成される（決定的）
```

**ユイ**「あ！　ユーザーごとに違うシードだから、ユーザーごとに別のアドレスが生成されるんですね」

**レイ**「そういうこと。しかも決定的だから、後から『ユーザーAのプロフィールアカウントはどこ？』って計算で求められる」

**ユイ**「じゃあ`bump`は？」

**レイ**「これは、ちょっと技術的な話になるけど...」

```
┌─────────────────────────────────────────────────────────────────┐
│  bump（バンプシード）とは                                        │
├─────────────────────────────────────────────────────────────────┤
│  PDAは「楕円曲線上にない点」でなければならない                    │
│  （秘密鍵が存在しないことを保証するため）                         │
│                                                                 │
│  計算方法:                                                       │
│  1. bump = 255 から開始                                          │
│  2. hash(seeds + program_id + bump) を計算                       │
│  3. 結果が曲線上なら bump -= 1 して再計算                        │
│  4. 曲線上にない点が見つかったらそれがPDA                         │
│                                                                 │
│  Anchorの`bump`制約:                                             │
│  - 初回: bumpを自動計算してアカウントに保存                       │
│  - 2回目以降: 保存されたbumpを使用（計算コスト削減）              │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「うーん、楕円曲線...難しいですね」

**レイ**「詳細は理解しなくていい。要は『秘密鍵が絶対に存在しないアドレスを作るための調整値』って覚えておけばOK。Anchorが全部自動でやってくれるから」

---

## エラーハンドリング：安全なコードを書く

**レイ**「さて、カウンターができたけど、今のままだと問題がある」

**ユイ**「問題...？」

**レイ**「カウンターを増やし続けると、いつかオーバーフローする」

**ユイ**「あ！　u64の最大値を超えちゃう」

**レイ**「そう。だからエラーハンドリングが必要」

```rust
use anchor_lang::prelude::*;

#[error_code]
pub enum ErrorCode {
    #[msg("Counter overflow")]
    Overflow,

    #[msg("Unauthorized access")]
    Unauthorized,

    #[msg("Invalid amount: must be greater than 0")]
    InvalidAmount,
}

#[program]
pub mod my_program {
    use super::*;

    pub fn increment(ctx: Context<Increment>, amount: u64) -> Result<()> {
        require!(amount > 0, ErrorCode::InvalidAmount);

        let counter = &mut ctx.accounts.counter;
        counter.count = counter.count
            .checked_add(amount)
            .ok_or(ErrorCode::Overflow)?;

        Ok(())
    }
}
```

**ユイ**「`#[error_code]`でエラーを定義するんですね」

**レイ**「そう。`require!`マクロで入力値をチェックして、`checked_add`でオーバーフローチェック」

**ユイ**「Chapter 01で学んだ`checked_add`だ！」

**レイ**「Rustの安全性の考え方が、そのままスマートコントラクトの安全性につながるんだよ」

**レイ**「エラー処理のパターンは主に3つ」

```rust
// パターン1: require!マクロ
require!(amount > 0, ErrorCode::InvalidAmount);
// → amountが0以下ならエラーを返して終了

// パターン2: checked演算 + ok_or
counter.count
    .checked_add(amount)    // オーバーフローならNone
    .ok_or(ErrorCode::Overflow)?;  // NoneならエラーReturn

// パターン3: if文 + return Err
if amount == 0 {
    return Err(ErrorCode::InvalidAmount.into());
}
```

**ユイ**「パターン2、Chapter 04で学んだ`Option`と`Result`の変換ですね」

**レイ**「よく覚えてる。今まで学んだこと全部活きてるでしょ」

---

## CPI：他のProgramを呼び出す

**レイ**「最後にもう一つ重要な概念。CPI、Cross-Program Invocationっていうのがある」

**ユイ**「クロス...プログラム？」

**レイ**「自分のProgramから、他のProgramを呼び出す機能。例えばトークンを転送する時とか」

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    // CPI用のアカウント構造体を作成
    let cpi_accounts = Transfer {
        from: ctx.accounts.from.to_account_info(),
        to: ctx.accounts.to.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    };

    // 呼び出すProgramを指定
    let cpi_program = ctx.accounts.token_program.to_account_info();

    // CPIコンテキストを作成
    let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);

    // Token Programのtransfer命令を呼び出し
    token::transfer(cpi_ctx, amount)?;

    Ok(())
}

#[derive(Accounts)]
pub struct TransferTokens<'info> {
    #[account(mut)]
    pub from: Account<'info, TokenAccount>,
    #[account(mut)]
    pub to: Account<'info, TokenAccount>,
    pub authority: Signer<'info>,
    pub token_program: Program<'info, Token>,
}
```

**ユイ**「自分のProgramから、Solanaのトークンプログラムを呼んでるんですね」

**レイ**「そう。こうすることで、既存のProgramの機能を再利用できる」

```
┌─────────────────────────────────────────────────────────────────┐
│  CPI（Cross-Program Invocation）の仕組み                         │
├─────────────────────────────────────────────────────────────────┤
│  クライアント                                                    │
│      │                                                          │
│      ▼                                                          │
│  ┌────────────────┐                                             │
│  │ あなたのProgram │                                             │
│  │  (counter)     │                                             │
│  └───────┬────────┘                                             │
│          │ CPI: token::transfer(...)                            │
│          ▼                                                      │
│  ┌────────────────┐                                             │
│  │ Token Program  │  ← Solana標準のトークン管理Program            │
│  │  (SPL Token)   │                                             │
│  └────────────────┘                                             │
└─────────────────────────────────────────────────────────────────┘
```

**ユイ**「これって、Chapter 09のAPIを呼び出すみたいな感じですね」

**レイ**「いい例え！　Web APIを呼ぶのと似てる。DeFiのプロトコルは、このCPIを駆使して複雑な処理を実現してるんだ」

---

## TypeScriptクライアント：フロントエンドから呼び出す

**ユイ**「プログラムはできましたけど、これってどうやってフロントエンドから呼ぶんですか？」

**レイ**「Anchorが自動でTypeScriptクライアントを生成してくれるんだ」

```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

describe("my_program", () => {
    // プロバイダー（接続+ウォレット）を取得
    const provider = anchor.AnchorProvider.env();
    anchor.setProvider(provider);

    // Programインスタンスを取得（IDLから自動生成された型付き）
    const program = anchor.workspace.MyProgram as Program<MyProgram>;

    it("Initializes the counter", async () => {
        // 新しいアカウントのキーペアを生成
        const counter = anchor.web3.Keypair.generate();

        // トランザクションを送信
        await program.methods
            .initialize()          // 命令名（Rustの関数名がキャメルケースに）
            .accounts({            // 必要なアカウント
                counter: counter.publicKey,
                authority: provider.wallet.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
            })
            .signers([counter])    // 署名者（新規アカウントは署名が必要）
            .rpc();                // トランザクション送信

        // アカウントデータを取得
        const account = await program.account.counter.fetch(counter.publicKey);
        console.log("Count:", account.count.toString());
    });
});
```

**ユイ**「TypeScriptから、Rustの関数を型安全に呼べるんですね」

**レイ**「そう。`anchor build`すると、IDLっていうJSON形式の定義ファイルが生成される」

**レイ**「IDLはInterface Definition Languageの略で、Programの『仕様書』みたいなもの」

```json
{
  "version": "0.1.0",
  "name": "my_program",
  "instructions": [
    {
      "name": "initialize",
      "accounts": [
        { "name": "counter", "isMut": true, "isSigner": true },
        { "name": "authority", "isMut": true, "isSigner": true },
        { "name": "systemProgram", "isMut": false, "isSigner": false }
      ],
      "args": []
    }
  ],
  "accounts": [
    {
      "name": "Counter",
      "type": {
        "kind": "struct",
        "fields": [
          { "name": "count", "type": "u64" },
          { "name": "authority", "type": "publicKey" }
        ]
      }
    }
  ]
}
```

**ユイ**「これがあるから、TypeScriptの型定義も自動生成できるんですね」

**レイ**「正解。だから、Rustでコード書くだけで、フロントエンドの型も自動で同期される。めちゃくちゃ開発しやすいでしょ」

---

## セキュリティ：絶対に気をつけること

**レイ**「さて、実装の話はだいたい終わったけど、最後に一番重要な話」

**ユイ**「何ですか？」

**レイ**「セキュリティ。スマートコントラクトは一度デプロイすると、基本的に変更できない。しかもお金を扱うから、脆弱性があると実際に資金が盗まれる」

**ユイ**「怖い...」

**レイ**「でも、Anchorと基本的なベストプラクティスを守れば、かなり安全なコードが書ける」

```
| リスク | 対策 | Anchorの支援 |
|-------|------|-------------|
| 署名者検証漏れ | Signer<'info>を使用 | 自動検証 |
| 所有者検証漏れ | Account<'info, T>が自動検証 | discriminator + owner |
| オーバーフロー | checked_add, checked_sub | 手動で実装 |
| 再入攻撃 | - | Solanaは構造的に防止 |
| PDA署名偽装 | seeds/bumpを検証 | 自動検証 |
| 不正なアカウント型 | discriminator検証 | 自動検証 |
```

**ユイ**「Anchorが自動でやってくれるのと、自分で気をつけないといけないのがあるんですね」

**レイ**「そう。特にオーバーフローは自分で`checked_add`とか使う必要がある」

**レイ**「安全なコードの例を見てみよう」

```rust
pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    let vault = &mut ctx.accounts.vault;

    // 1. 入力値の検証
    require!(amount > 0, ErrorCode::InvalidAmount);

    // 2. 残高チェック（オーバーフロー防止）
    require!(vault.balance >= amount, ErrorCode::InsufficientFunds);

    // 3. checked演算で安全に減算
    vault.balance = vault.balance
        .checked_sub(amount)
        .ok_or(ErrorCode::Overflow)?;

    // 4. 転送処理...

    Ok(())
}
```

**ユイ**「まず入力値をチェックして、残高をチェックして、checked演算で安全に計算...」

**レイ**「そう。この順番も大事。先に残高を減らしてから転送すると、転送失敗時に不整合が起きる可能性がある」

**ユイ**「セキュリティって、いろいろ考えることがあるんですね」

**レイ**「だからこそ、Anchorのような安全なフレームワークが重要なんだ」

---

## 実際にデプロイしてみる

**レイ**「じゃあ、実際にテストネットにデプロイしてみようか」

**ユイ**「え、本当にブロックチェーンに？」

**レイ**「テストネット（Devnet）だから、本物のお金は使わない。でも、本番環境（Mainnet）と同じ体験ができるよ」

```bash
# Devnetに切り替え
solana config set --url devnet

# エアドロップでテスト用のSOLを取得
solana airdrop 2

# デプロイ
anchor deploy --provider.cluster devnet
```

**ユイ**「エアドロップ...タダでもらえるんですか？」

**レイ**「テストネットだからね。本番のMainnetだと、もちろん買わないといけない」

**ユイ**「デプロイできました！」

**レイ**「おめでとう。これで君のコードが、Solanaブロックチェーン上で動いてる」

**ユイ**「なんか感動...」

**レイ**「これからフロントエンド作って、実際にブラウザから呼び出せるようにしてみるといいよ」

---

## まとめ：EthereumとSolanaの比較

**レイ**「最後に、EthereumとSolanaの違いを整理しておこう」

```
| 概念 | Ethereum/Solidity | Solana/Anchor |
|-----|------------------|---------------|
| コントラクト | Contract | Program |
| ストレージ | Contract内 | 別アカウント |
| 言語 | Solidity | Rust |
| フレームワーク | Hardhat/Foundry | Anchor |
| アドレス生成 | CREATE/CREATE2 | PDA |
| 外部呼び出し | call/delegatecall | CPI |
| ガス/手数料 | 高い | 非常に低い |
| スループット | ~15 TPS | ~65,000 TPS |
```

**ユイ**「Solana、めちゃくちゃ速いし安いんですね」

**レイ**「そう。特にゲームとかNFTミントとか、大量のトランザクションが必要な用途に向いてる」

**ユイ**「じゃあEthereumは？」

**レイ**「Ethereumは、エコシステムが一番大きい。DeFiプロトコルも、開発者も圧倒的に多い。どっちがいいってわけじゃなくて、用途次第」

---

## エピローグ：次のステップ

**ユイ**「レイ先輩、今日一日でめちゃくちゃ学びました」

**レイ**「お疲れさま。どうだった？」

**ユイ**「最初はWeb3って難しそうって思ってたけど、Rustの知識があれば意外といけるんだなって」

**レイ**「そうそう。Rustの所有権→アカウント所有者検証、ライフタイム→`<'info>`、トレイト→Anchorの型制約、エラーハンドリング→`Result`型...全部つながってるでしょ」

**ユイ**「Chapter 01から09まで学んだこと、全部活きてますね」

**レイ**「これからどうする？」

**ユイ**「まずは、Solana Playgroundでいろいろ試してみたいです。それから、シンプルなNFTミントのプロジェクトとか作ってみたいな」

**レイ**「いいね。あとは、既存のプロトコルのコードを読むのもオススメ。JupiterとかMarinadeとか、OSSになってるから」

**ユイ**「あ、それ面白そう！」

**レイ**「Anchor Discordに参加するのもいい。コミュニティが活発だから、質問すればすぐ答えが返ってくるよ」

**ユイ**「ありがとうございます！　あの、レイ先輩」

**レイ**「ん？」

**ユイ**「Chapter 00から始めて、ここまで来れて...なんか、自分でもびっくりしてます」

**レイ**「ユイちゃん、すごく成長したよ。最初は所有権でつまずいてたのに、今じゃアカウント所有者検証とか普通に理解してるもんね」

**ユイ**「Rustって、最初は難しいけど、分かってくるとめちゃくちゃ楽しいですね」

**レイ**「そうそう。そしてその厳格さが、スマートコントラクトのセキュリティに直結する。Rustを学んだことは、Web3開発でも絶対に活きるよ」

**ユイ**「次は、自分のDApp作って、レイ先輩に見せますね！」

**レイ**「楽しみにしてる。Happy Building on Solana!」

---

## 学習リソース

この章で学んだことをさらに深めるためのリソース：

| リソース | 説明 |
|---------|------|
| [Anchor Book](https://book.anchor-lang.com/) | 公式ドキュメント、詳細な解説 |
| [Solana Cookbook](https://solanacookbook.com/) | レシピ集、よくあるパターン |
| [Solana Playground](https://beta.solpg.io/) | ブラウザIDE、環境構築不要 |
| [Buildspace](https://buildspace.so/) | ハンズオンコース |
| [Solana Stack Exchange](https://solana.stackexchange.com/) | Q&A、困った時に |

---

## この章のポイント

- **Solana = 高速・低コスト・Rustネイティブ** のブロックチェーン
- **Program（コード）とAccount（データ）が分離** → 並列実行可能
- **Anchor = セキュリティチェックを自動生成** するフレームワーク
- **`<'info>` = トランザクション処理中のライフタイム**
- **`#[account]` = discriminator + シリアライズ + 所有者検証** を自動実装
- **`#[derive(Accounts)]` = アカウント検証コード** を自動生成
- **PDA = Programが所有できる、シードから決定的に生成されるアドレス**
- **CPI = 他のProgramを呼び出す** 機能（DeFiで重要）
- **TypeScript/Rust の型安全な連携** が自動で実現
- **Rustの基礎知識（所有権、ライフタイム、エラーハンドリング）が全部活きる**

---

## Web開発者のための補足

### 「状態がコントラクト内にない」という概念

```typescript
// 従来のWeb開発の感覚
class UserService {
    private users: Map<string, User> = new Map();  // サービス内に状態

    getUser(id: string) { return this.users.get(id); }
}

// Solanaの感覚
// Program = ロジックのみ（ステートレス）
// 状態 = 別のアカウント（外部のデータベースのようなもの）
```

### 「すべてのデータアクセスを事前に宣言」

```typescript
// Web開発: 関数内で自由にDB読み書き
async function updateUser(userId: string) {
    const user = await db.users.findById(userId);  // 動的にアクセス
    user.name = "New Name";
    await user.save();
}

// Solana: 使うアカウントを事前に宣言
#[derive(Accounts)]
pub struct UpdateUser<'info> {
    #[account(mut)]
    pub user: Account<'info, User>,  // ← 事前に宣言必須
}
```

### 「トランザクション≠関数呼び出し」

```
Web開発:
クライアント → サーバー → DB → レスポンス
(同期的に結果が返る)

Solana:
クライアント → トランザクション送信 → ブロック確定を待つ
(非同期、確定まで数秒かかる場合も)
```

---

**Happy Building on Solana! 🦀⚡**
