# Chapter 10: Solana/Anchor

> **この章で学ぶこと**: Solanaの概要、Anchorフレームワーク、Program開発の基礎 — RustでのWeb3開発

---

## なぜRust × Solana？

| ブロックチェーン | 主要言語 | 特徴 |
|----------------|---------|------|
| Ethereum | Solidity | 最大のエコシステム、EVM |
| Solana | **Rust** | 高速（65,000+ TPS）、低手数料 |
| Polkadot | **Rust** | クロスチェーン、Substrate |
| Near | **Rust** | シャーディング |
| Cosmos | Go, Rust | IBC |

**Solanaを選ぶ理由**:
1. **高速・低コスト**: 秒間65,000トランザクション、手数料 ~$0.00025
2. **Rustネイティブ**: Rustの安全性がスマートコントラクトのセキュリティに貢献
3. **成長するエコシステム**: DeFi、NFT、ゲームで活発

---

## Solanaの基本概念

### アカウントモデル

Solanaは**アカウントモデル**を採用しています（Ethereumも同様）。

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

### Program（スマートコントラクト）

Solanaでは「スマートコントラクト」を**Program**と呼びます。

- **ステートレス**: Program自体はデータを持たない
- **アカウントにデータを保存**: データは別のアカウントに格納
- **並列実行可能**: アクセスするアカウントが異なれば並列実行

🔄 **比較（Ethereum）**:
```
Ethereum: Contract = コード + ストレージ（一体化）
Solana:   Program = コードのみ、データは別アカウント
```

---

## 開発環境のセットアップ

### 必要なツール

```bash
# Rust（既にインストール済みと仮定）
rustup update

# Solana CLI
sh -c "$(curl -sSfL https://release.solana.com/stable/install)"

# Anchor CLI
cargo install --git https://github.com/coral-xyz/anchor anchor-cli --locked

# 確認
solana --version
anchor --version
```

### ローカル環境の設定

```bash
# ローカルネットワークに設定
solana config set --url localhost

# 新しいキーペアを作成
solana-keygen new

# ローカルバリデータを起動（別ターミナル）
solana-test-validator
```

---

## Anchorとは

**Anchor**はSolana Program開発のフレームワークです。

| 機能 | 素のSolana | Anchor |
|-----|-----------|--------|
| アカウント検証 | 手動で全チェック | マクロで自動生成 |
| シリアライズ | Borsh手動実装 | 自動 |
| エラーハンドリング | 生のエラーコード | 構造化エラー |
| テスト | 複雑 | TypeScript統合 |
| クライアント | 手動生成 | IDL自動生成 |

Anchorを使うと、開発生産性が大幅に向上します。

---

## 最初のProgram

### プロジェクト作成

```bash
anchor init my_program
cd my_program
```

### プロジェクト構造

```
my_program/
├── Anchor.toml         # プロジェクト設定
├── Cargo.toml
├── programs/
│   └── my_program/
│       ├── Cargo.toml
│       └── src/
│           └── lib.rs  # Programのコード
├── tests/
│   └── my_program.ts   # TypeScriptテスト
└── migrations/
    └── deploy.ts
```

### Hello Worldプログラム

**programs/my_program/src/lib.rs**:
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

### ビルドとデプロイ

```bash
# ビルド
anchor build

# ローカルにデプロイ
anchor deploy

# テスト実行
anchor test
```

---

## アカウントの定義

Solanaではデータはアカウントに保存します。

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

// アカウントの構造体
#[account]
pub struct Counter {
    pub count: u64,
    pub authority: Pubkey,
}

// initialize命令に必要なアカウント
#[derive(Accounts)]
pub struct Initialize<'info> {
    #[account(
        init,                           // 新規作成
        payer = authority,              // 支払い者
        space = 8 + 8 + 32              // discriminator + count + authority
    )]
    pub counter: Account<'info, Counter>,
    
    #[account(mut)]
    pub authority: Signer<'info>,
    
    pub system_program: Program<'info, System>,
}

// increment命令に必要なアカウント
#[derive(Accounts)]
pub struct Increment<'info> {
    #[account(
        mut,
        has_one = authority             // authorityの検証
    )]
    pub counter: Account<'info, Counter>,
    
    pub authority: Signer<'info>,
}
```

### アカウント制約

| 制約 | 説明 |
|-----|------|
| `init` | アカウントを新規作成 |
| `mut` | アカウントを変更可能に |
| `has_one = field` | フィールドの値とアカウントを検証 |
| `seeds`, `bump` | PDA（後述） |
| `constraint = expr` | カスタム制約 |

🔄 **比較（Solidity）**:
```solidity
// Solidityではストレージ変数として直接定義
contract Counter {
    uint256 public count;
    address public authority;
    
    function increment() public {
        require(msg.sender == authority);
        count += 1;
    }
}
```

---

## PDA（Program Derived Address）

**PDA**はProgramが「所有」できる特殊なアドレスです。秘密鍵がないため、Programのみが署名できます。

```rust
#[derive(Accounts)]
pub struct CreateUserProfile<'info> {
    #[account(
        init,
        payer = user,
        space = 8 + UserProfile::INIT_SPACE,
        seeds = [b"user_profile", user.key().as_ref()],  // シード
        bump  // 自動計算
    )]
    pub user_profile: Account<'info, UserProfile>,
    
    #[account(mut)]
    pub user: Signer<'info>,
    
    pub system_program: Program<'info, System>,
}

#[account]
#[derive(InitSpace)]
pub struct UserProfile {
    pub authority: Pubkey,
    #[max_len(50)]
    pub name: String,
    pub created_at: i64,
}
```

PDAの特徴：
- **決定的**: 同じシードからは常に同じアドレス
- **署名不要**: Programが直接操作可能
- **ユニーク**: ユーザーごとに1つのプロファイル等に使える

---

## エラーハンドリング

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

---

## CPI（Cross-Program Invocation）

あるProgramから別のProgramを呼び出す：

```rust
use anchor_lang::prelude::*;
use anchor_spl::token::{self, Token, TokenAccount, Transfer};

pub fn transfer_tokens(ctx: Context<TransferTokens>, amount: u64) -> Result<()> {
    let cpi_accounts = Transfer {
        from: ctx.accounts.from.to_account_info(),
        to: ctx.accounts.to.to_account_info(),
        authority: ctx.accounts.authority.to_account_info(),
    };
    
    let cpi_program = ctx.accounts.token_program.to_account_info();
    let cpi_ctx = CpiContext::new(cpi_program, cpi_accounts);
    
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

---

## TypeScriptクライアント

Anchorは自動でTypeScriptクライアントを生成します。

**tests/my_program.ts**:
```typescript
import * as anchor from "@coral-xyz/anchor";
import { Program } from "@coral-xyz/anchor";
import { MyProgram } from "../target/types/my_program";

describe("my_program", () => {
    const provider = anchor.AnchorProvider.env();
    anchor.setProvider(provider);

    const program = anchor.workspace.MyProgram as Program<MyProgram>;

    it("Initializes the counter", async () => {
        const counter = anchor.web3.Keypair.generate();
        
        await program.methods
            .initialize()
            .accounts({
                counter: counter.publicKey,
                authority: provider.wallet.publicKey,
                systemProgram: anchor.web3.SystemProgram.programId,
            })
            .signers([counter])
            .rpc();

        const account = await program.account.counter.fetch(counter.publicKey);
        console.log("Count:", account.count.toString());
    });

    it("Increments the counter", async () => {
        // ...
    });
});
```

🔄 **比較（Ethers.js）**:
```typescript
// Ethereumの場合
const contract = new ethers.Contract(address, abi, signer);
await contract.increment();
```

---

## セキュリティのベストプラクティス

| リスク | 対策 |
|-------|------|
| 署名者検証漏れ | `Signer<'info>`を使用 |
| 所有者検証漏れ | `Account<'info, T>`が自動検証 |
| オーバーフロー | `checked_add`, `checked_sub` |
| 再入攻撃 | Solanaは構造的に防止 |
| PDA署名偽装 | Anchorが自動検証 |

```rust
// 安全なコード例
pub fn withdraw(ctx: Context<Withdraw>, amount: u64) -> Result<()> {
    let vault = &mut ctx.accounts.vault;
    
    // オーバーフローチェック
    require!(vault.balance >= amount, ErrorCode::InsufficientFunds);
    
    vault.balance = vault.balance
        .checked_sub(amount)
        .ok_or(ErrorCode::Overflow)?;
    
    // 転送処理...
    
    Ok(())
}
```

---

## 開発フロー

```
1. anchor init project_name     # プロジェクト作成
2. lib.rsを編集                 # Programを実装
3. anchor build                 # ビルド
4. anchor test                  # テスト（ローカル）
5. anchor deploy --provider.cluster devnet  # Devnetにデプロイ
6. フロントエンド開発           # React + @coral-xyz/anchor
```

### Devnetでのテスト

```bash
# Devnetに切り替え
solana config set --url devnet

# エアドロップでSOLを取得
solana airdrop 2

# デプロイ
anchor deploy --provider.cluster devnet
```

---

## 次のステップ

Solana/Anchor開発を深めるためのリソース：

| リソース | 説明 |
|---------|------|
| [Anchor Book](https://book.anchor-lang.com/) | 公式ドキュメント |
| [Solana Cookbook](https://solanacookbook.com/) | レシピ集 |
| [Solana Playground](https://beta.solpg.io/) | ブラウザIDE |
| [Buildspace](https://buildspace.so/) | ハンズオンコース |
| [Solana Stack Exchange](https://solana.stackexchange.com/) | Q&A |

---

## まとめ

| 概念 | Ethereum/Solidity | Solana/Anchor |
|-----|------------------|---------------|
| コントラクト | Contract | Program |
| ストレージ | Contract内 | 別アカウント |
| 言語 | Solidity | **Rust** |
| フレームワーク | Hardhat/Foundry | Anchor |
| アドレス生成 | CREATE/CREATE2 | PDA |
| 外部呼び出し | call/delegatecall | CPI |
| ガス/手数料 | 高い | **非常に低い** |
| スループット | ~15 TPS | **~65,000 TPS** |

🎯 **ポイント**:
- **Rustの知識が活きる** — 所有権、エラーハンドリング、型安全性
- **Anchor** = 生産性を大幅に向上するフレームワーク
- **アカウントモデル** — データとロジックの分離
- **PDA** — Programが所有できる特殊なアドレス
- **高速・低コスト** — Web3の実用的なユースケースに適している

---

## おわりに

この教材では、Python/TypeScriptの知識をベースに、Rustの基礎からWeb開発、そしてSolana/Anchorによるブロックチェーン開発まで学びました。

### 学習の振り返り

| Part | 内容 | 身についたスキル |
|-----|------|----------------|
| 基礎（Ch.0-4） | 型、所有権、エラー処理 | Rustコードの読み書き |
| 中級（Ch.5-8） | トレイト、非同期、モジュール | ライブラリの活用 |
| 応用（Ch.9-10） | Web API、スマートコントラクト | 実践的な開発 |

### 次のアクション

1. **小さなCLIツールを作る** — 学んだ基礎を実践
2. **Axumで簡単なAPIを作る** — CRUD操作を実装
3. **Solana Playgroundで実験** — ブラウザで気軽に試す
4. **既存のOSSを読む** — 実際のコードから学ぶ

**Happy Coding with Rust! 🦀**
