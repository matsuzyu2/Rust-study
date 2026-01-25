# Chapter 08: 非同期プログラミング 対話編

> **この章で学ぶこと**: async/await、Future、tokioランタイム — Rustでの非同期プログラミングの仕組みと実践

> 📖 **記号リファレンス**: この章では `async`, `await`, `move`, `'static` など多くのキーワードが登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## 登場人物紹介

**レイ（レイナ）**: 実務経験3年のTypeScript/Node.js開発者。最近Rustに興味を持ち始めた。非同期プログラミングにはPromiseで慣れている。

**ユイ**: Rustエンジニア。非同期プログラミングの深い知識を持ち、レイにRustの非同期の仕組みを教える。

---

## 1. 非同期プログラミングとの出会い

**レイ**: ねえユイ、Rustでも非同期プログラミングができるって聞いたんだけど、`async/await`ってJavaScriptと同じ感じで使えるの？

**ユイ**: うん、基本的な使い方は似てるよ。でも、Rustの非同期は**裏側の仕組みが全然違う**の。まずは簡単な例から見てみようか。

```rust
async fn hello() -> String {
    String::from("Hello, async world!")
}

#[tokio::main]
async fn main() {
    let result = hello().await;
    println!("{}", result);
}
```

**レイ**: あれ、`async function`じゃなくて`async fn`なんだね。それに`await`が後ろについてる！JavaScriptだと`await hello()`って書くのに。

**ユイ**: そう、それがRustの特徴！後置の`.await`はメソッドチェーンで使いやすくするためなの。

```rust
// Rustの後置.await — 読みやすい
let data = fetch_user()
    .await?
    .parse_json()
    .await?
    .validate()?;

// もし前置だったら — ネストが深くなる
let data = validate(await parse_json(await fetch_user())?)?;
```

**レイ**: なるほど！確かにメソッドチェーンだと後置の方が自然かも。あと`#[tokio::main]`って何？

**ユイ**: それは**tokioランタイム**を起動するマクロよ。実はね、Rustの`async/await`は**ランタイムがないと動かない**の。

**レイ**: え、JavaScriptだと普通に`async/await`使えるのに？

**ユイ**: そう、Node.jsやブラウザには最初からランタイムが組み込まれてるからね。でもRustは「使わない人にコストを払わせない」という哲学だから、ランタイムは外部クレートとして提供されてるの。

---

## 2. なぜ非同期処理が必要なのか

**レイ**: そもそもなんで非同期処理が必要なの？Node.jsだと当たり前に使ってたけど、改めて理由を聞かれると...

**ユイ**: いい質問！具体例で考えてみよう。APIからユーザー情報を取得して、データベースに保存して、別のAPIに通知する処理があるとするよね。

```
1. APIからユーザー情報を取得（100ms）
2. データベースに保存（50ms）
3. 別のAPIに通知（80ms）
```

**ユイ**: **同期処理**だと、こうなるの。

```
時間 →
[APIリクエスト待ち 100ms][DB待ち 50ms][API通知待ち 80ms]
合計: 230ms、この間スレッドは何もせず待機
```

**ユイ**: でも**非同期処理**なら、待ち時間中に他の処理ができる。

```
時間 →
[APIリクエスト]  [DB]  [API通知]
    ↓待ち時間中に他のリクエストを処理できる
```

**レイ**: ああ、I/O待ちの時間を無駄にしないってことね。Node.jsだとシングルスレッドでも大量のリクエストを捌けるのはそういうわけか。

**ユイ**: その通り！それに、Rustの非同期処理は**メモリ効率がすごくいい**の。

| モデル | 説明 | メモリ使用量 | 例 |
|-------|------|-------------|-----|
| 同期（ブロッキング） | 処理完了まで待つ | 効率的 | 従来のPython |
| マルチスレッド | 各接続にスレッド | 高い（スレッド毎にMB単位） | Java、Go |
| 非同期（ノンブロッキング） | 待ち時間に他の処理 | 低い（タスク毎にKB単位） | Node.js、Python asyncio、Rust |

**レイ**: マルチスレッドって1万接続だとどのくらいメモリ使うの？

**ユイ**: スレッドモデルだと数GBになっちゃう。でも非同期モデルなら数十MBで済むの。これがRustで高スループットなサーバーを作れる理由の一つよ。

---

## 3. Futureの秘密 — JavaScriptとの最大の違い

**レイ**: じゃあ基本的にPromiseと同じ感じで使えばいいんだね。

**ユイ**: ちょっと待って！ここからが**Rustの非同期の最大の特徴**なの。実は、`async fn`は**Futureを返す関数**なんだよ。

```rust
use std::future::Future;

// この2つは同等
async fn hello() -> String {
    String::from("Hello")
}

fn hello_explicit() -> impl Future<Output = String> {
    async {
        String::from("Hello")
    }
}
```

**レイ**: え、`impl Future<Output = String>`？

**ユイ**: うん。`async fn hello() -> String`って書くと、実際には「`String`を返すFutureを返す関数」になるの。

**レイ**: Promiseと同じようなものか。

**ユイ**: ううん、**ここが全然違う**の！見て、このコード。

```rust
async fn expensive_computation() -> i32 {
    println!("計算開始!");  // awaitされるまで実行されない！
    42
}

#[tokio::main]
async fn main() {
    let future = expensive_computation();  // ← ここでは何も実行されない
    println!("Futureを作成");

    let result = future.await;  // ← ここで初めて「計算開始!」が出力される
    println!("結果: {}", result);
}
```

**レイ**: え、何これ。出力順は...

**ユイ**: こうなるよ。

```
Futureを作成
計算開始!
結果: 42
```

**レイ**: ！？ `expensive_computation()`を呼んだのに実行されてない！

**ユイ**: そう、これが**Futureの遅延評価（Lazy）**！JavaScriptとの最大の違いなの。

```
┌─────────────────────────────────────────────────────────────────┐
│  JavaScript/Python vs Rust の非同期                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JavaScript:                                                    │
│    const promise = fetchData();  // 即座に実行開始               │
│    await promise;                // 完了を待つだけ               │
│                                                                 │
│  Python:                                                        │
│    coro = fetch_data()           # コルーチンオブジェクト作成     │
│    await coro                    # 実行開始＋完了待ち            │
│                                                                 │
│  Rust:                                                          │
│    let future = fetch_data();    // Futureを作成（何も実行しない）│
│    future.await;                 // 実行開始＋完了待ち            │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│  Rustの設計理由:                                                 │
│  1. ゼロコスト抽象化 — 使わないものにコストを払わない             │
│  2. 明示的な制御 — いつ実行するかを完全にコントロール             │
│  3. キャンセル — awaitしなければリソースを消費しない              │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: へえ...JavaScriptだと関数呼んだら即座にネットワークリクエストが飛ぶけど、Rustだとawaitするまで何もしないんだ。

**ユイ**: そう！これが「ゼロコスト抽象化」の一部。使わないFutureは何もコストがかからないの。それに、Futureをawaitしなければキャンセルと同じ効果になる。

**レイ**: なるほど、明示的でいいね。でも、なんでそういう設計になってるの？

**ユイ**: Futureの内部構造を見るとわかるよ。

---

## 4. Futureトレイトの内側

**ユイ**: Futureトレイトはこんな感じで定義されてるの。

```rust
// 標準ライブラリでの定義（簡略化）
trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

enum Poll<T> {
    Ready(T),    // 完了、結果はT
    Pending,     // まだ完了していない
}
```

**レイ**: `poll`メソッド...？JavaScriptのPromiseには見たことないメソッドだけど。

**ユイ**: そう、ここがRustの非同期の核心！Rustの非同期は**poll駆動**なの。

```
┌─────────────────────────────────────────────────────────────────┐
│  Future の実行モデル                                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ランタイム（tokio）がFutureのpoll()を呼ぶ                    │
│                                                                 │
│  2. Futureは処理を進める                                         │
│     - I/O待ちになったら → Poll::Pending を返す                   │
│     - 完了したら → Poll::Ready(結果) を返す                      │
│                                                                 │
│  3. Pendingの場合、ランタイムは他のFutureを実行                   │
│                                                                 │
│  4. I/Oが完了すると、ランタイムに通知（Waker）                    │
│                                                                 │
│  5. ランタイムは再度poll()を呼ぶ                                 │
│                                                                 │
│  これを「協調的マルチタスク」と呼ぶ                               │
│  （Futureが自発的に制御を返す）                                   │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: へえ、ランタイムが何度もpollを呼んで「まだ？」「まだ？」って聞いてる感じ？

**ユイ**: そう！でも賢いのは、I/Oが完了したら**Waker**っていう仕組みでランタイムに通知するから、無駄なポーリングはしないの。

**レイ**: なるほど。JavaScriptのPromiseはそういう仕組みが見えないけど、Rustだと明示的なんだね。

**ユイ**: うん。だからこそパフォーマンスチューニングもしやすいし、デバッグもしやすい。「なぜこのFutureが進まないのか」がわかるからね。

**レイ**: でも`Pin<&mut Self>`とか`Context<'_>`とか、難しそうなのが出てきた...

**ユイ**: それは後で説明するね。今は「pollで進捗を確認する」っていうモデルだけ覚えておけば大丈夫。普通にasync/awaitを使う分には意識しなくていいから。

---

## 5. tokioランタイムの世界

**レイ**: さっきから「tokioランタイム」って言ってるけど、それって何？

**ユイ**: tokioはRustで最も人気のある非同期ランタイムよ。JavaScriptならNode.jsやブラウザが非同期を動かす環境を提供してるでしょ？Rustではそれをtokioが担当するの。

**レイ**: なんで標準ライブラリに入ってないの？

**ユイ**: それがRustの哲学なの。

```
┌─────────────────────────────────────────────────────────────────┐
│  Rustの設計哲学                                                  │
├─────────────────────────────────────────────────────────────────┤
│  「ゼロコスト抽象化」と「選択の自由」                             │
│                                                                 │
│  - 非同期を使わないプログラムにランタイムのコストを強制しない      │
│  - 用途に応じて最適なランタイムを選べる                          │
│                                                                 │
│  主なランタイム:                                                 │
│  - tokio: 最も人気。汎用的。                                     │
│  - async-std: 標準ライブラリに近いAPI。                          │
│  - smol: 軽量。組み込み向け。                                    │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: ふーん、選択肢があるんだ。でもtokioが一番人気なんだよね？

**ユイ**: そう。Web開発ではほぼtokio一択と言っていいくらい。セットアップは簡単よ。

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
```

**レイ**: `features = ["full"]`？

**ユイ**: tokioは機能がモジュール化されてるの。`"full"`は全部入りで、開発中は便利。本番では使う機能だけ選ぶと軽くなるよ。

```toml
# 本番環境の例
tokio = { version = "1", features = ["rt-multi-thread", "net", "time"] }
```

**レイ**: なるほど。じゃあ使ってみよう。

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    println!("開始");
    sleep(Duration::from_secs(1)).await;
    println!("1秒後");
}
```

**ユイ**: いいね！ちなみに`#[tokio::main]`は実はマクロで、こう展開されるの。

```rust
// #[tokio::main]
// async fn main() { ... }

// ↓ 展開後

fn main() {
    tokio::runtime::Builder::new_multi_thread()
        .enable_all()
        .build()
        .unwrap()
        .block_on(async {
            // ここにasync fn main()の中身
        })
}
```

**レイ**: へえ、`block_on`でasyncのmainを実行してるんだ。

**ユイ**: そう。これを知ってるとランタイムをカスタマイズできるよ。

```rust
// シングルスレッドランタイム（軽量）
#[tokio::main(flavor = "current_thread")]
async fn main() { ... }

// ワーカースレッド数を指定
#[tokio::main(worker_threads = 4)]
async fn main() { ... }
```

---

## 6. 並行実行の魔法 — join!とselect!

**レイ**: ねえ、JavaScriptの`Promise.all`みたいに複数の非同期処理を並行実行したいんだけど、どうすればいい？

**ユイ**: `tokio::join!`を使えばいいよ！

```rust
use tokio::time::{sleep, Duration};

async fn task1() -> String {
    println!("Task1: 開始");
    sleep(Duration::from_secs(1)).await;
    println!("Task1: 完了");
    String::from("Task 1 の結果")
}

async fn task2() -> String {
    println!("Task2: 開始");
    sleep(Duration::from_secs(2)).await;
    println!("Task2: 完了");
    String::from("Task 2 の結果")
}

#[tokio::main]
async fn main() {
    // 並行実行（合計2秒で完了）
    let (result1, result2) = tokio::join!(task1(), task2());

    println!("{}", result1);
    println!("{}", result2);
}
```

**レイ**: 出力は...？

**ユイ**: こうなるよ。

```
Task1: 開始
Task2: 開始
Task1: 完了    ← 1秒後
Task2: 完了    ← 2秒後（task1と並行して進んでいた）
Task 1 の結果
Task 2 の結果
```

**レイ**: おお、ちゃんと並行実行されてる！合計2秒で終わってるもんね。これ並列実行？

**ユイ**: いい質問！実は**並行（concurrent）**であって**並列（parallel）**ではないの。

```
┌─────────────────────────────────────────────────────────────────┐
│  並行 vs 並列                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  並行（Concurrent）:                                             │
│    1つのスレッドで複数のタスクを交互に実行                        │
│    タスクA → I/O待ち → タスクB → I/O待ち → タスクA...            │
│                                                                 │
│  並列（Parallel）:                                               │
│    複数のスレッド/CPUで同時に実行                                 │
│    スレッド1: タスクA                                            │
│    スレッド2: タスクB                                            │
│                                                                 │
│  tokio::join! → 並行                                            │
│  tokio::spawn → 並列も可能（マルチスレッドランタイムの場合）       │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: ああ、Node.jsと同じか。シングルスレッドで並行処理。

**ユイ**: デフォルトのtokioランタイムはマルチスレッドだから、状況によっては並列にもなるけどね。

**レイ**: じゃあ`Promise.race`みたいに「最初に終わった方」を待つのは？

**ユイ**: `tokio::select!`を使うよ。

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    tokio::select! {
        _ = sleep(Duration::from_secs(1)) => {
            println!("1秒経過");
        }
        _ = sleep(Duration::from_secs(2)) => {
            println!("2秒経過");  // これは実行されない
        }
    }
}
```

**レイ**: これタイムアウト処理に使えそう！

**ユイ**: その通り！tokioには専用の関数もあるよ。

```rust
use tokio::time::{sleep, Duration, timeout};

async fn slow_operation() -> String {
    sleep(Duration::from_secs(10)).await;
    String::from("完了")
}

#[tokio::main]
async fn main() {
    // 5秒でタイムアウト
    match timeout(Duration::from_secs(5), slow_operation()).await {
        Ok(result) => println!("成功: {}", result),
        Err(_) => println!("タイムアウト！"),
    }
}
```

**レイ**: これめっちゃ便利じゃん！

---

## 7. spawn — バックグラウンドタスクの落とし穴

**レイ**: ところで、バックグラウンドでタスクを実行したい場合は？

**ユイ**: `tokio::spawn`を使うよ。

```rust
use tokio::time::{sleep, Duration};

#[tokio::main]
async fn main() {
    // タスクをspawn（バックグラウンドで実行開始）
    let handle = tokio::spawn(async {
        sleep(Duration::from_secs(1)).await;
        42
    });

    println!("タスクを生成しました");

    // 他の処理をしている間もタスクは進行中

    // 結果を待つ
    let result = handle.await.unwrap();
    println!("結果: {}", result);
}
```

**レイ**: `join!`と何が違うの？

**ユイ**: `join!`は現在のタスク内で並行実行するけど、`spawn`は**別タスク**として実行するの。マルチスレッドランタイムなら別のCPUコアで動く可能性もある。

**レイ**: じゃあspawnの方がいいじゃん！

**ユイ**: ちょっと待って。spawnには**制約**があるの。これが初心者の落とし穴。

```rust
// ❌ コンパイルエラー！
fn bad_spawn() {
    let data = String::from("hello");

    tokio::spawn(async {
        println!("{}", data);  // dataを借用
    });  // エラー: `data`はこのスコープで破棄されるが、タスクはまだ使おうとしている
}
```

**レイ**: あ、借用のエラー？

**ユイ**: そう。spawnされたタスクはバックグラウンドで動くから、いつまで生きてるかわからない。だから`data`が先に破棄されちゃうかもしれないの。

**レイ**: どうすればいいの？

**ユイ**: `move`で所有権を移動させるの。

```rust
// ✅ move で所有権を移動
fn good_spawn() {
    let data = String::from("hello");

    tokio::spawn(async move {  // move を追加
        println!("{}", data);  // dataの所有権を持っている
    });
}
```

**レイ**: ああ、`async move`か。これChapter 5のクロージャで習ったやつだ。

**ユイ**: その通り！でも制約はそれだけじゃないの。

```
┌─────────────────────────────────────────────────────────────────┐
│  spawn と 'static 境界                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  tokio::spawn(future) の future には以下が要求される：           │
│                                                                 │
│  1. Send — 別スレッドに送れる                                    │
│     （マルチスレッドランタイムでは別スレッドで実行される可能性）   │
│                                                                 │
│  2. 'static — 参照がない、または'staticな参照のみ                 │
│     （タスクがいつまで生存するか不明なため）                      │
│                                                                 │
│  move を使うと:                                                  │
│  - 変数の所有権がクロージャに移動                                 │
│  - クロージャは外部への参照を持たなくなる                         │
│  - 'static を満たせる                                            │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: `Send`と`'static`...難しそう。

**ユイ**: 例を見せるね。これはエラーになるよ。

```rust
use std::rc::Rc;

// ❌ Rc は Send を実装していない
let data = Rc::new(42);
tokio::spawn(async move {
    println!("{}", data);  // エラー！
});
```

**レイ**: `Rc`がダメ？

**ユイ**: `Rc`はシングルスレッド専用の参照カウンタだから、`Send`を実装してないの。マルチスレッドで使える`Arc`なら大丈夫。

```rust
// ✅ Arc は Send を実装している
use std::sync::Arc;
let data = Arc::new(42);
tokio::spawn(async move {
    println!("{}", data);  // OK
});
```

**レイ**: なるほど...spawnは便利だけど、制約がいろいろあるんだね。

**ユイ**: そう。だから**基本的にはjoin!を使って、本当に必要なときだけspawnを使う**のがおすすめよ。

---

## 8. async moveの正体

**レイ**: `async move`ってもうちょっと詳しく教えて。いつ使えばいいの？

**ユイ**: いい質問！比較して見せるね。

```rust
async fn process_data(data: String) {
    println!("処理: {}", data);
}

#[tokio::main]
async fn main() {
    let data = String::from("important data");

    // move なし — data を借用
    let task1 = async {
        println!("{}", data);  // 借用
    };

    // move あり — data の所有権を奪う
    let task2 = async move {
        println!("{}", data);  // 所有
    };

    // task2 の後、data は使えない
    // println!("{}", data);  // ❌ エラー！所有権はtask2に移動した
}
```

**レイ**: ああ、普通のクロージャの`move`と同じか。

**ユイ**: そう。でも`async move`が**特に必要になるパターン**があるの。

```
┌─────────────────────────────────────────────────────────────────┐
│  async move が必要なパターン                                     │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. tokio::spawn に渡す                                         │
│     tokio::spawn(async move { ... });                           │
│                                                                 │
│  2. Futureを返す関数                                            │
│     fn foo() -> impl Future { async move { ... } }             │
│                                                                 │
│  3. Futureが元のスコープより長く生存                             │
│     let future = async move { ... };                            │
│     // 元のスコープを抜けた後もfutureを使いたい                   │
│                                                                 │
│  4. ループ内でspawn                                             │
│     for item in items {                                         │
│         tokio::spawn(async move { use(item) });                 │
│     }                                                           │
└─────────────────────────────────────────────────────────────────┘
```

**レイ**: ループ内でspawnする例見せて。

**ユイ**: こういう感じ。

```rust
#[tokio::main]
async fn main() {
    let items = vec![1, 2, 3, 4, 5];

    let mut handles = vec![];
    for item in items {
        let handle = tokio::spawn(async move {
            // item を所有しているので、タスクが終わるまで有効
            println!("処理中: {}", item);
            item * 2
        });
        handles.push(handle);
    }

    // 全タスクの完了を待つ
    for handle in handles {
        let result = handle.await.unwrap();
        println!("結果: {}", result);
    }
}
```

**レイ**: `async move`がないとどうなるの？

**ユイ**: `item`の借用が問題になるよ。ループが次に進んだら`item`の値が変わっちゃうから、タスクが参照する値が不定になる。

**レイ**: なるほど、所有権を渡すことで各タスクが独立するんだ。

---

## 9. 非同期とライフタイムの戦い

**レイ**: ライフタイムとasyncって相性悪そう...

**ユイ**: そうなの！よくあるエラーパターンを見せるね。

```rust
// ❌ コンパイルエラー！
async fn bad_example(data: &String) {
    tokio::spawn(async {
        println!("{}", data);  // dataは借用だが、spawnされたタスクの生存期間は不明
    });
}
```

**レイ**: これなんでダメなの？

**ユイ**: `data`は参照（借用）でしょ？でもspawnされたタスクがいつまで生きてるかわからない。だから`data`が先に破棄されちゃうかもしれないの。

```
error[E0597]: `data` does not live long enough
```

**レイ**: じゃあどうすればいいの？

**ユイ**: 3つの解決策があるよ。

**解決策1: クローン**

```rust
async fn good_example(data: &String) {
    let owned_data = data.clone();  // クローンして所有
    tokio::spawn(async move {
        println!("{}", owned_data);
    });
}
```

**レイ**: クローンか...コストかかりそう。

**ユイ**: そうだね。大きなデータだと気になるかも。そういうときは次の方法。

**解決策2: Arcで共有**

```rust
use std::sync::Arc;

async fn good_example(data: Arc<String>) {
    let data_clone = Arc::clone(&data);
    tokio::spawn(async move {
        println!("{}", data_clone);
    });
}
```

**レイ**: `Arc::clone`ってコスト低いんだっけ？

**ユイ**: うん、参照カウントをインクリメントするだけだから、実際のデータはコピーされないの。複数のタスクで同じデータを共有したいときに便利。

**レイ**: 他の解決策は？

**解決策3: spawnせずにjoin!**

```rust
async fn good_example(data: &String) {
    // spawnではなくjoin!なら借用でOK
    tokio::join!(
        async { println!("1: {}", data) },
        async { println!("2: {}", data) }
    );
}
```

**レイ**: ああ、`join!`は同じスコープ内で動くから借用でもいいんだ。

**ユイ**: その通り！だから、できるだけ`join!`を使って、本当に必要なときだけ`spawn`を使うのがおすすめなの。

---

## 10. 実践: HTTPリクエストとファイル操作

**レイ**: 実際のコードでasyncを使ってみたいな。

**ユイ**: じゃあHTTPリクエストをやってみよう。`reqwest`クレートを使うよ。

```toml
# Cargo.toml
[dependencies]
tokio = { version = "1", features = ["full"] }
reqwest = { version = "0.11", features = ["json"] }
serde = { version = "1.0", features = ["derive"] }
```

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize)]
struct User {
    id: i32,
    name: String,
}

#[tokio::main]
async fn main() -> Result<(), reqwest::Error> {
    let user: User = reqwest::get("https://api.example.com/users/1")
        .await?          // リクエスト完了を待つ
        .json()          // JSONとしてパース（これも非同期）
        .await?;

    println!("{:?}", user);
    Ok(())
}
```

**レイ**: あれ、`?`が2回ある？

**ユイ**: そう、いい質問！

1. 最初の`?` — HTTPリクエストのエラーを伝播
2. 2番目の`?` — JSONパースのエラーを伝播

**レイ**: JavaScriptだとこんな感じだよね。

```javascript
const response = await fetch("https://api.example.com/users/1");
const user = await response.json();
```

**ユイ**: そうそう。Rustの方が明示的にエラーハンドリングしてるね。

**レイ**: ファイル操作は？

**ユイ**: tokioには非同期ファイルI/Oもあるよ。

```rust
use tokio::fs;

#[tokio::main]
async fn main() -> std::io::Result<()> {
    // ファイル書き込み
    fs::write("hello.txt", "Hello, Tokio!").await?;

    // ファイル読み込み
    let content = fs::read_to_string("hello.txt").await?;
    println!("{}", content);

    Ok(())
}
```

**レイ**: めっちゃシンプル！

**ユイ**: 注意点として、小さいファイルなら同期I/Oの方が速いこともあるの。非同期I/Oは**大量のファイルを並行処理する**ときに真価を発揮するよ。

---

## 11. チャネル — タスク間通信の魔法

**レイ**: 複数のタスクでデータをやり取りしたいんだけど、どうすればいい？

**ユイ**: **チャネル**を使うといいよ！

```rust
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    // バッファサイズ32のチャネルを作成
    let (tx, mut rx) = mpsc::channel(32);

    // 送信側タスク
    let tx_clone = tx.clone();  // 複数の送信者を作れる
    tokio::spawn(async move {
        for i in 0..5 {
            tx_clone.send(i).await.unwrap();
        }
    });

    // 別の送信者
    tokio::spawn(async move {
        for i in 100..105 {
            tx.send(i).await.unwrap();
        }
    });

    // 受信側（全送信者がdropされるとNoneが返る）
    while let Some(value) = rx.recv().await {
        println!("受信: {}", value);
    }
}
```

**レイ**: おお、複数の送信者から1つの受信者！

**ユイ**: これを**mpsc（multiple producer, single consumer）**って呼ぶの。他にもチャネルの種類があるよ。

| 種類 | 用途 | 特徴 |
|-----|------|------|
| `mpsc` | 多対一 | 複数のProducerから1つのConsumerへ |
| `oneshot` | 一対一、一回限り | 結果を1回だけ返す |
| `broadcast` | 一対多 | 全購読者にメッセージを送信 |
| `watch` | 一対多、最新値のみ | 設定変更の通知など |

**レイ**: `oneshot`って何に使うの？

**ユイ**: バックグラウンドタスクから1回だけ結果を返すときとかね。

```rust
use tokio::sync::oneshot;

#[tokio::main]
async fn main() {
    let (tx, rx) = oneshot::channel();

    tokio::spawn(async move {
        // 重い計算
        let result = 42;
        tx.send(result).unwrap();
    });

    let result = rx.await.unwrap();
    println!("結果: {}", result);
}
```

**レイ**: なるほど、シンプル！

---

## 12. spawn_blockingの重要性

**レイ**: 重い計算処理を非同期タスクの中でやっちゃいけないって聞いたんだけど...

**ユイ**: いい情報を掴んだね！CPUバウンドな処理は**非同期タスクをブロック**しちゃうの。

```rust
// ❌ 悪い例
#[tokio::main]
async fn main() {
    // 重い計算（10秒かかる）
    let mut sum = 0;
    for i in 0..10_000_000_000 {
        sum += i;
    }

    // この間、他の非同期タスクが止まってしまう！
}
```

**レイ**: え、なんで？非同期なんだから大丈夫じゃないの？

**ユイ**: 非同期は**I/O待ちの間に他のタスクを実行する**仕組みでしょ？でもCPU計算中はI/O待ちじゃないから、制御を返さないの。

**レイ**: ああ、なるほど...

**ユイ**: だから`spawn_blocking`を使うの。これは別スレッドで実行されるから、非同期ランタイムをブロックしない。

```rust
use tokio::task::spawn_blocking;

fn heavy_computation() -> i32 {
    // CPUバウンドな同期処理
    let mut sum = 0;
    for i in 0..1_000_000 {
        sum += i;
    }
    sum
}

#[tokio::main]
async fn main() {
    // 別スレッドで実行（非同期ランタイムをブロックしない）
    let result = spawn_blocking(|| heavy_computation()).await.unwrap();
    println!("結果: {}", result);
}
```

**レイ**: いつ`spawn_blocking`が必要なの？

**ユイ**: こういうとき。

```
いつspawn_blockingが必要か:
- 重い計算処理
- 同期的なライブラリを使う場合
- std::thread::sleepを使う場合（非同期ではtokio::time::sleepを使う）
```

**レイ**: `std::thread::sleep`を使っちゃダメなの？

**ユイ**: asyncの中で使うと、その間**ランタイム全体が止まっちゃう**の。

```rust
// ❌ 悪い例
async fn bad_sleep() {
    std::thread::sleep(Duration::from_secs(1));  // ランタイムが止まる！
}

// ✅ 良い例
async fn good_sleep() {
    tokio::time::sleep(Duration::from_secs(1)).await;  // 他のタスクは動き続ける
}
```

**レイ**: 気をつけないとな...

---

## 13. 実践: 並列API呼び出し

**レイ**: 複数のAPIを並列に呼び出す実践的なコード見せて！

**ユイ**: いいね、よくあるパターンだよ。

```rust
use reqwest::Client;
use serde::Deserialize;
use anyhow::Result;

#[derive(Debug, Deserialize)]
struct User {
    id: i32,
    name: String,
}

async fn fetch_user(client: &Client, id: i32) -> Result<User> {
    let url = format!("https://api.example.com/users/{}", id);
    let user = client.get(&url).send().await?.json().await?;
    Ok(user)
}

#[tokio::main]
async fn main() -> Result<()> {
    let client = Client::new();

    // 複数ユーザーを並行取得
    let ids = vec![1, 2, 3, 4, 5];

    // 各IDに対してFutureを作成
    let futures: Vec<_> = ids.iter()
        .map(|&id| fetch_user(&client, id))
        .collect();

    // すべてのFutureを並行実行
    let results = futures::future::join_all(futures).await;

    for result in results {
        match result {
            Ok(user) => println!("{:?}", user),
            Err(e) => eprintln!("Error: {}", e),
        }
    }

    Ok(())
}
```

**レイ**: `futures::future::join_all`？これtokioじゃないの？

**ユイ**: これは`futures`クレートの関数。可変長のFutureを扱うときに便利なの。`tokio::join!`はマクロで、コンパイル時に個数が決まってないといけないから。

**レイ**: なるほど。JavaScriptだと...

```javascript
const ids = [1, 2, 3, 4, 5];
const promises = ids.map(id => fetchUser(client, id));
const results = await Promise.all(promises);
```

**ユイ**: そうそう、同じ発想だね。

---

## 14. エラーハンドリングのベストプラクティス

**レイ**: asyncでのエラーハンドリングってどうすればいい？

**ユイ**: 基本は普通のRustと同じで`Result`と`?`を使うよ。

```rust
use tokio::fs;
use anyhow::{Context, Result};

async fn read_config() -> Result<String> {
    let content = fs::read_to_string("config.toml")
        .await
        .context("設定ファイルが読み込めません")?;
    Ok(content)
}

#[tokio::main]
async fn main() -> Result<()> {
    match read_config().await {
        Ok(config) => println!("Config: {}", config),
        Err(e) => eprintln!("Error: {:?}", e),
    }
    Ok(())
}
```

**レイ**: `anyhow`って何？

**ユイ**: エラーハンドリングを楽にするクレート。`context`でエラーにコンテキストを追加できるの。すごく便利よ。

**レイ**: `main`も`Result`を返せるんだ。

**ユイ**: うん、`#[tokio::main]`が対応してるの。エラーが起きたらプログラムが終了して、エラーメッセージを出力してくれる。

---

## 15. Pinの謎（上級者向け）

**レイ**: さっき`Pin<&mut Self>`っていうのがあったけど、あれって何？

**ユイ**: これは上級トピックだけど、興味あるなら説明するよ。

**レイ**: 教えて！

**ユイ**: `async`ブロックは内部で**自己参照構造体**を生成することがあるの。

```rust
async fn example() {
    let data = vec![1, 2, 3];
    let reference = &data;  // dataへの参照

    some_async_operation().await;  // ← この間に構造体がメモリ上で移動すると...

    println!("{:?}", reference);  // referenceが無効になる可能性！
}
```

**レイ**: `.await`をまたいで参照を持ってるのが問題？

**ユイ**: そう。Futureは内部でステートマシンになってて、`.await`ごとに状態が変わるの。その際にメモリ上で移動（ムーブ）すると、自己参照が壊れちゃう。

**レイ**: それで`Pin`？

**ユイ**: うん。`Pin`は「この値はメモリ上で移動しない」ことを保証する型なの。

```rust
use std::pin::Pin;

// poll の定義
fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
//           ^^^^^^^^^^^^
//           Pin<&mut Self> = 「このFutureは移動しない」という保証
```

**レイ**: 普通にasync/awaitを使うだけなら意識しなくていい？

**ユイ**: そう！コンパイラとランタイムが全部やってくれるから。`Pin`を意識するのは、手動でFutureを実装するときとか、ライブラリを作るときだけ。

**レイ**: ふー、難しい世界だ...

**ユイ**: でも知っておくと、非同期の裏側がわかって面白いでしょ？

---

## 16. JavaScript/TypeScriptからの移行ガイド

**レイ**: ここまで学んで、JavaScriptとの違いをまとめたいな。

**ユイ**: いいね！表にまとめてみよう。

| 概念 | JavaScript | Python | Rust |
|-----|------------|--------|------|
| 非同期関数 | `async function` | `async def` | `async fn` |
| 待機 | `await` | `await` | `.await`（後置） |
| Promise/Future | Promise | Coroutine | Future |
| ランタイム | 組み込み | asyncio | tokio（外部） |
| 並行実行 | `Promise.all` | `asyncio.gather` | `tokio::join!` |
| レース | `Promise.race` | `asyncio.wait` | `tokio::select!` |
| 遅延評価 | ❌（即時実行） | △ | ✅（明示的） |

**レイ**: 最大の違いは「遅延評価」だね。

**ユイ**: そう。Rustは`await`するまで何も起きないから、**明示的にコントロールできる**のが強み。

**レイ**: あと、GILがないのも重要だよね。

**ユイ**: そうなの！PythonはGIL（Global Interpreter Lock）があるから、マルチスレッドでもCPU並列は効かない。でもRustは真のマルチスレッドができる。

```
GILがない = 真の並列

Python: GILのため、CPUバウンドな処理は1コアしか使えない
        asyncioはI/Oバウンドな処理には有効だが、CPU並列はmultiprocessingが必要

Rust:   GILなし。tokio::spawnされたタスクは異なるCPUコアで実行される可能性
```

---

## 17. よくあるコンパイルエラーと対処法

**レイ**: 最後に、よくあるエラーと対処法を教えて。

**ユイ**: いいね。3つの頻出エラーを覚えておこう。

**エラー1: `future cannot be sent between threads safely`**

```
error: future cannot be sent between threads safely
...the trait `Send` is not implemented for `Rc<...>`
```

**原因**: `Rc`など`Send`を実装していない型を使っている
**対処**: `Arc`に置き換える

```rust
// ❌ use std::rc::Rc;
// ✅ use std::sync::Arc;
```

**エラー2: `borrowed value does not live long enough`**

```
error[E0597]: `data` does not live long enough
```

**原因**: spawnされたタスクが参照を持っているが、元の変数が先に破棄される
**対処**: `clone()`または`Arc`で所有権を渡す、または`move`を使う

**エラー3: `async block may outlive the current function`**

**原因**: async blockが関数より長く生存する可能性があるが、ローカル変数を借用している
**対処**: `async move`を使って所有権を移動

**レイ**: これ覚えておけば大丈夫そう。

**ユイ**: あとは実際にコードを書いて、エラーメッセージと格闘することね。Rustのコンパイラはエラーメッセージが親切だから、よく読めば解決できるよ。

---

## まとめ

**レイ**: Rustの非同期、最初は難しそうだったけど理解できてきた！

**ユイ**: すごい！最後にポイントをおさらいしよう。

🎯 **この章のポイント**:

1. **Futureは遅延評価** — `await`するまで実行されない
2. **ランタイムが必要** — tokioが最もポピュラー
3. **join!** = 並行実行、**select!** = レース
4. **spawn** = バックグラウンドタスク（`Send + 'static`が必要）
5. **async move** = クロージャに所有権を移動
6. **チャネル** = タスク間通信
7. **spawn_blocking** = CPU重い処理は別スレッドへ

**レイ**: JavaScriptと似てるようで、全然違うんだね。

**ユイ**: でもその違いこそが、Rustの**パフォーマンス**と**安全性**を実現してるの。非同期処理でもメモリ安全性を保証できるのがRustの強みよ。

**レイ**: 次は実際にWebアプリケーションを作りたいな。

**ユイ**: いいね！[Chapter 09: Web開発](09-web-development.md) では、Axumフレームワークを使ったREST API開発を学ぶよ。この章で学んだ非同期の知識が全部活きてくるから楽しみにしてて！

**レイ**: 楽しみ！ユイ、ありがとう！

**ユイ**: どういたしまして。一緒に頑張ろうね！

---

## 次のステップ

[Chapter 09: Web開発](09-web-development.md) では、Axumフレームワークを使ったREST API開発を学びます。この章で学んだ非同期の知識が実践的に活用されます。

---

**🎓 学習チェックリスト**

- [ ] async/awaitの基本文法を理解した
- [ ] Futureの遅延評価を理解した
- [ ] tokioランタイムの役割を理解した
- [ ] join!とselect!の使い分けができる
- [ ] spawnの制約（Send + 'static）を理解した
- [ ] async moveをいつ使うか理解した
- [ ] チャネルでタスク間通信ができる
- [ ] spawn_blockingの重要性を理解した
- [ ] よくあるコンパイルエラーに対処できる

---

**📚 さらに学びたい人へ**

- [Tokio公式チュートリアル](https://tokio.rs/tokio/tutorial)
- [Async Book](https://rust-lang.github.io/async-book/)
- [Jon Gjengset - Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM)

---

💬 **レイより**: 非同期プログラミング、最初は難しいと思ってたけど、ユイのおかげで理解できた！Futureの遅延評価とか、JavaScriptとは全然違う世界で新鮮だったな。次のWeb開発の章が楽しみ！
