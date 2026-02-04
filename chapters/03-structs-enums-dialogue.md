# Chapter 03: 構造体・列挙型

> **この章で学ぶこと**: struct、enum、impl、パターンマッチ — Rustでデータを構造化する方法

> 📖 **記号リファレンス**: この章では `impl`, `Self`, `::`, `&self` など重要な構文が登場します。意味がわからなくなったら [Chapter 11: 記号・構文リファレンス](11-syntax-reference.md) を参照してください。

---

## プロローグ：クラスがない！？

ユイが意気揚々とエディタを開く。「今日こそRustでちゃんとしたアプリを書くぞ！」

> 👩‍💻 **ユイ**: 「先輩、ユーザー管理の機能作りたいんですけど、まずクラスを…」

```rust
// ユイの最初の試み（Pythonの感覚で）
class User {  // ❌ コンパイルエラー！
    username: String,
    email: String,
}
```

```
error: expected one of `!` or `::`, found `User`
 --> src/main.rs:1:7
  |
1 | class User {
  |       ^^^^ expected one of `!` or `::`
```

> 👩‍💻 **ユイ**: 「…え？`class`が認識されない？スペルミス…？」
>
> 👩‍🏫 **レイ**: （後ろから画面を覗き込む）「あぁ、それね。Rustには**classキーワードがない**のよ。」
>
> 👩‍💻 **ユイ**: 「えっっ！？class**ない**んですか！？じゃあオブジェクト指向は…？継承は…？ポリモーフィズムは！？」
>
> 👩‍🏫 **レイ**: （クスッと笑う）「落ち着いて。その反応、私も最初同じだったから分かるわ。でもね、classがないのには**ちゃんとした理由**があるの。」
>
> 👩‍💻 **ユイ**: 「理由…？でも私、PythonでもTypeScriptでもずっとclass使ってきたんです。それなしでどうやってデータとメソッドをまとめるんですか？」
>
> 👩‍🏫 **レイ**: 「**構造体（struct）**と**実装（impl）**を使うの。データとメソッドを**別々に定義**するのよ。」
>
> 👩‍💻 **ユイ**: 「別々に…？それって面倒じゃないですか？」
>
> 👩‍🏫 **レイ**: 「最初はそう感じるわよね。でも実は、これは『面倒』じゃなくて『設計ミスを防ぐ仕組み』なの。ホワイトボード使って、**なぜRustがclassを捨てたのか**から説明するわね。」

### なぜRustにクラスがないのか — オブジェクト指向の歴史から

> 👩‍🏫 **レイ**: 「まず、ちょっと歴史の話をさせて。オブジェクト指向って、1960年代のSimulaで始まって、1990年代にJavaとC++で爆発的に普及したのよ。」
>
> 👩‍💻 **ユイ**: 「はい、『すべてはオブジェクト』みたいな思想ですよね。」
>
> 👩‍🏫 **レイ**: 「そう。でも2000年代に入って、オブジェクト指向の**継承**に対する反省が広がったの。」

```
┌─────────────────────────────────────────────────────────────────┐
│  オブジェクト指向の『継承』が引き起こした問題                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ダイヤモンド継承問題                                         │
│     Animal                                                      │
│      ↙   ↘                                                      │
│   Bird   Mammal                                                 │
│      ↘   ↙                                                      │
│       Bat     ← 飛べる哺乳類。どちらの特性を継承？               │
│                                                                 │
│  2. 脆弱な基底クラス問題（Fragile Base Class Problem）           │
│     親クラスの変更が、予期しない子クラスを壊す                    │
│     → 大規模システムでメンテナンス地獄に                         │
│                                                                 │
│  3. is-a関係の誤用                                               │
│     『正方形 is-a 長方形』→ でも正方形のwidthを変えたら          │
│     heightも変わるべき？ → リスコフの置換原則違反                │
│                                                                 │
│  有名な引用:                                                     │
│  "Prefer composition over inheritance"                          │
│  （継承より合成を好め）— GoFデザインパターン (1994年)             │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「あっ、『継承より合成』って聞いたことあります！でも正直、なんで継承がダメなのかピンと来てなくて…」
>
> 👩‍🏫 **レイ**: 「私も最初はそうだったわ。でもね…」

（レイが少し遠い目をする）

> 👩‍🏫 **レイ**: 「前職で、**継承地獄**に巻き込まれたことがあってね。」
>
> 👩‍💻 **ユイ**: 「継承地獄…？」
>
> 👩‍🏫 **レイ**: 「とあるJavaプロジェクトで、`BaseEntity` → `AbstractUser` → `RegisteredUser` → `PremiumUser` → `EnterpriseUser`って、**5段階の継承**があったの。」
>
> 👩‍💻 **ユイ**: 「うわぁ…」
>
> 👩‍🏫 **レイ**: 「で、『BaseEntityにcreatedAtフィールド追加して』って言われて、軽い気持ちで追加したら…」
>
> 👩‍💻 **ユイ**: 「…なにが起きたんですか？」
>
> 👩‍🏫 **レイ**: 「**テストが47個落ちた**の。孫クラスのどこかでcreatedAtって名前の別のフィールド使ってて、それと衝突したのよ。修正に2日かかったわ。」
>
> 👩‍💻 **ユイ**: 「ひぃ…！それは怖い…」
>
> 👩‍🏫 **レイ**: 「そう。GoFが1994年に『継承より合成を』って言ったけど、当時のJavaやC++は継承が言語の中心だったから、**プログラマの心がけに頼るしかなかった**の。で、Rustはどうしたと思う？」
>
> 👩‍💻 **ユイ**: 「えっと…心がけじゃなくて…」
>
> 👩‍🏫 **レイ**: 「**言語レベルで継承を削除した**の。2010年代に設計された言語として、過去30年の教訓を活かしたのよ。代わりに**トレイト**と**合成**でコードを再利用する。強制的に『良い設計』しかできないようにしたの。」
>
> 👩‍💻 **ユイ**: 「なるほど…！つまり、私がclassって書こうとしてエラーになったのは、**Rustが私を守ってくれてた**ってことですか？」
>
> 👩‍🏫 **レイ**: 「その通り！さすが飲み込みが早いわね。Rustは『プログラマを信用しない』言語なのよ。いい意味でね。」

---

## なぜ構造体と列挙型が重要なのか

> 👩‍🏫 **レイ**: （ホワイトボードに書き始める）「各言語でデータ構造をどう扱うか比較するわね。」

```
┌─────────────────────────────────────────────────────────────────┐
│  各言語のデータ構造化                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  Python:                                                        │
│  - class で データ + メソッド を定義                             │
│  - 継承でコードを再利用                                          │
│  - @dataclass で単純なデータ構造                                 │
│                                                                 │
│  TypeScript:                                                    │
│  - interface で型を定義                                          │
│  - class でデータ + メソッド                                     │
│  - type で ユニオン型                                            │
│                                                                 │
│  Rust:                                                          │
│  - struct でデータを定義                                         │
│  - impl でメソッドを別に定義 ← 分離されている！                    │
│  - enum でデータを持つバリアント（TypeScriptのユニオン型相当）    │
│  - trait で共通インターフェース（継承の代わり）                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「データとメソッドを分けるって、どういうことですか…？」
>
> 👩‍🏫 **レイ**: 「実際にコード書いて見せるわね。」

---

## 構造体（Struct）— クラスの代わり

### 最初の失敗から学ぶ

> 👩‍🏫 **レイ**: 「じゃあ、さっきのUserを構造体で書き直してみて。」
>
> 👩‍💻 **ユイ**: 「はい！えっと…classをstructに変えて…」

```rust
// ユイの2回目の試み
struct User {
    username: String,
    email: String,
}

fn main() {
    let user1 = User {
        username: "someuser",  // ❌ エラー！
        email: "user@example.com",
    };
}
```

```
error[E0308]: mismatched types
 --> src/main.rs:8:19
  |
8 |         username: "someuser",
  |                   ^^^^^^^^^^ expected `String`, found `&str`
```

> 👩‍💻 **ユイ**: 「え！？また怒られた！`String`と`&str`が違う…？」
>
> 👩‍🏫 **レイ**: 「あぁ、これも**Rust初心者あるある**ね。`"someuser"`は文字列リテラルで、型は`&str`（文字列スライス）なの。`String`型のフィールドには`String`を渡さないといけないわ。」
>
> 👩‍💻 **ユイ**: 「えっと、どうすれば…」
>
> 👩‍🏫 **レイ**: 「`String::from()`か`.to_string()`を使うの。」

```rust
fn main() {
    let user1 = User {
        username: String::from("someuser"),  // ✅ OK
        email: "user@example.com".to_string(), // ✅ これもOK
    };
}
```

> 👩‍💻 **ユイ**: 「動いた！でも、なんでこんな区別があるんですか？Pythonだと全部`str`じゃないですか。」
>
> 👩‍🏫 **レイ**: 「いい質問ね。これはメモリの話と密接に関係してるの。」

```
┌─────────────────────────────────────────────────────────────────┐
│  Stringと&strの違い — メモリの視点から                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  &str（文字列スライス）                                          │
│  ├─ 固定サイズ、読み取り専用                                     │
│  ├─ "hello" のようなリテラルはプログラムに埋め込まれる            │
│  └─ 所有権なし（借用）                                           │
│                                                                 │
│  String（文字列）                                                │
│  ├─ 可変長、変更可能                                             │
│  ├─ ヒープにメモリを確保                                         │
│  └─ 所有権あり（構造体に持たせるならこっち）                      │
│                                                                 │
│  Python/JSとの違い:                                              │
│  Python: str型ひとつ（内部でヒープ確保、GCが管理）               │
│  Rust:   用途に応じて使い分け（GCがないから明示が必要）           │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「なるほど…！構造体のフィールドに持たせるなら、所有権がある`String`じゃないとダメなんですね。」
>
> 👩‍🏫 **レイ**: 「完璧！`&str`を持たせることもできるけど、ライフタイムっていう概念が必要になるから、今は`String`を使うのがベストね。」

### 基本的な構造体

> 👩‍🏫 **レイ**: 「まず、structでデータだけ定義するのよ。」

```rust
struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}

fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };

    println!("Username: {}", user1.username);
}
```

> 👩‍💻 **ユイ**: 「あれ、これだけですか？メソッドはどこですか？」
>
> 👩‍🏫 **レイ**: 「それは後で`impl`ブロックで別に定義するの。まずは構造だけね。」

### 各行を読み解く

> 👩‍🏫 **レイ**: 「1行ずつ見ていくわよ。」

```rust
struct User {
```

> 👩‍🏫 **レイ**: 「`struct` = 構造体を定義するキーワード。`User` = 構造体の名前で、PascalCaseで書くのがRustの慣習よ。」

```rust
    username: String,
    email: String,
```

> 👩‍🏫 **レイ**: 「`フィールド名: 型` の形式。カンマ区切りで複数フィールドを定義するの。**すべてのフィールドに型が必要**よ。」
>
> 👩‍💻 **ユイ**: 「TypeScriptの`interface`と似てますね！」

```rust
let user1 = User {
    email: String::from("user@example.com"),
    ...
};
```

> 👩‍🏫 **レイ**: 「構造体のインスタンスを作成してるの。**すべてのフィールドに値を指定する必要がある**わ。」
>
> 👩‍💻 **ユイ**: 「省略できないんですか？」
>
> 👩‍🏫 **レイ**: 「Rustにはデフォルト値の概念がないからね。でも、`Default`トレイトを使えば似たようなことはできるわよ。後で説明するわ。」

### 比較してみましょう

> 👩‍🏫 **レイ**: 「TypeScriptとPythonと比較してみるわね。」

**TypeScript:**

```typescript
interface User {
    username: string;
    email: string;
    signInCount: number;
    active: boolean;
}

const user1: User = {
    email: "user@example.com",
    username: "someuser",
    active: true,
    signInCount: 1,
};
```

**Python:**

```python
from dataclasses import dataclass

@dataclass
class User:
    username: str
    email: str
    sign_in_count: int
    active: bool

user1 = User(
    email="user@example.com",
    username="someuser",
    active=True,
    sign_in_count=1,
)
```

> 👩‍💻 **ユイ**: 「ほんとに似てますね！」

### ミュータブルな構造体

> 👩‍🏫 **レイ**: 「あと、重要なのは**構造体全体**がミュータブルかイミュータブルかを決めることよ。」

```rust
fn main() {
    // mutなしの場合、すべてのフィールドが変更不可
    let user1 = User { ... };
    // user1.email = ...;  // ❌ エラー

    // mutありの場合、すべてのフィールドが変更可能
    let mut user2 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };

    user2.email = String::from("new@example.com");  // ✅ OK
}
```

> 👩‍💻 **ユイ**: 「あっ、フィールド単位じゃなくて、**インスタンス全体**でmutを決めるんですね。」
>
> 👩‍🏫 **レイ**: 「その通り！TypeScript/Pythonだとプロパティごとに`readonly`/`Final`を指定できるけど、Rustは全体で決めるのよ。」

### フィールド初期化省略記法

> 👩‍🏫 **レイ**: 「便利な省略記法もあるわ。変数名とフィールド名が同じなら省略できるの。」

```rust
fn build_user(email: String, username: String) -> User {
    User {
        email,      // email: email の省略
        username,   // username: username の省略
        active: true,
        sign_in_count: 1,
    }
}
```

> 👩‍💻 **ユイ**: 「あっ、JavaScriptのオブジェクト省略記法と同じですね！」
>
> 👩‍🏫 **レイ**: 「そう。Rustは他の言語のいいとこ取りしてるのよ。」

### 構造体の更新記法

> 👩‍🏫 **レイ**: 「既存のインスタンスから一部だけ変更したい場合は、こうするわ。」

```rust
fn main() {
    let user1 = User {
        email: String::from("user@example.com"),
        username: String::from("someuser"),
        active: true,
        sign_in_count: 1,
    };

    let user2 = User {
        email: String::from("another@example.com"),
        ..user1  // 残りのフィールドはuser1から
    };
}
```

> 👩‍💻 **ユイ**: 「TypeScriptのスプレッド構文みたいですね！」

```typescript
const user2 = { ...user1, email: "another@example.com" };
```

> 👩‍🏫 **レイ**: 「そうね。でも、**所有権に注意**よ。」

```rust
let user2 = User {
    email: String::from("another@example.com"),
    ..user1
};

// user1.usernameはムーブされた
println!("{}", user1.username);  // ❌ エラー！

// Copy型（u64, bool）は使える
println!("{}", user1.active);    // ✅ OK
println!("{}", user1.sign_in_count);  // ✅ OK
```

> 👩‍💻 **ユイ**: 「あっ、String型のフィールドはムーブされるんですね！」
>
> 👩‍🏫 **レイ**: （ホワイトボードに図を描く）「そう。メモリの動きを見てみましょう。」

```
┌─────────────────────────────────────────────────────────────────┐
│  ..user1 のメモリ動作                                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  user1:                       user2:                            │
│  ┌─────────────────┐         ┌─────────────────┐               │
│  │ username ───────│──×      │ username ───────│──┐            │
│  │ email    ───────│──┐      │ email    ───────│──│─→"another" │
│  │ sign_in_count: 1│  │      │ sign_in_count: 1│  │            │
│  │ active: true    │  │      │ active: true    │  │            │
│  └─────────────────┘  │      └─────────────────┘  │            │
│                       │                            ↓            │
│                       └→ "user@example"        "someuser"      │
│                          ↑                                      │
│                          ムーブでuser1.usernameは無効に         │
│                                                                 │
│  u64, boolはCopy型なのでコピーされる                             │
│  StringはCopy型でないのでムーブされる                            │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「なるほど…！Copy型とそうでない型で挙動が違うんですね。」

---

## タプル構造体

> 👩‍🏫 **レイ**: 「フィールド名のない構造体も作れるわよ。」

```rust
struct Color(i32, i32, i32);
struct Point(i32, i32, i32);

fn main() {
    let black = Color(0, 0, 0);
    let origin = Point(0, 0, 0);

    // ColorとPointは別の型（値が同じでも混同できない）
    println!("R: {}", black.0);  // インデックスでアクセス
    println!("G: {}", black.1);
    println!("B: {}", black.2);
}
```

> 👩‍💻 **ユイ**: 「値が同じでも**別の型**なんですね。」
>
> 👩‍🏫 **レイ**: 「そう。これを使った面白いパターンがあるわ。**ニュータイプパターン**って言うの。」

### ニュータイプパターン

```rust
// 単位を型で区別する
struct Meters(f64);
struct Kilometers(f64);

fn travel_distance(meters: Meters) {
    println!("距離: {}m", meters.0);
}

fn main() {
    let distance_m = Meters(1000.0);
    let distance_km = Kilometers(1.0);

    travel_distance(distance_m);   // ✅ OK
    // travel_distance(distance_km);  // ❌ 型が違う！
}
```

> 👩‍💻 **ユイ**: 「おお！メートルとキロメートルを間違えて渡せないんですね！」
>
> 👩‍🏫 **レイ**: 「そう。『メートルとキロメートルを間違えて渡す』バグを**コンパイル時に防止**できるの。型システムの力よ。」
>
> 👩‍💻 **ユイ**: 「すごい…！これ、金額とか時間とかでも使えそうですね。」
>
> 👩‍🏫 **レイ**: 「完璧！実際、本番コードでもよく使われるパターンよ。」

---

## メソッドの実装（impl）— 分離されたメソッド定義

> 👩‍🏫 **レイ**: 「じゃあ、いよいよメソッドを定義するわよ。」

```rust
struct Rectangle {
    width: u32,
    height: u32,
}

impl Rectangle {
    // メソッドを定義
    fn area(&self) -> u32 {
        self.width * self.height
    }
}

fn main() {
    let rect = Rectangle { width: 30, height: 50 };
    println!("Area: {}", rect.area());
}
```

> 👩‍💻 **ユイ**: 「`impl`ブロックでメソッドを定義するんですね！クラスの中に書くのと何が違うんですか？」
>
> 👩‍🏫 **レイ**: 「いい質問ね。Rustは**データとロジックを分離**するの。structは『どんなデータか』、implは『何ができるか』を定義するのよ。」

### `impl`ブロックを読み解く

```rust
impl Rectangle {
```

> 👩‍🏫 **レイ**: 「`impl` = implementation（実装）の略。`Rectangle` = メソッドを追加する対象の型よ。」
>
> 👩‍💻 **ユイ**: 「1つの型に複数の`impl`ブロック書けるんですか？」
>
> 👩‍🏫 **レイ**: 「書けるわ！関連する機能ごとに`impl`ブロックを分けたりするの。」

```rust
    fn area(&self) -> u32 {
```

> 👩‍🏫 **レイ**: 「`fn area` = メソッド名。`&self` = **このインスタンスへの不変参照**。`-> u32` = 戻り値の型よ。」
>
> 👩‍💻 **ユイ**: 「`&self`って何ですか？」
>
> 👩‍🏫 **レイ**: 「これが**超重要**なのよ。`self`には3つの形態があるの。」

### `self`の3つの形態 — 重要！

> 👩‍🏫 **レイ**: （ホワイトボードに大きく書く）

```rust
impl Rectangle {
    // 1. &self — 不変借用（読み取り専用）
    fn area(&self) -> u32 {
        self.width * self.height
    }

    // 2. &mut self — 可変借用（変更可能）
    fn double(&mut self) {
        self.width *= 2;
        self.height *= 2;
    }

    // 3. self — 所有権を取得（消費）
    fn destroy(self) -> (u32, u32) {
        (self.width, self.height)
    }
}
```

> 👩‍🏫 **レイ**: 「整理するわね。」

```
┌─────────────────────────────────────────────────────────────────┐
│  self の3形態                                                    │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  &self（不変借用）                                               │
│  ├─ インスタンスを読み取るだけ                                    │
│  ├─ 呼び出し後もインスタンスを使える                              │
│  └─ 最も一般的                                                   │
│                                                                 │
│  &mut self（可変借用）                                           │
│  ├─ インスタンスを変更できる                                     │
│  ├─ 呼び出し後もインスタンスを使える                              │
│  └─ 例: vec.push(), string.push_str()                           │
│                                                                 │
│  self（所有権取得）                                              │
│  ├─ インスタンスの所有権をメソッドに移動                          │
│  ├─ 呼び出し後はインスタンスを使えない                            │
│  └─ 例: 変換メソッド、ビルダーパターン                            │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「えっと…`&self`は読むだけ、`&mut self`は変更する、`self`は消費する…？」
>
> 👩‍🏫 **レイ**: 「完璧！実際に使ってみましょう。」

**使用例:**

```rust
fn main() {
    let mut rect = Rectangle { width: 30, height: 50 };

    // &self — 読み取り
    println!("Area: {}", rect.area());
    println!("Still usable: {:?}", rect);  // ✅ OK

    // &mut self — 変更
    rect.double();
    println!("Doubled: {:?}", rect);  // ✅ OK

    // self — 消費
    let dimensions = rect.destroy();
    // println!("{:?}", rect);  // ❌ rectはムーブされた
    println!("Dimensions: {:?}", dimensions);
}
```

> 👩‍💻 **ユイ**: 「あっ、`destroy()`を呼んだら`rect`が使えなくなった！」
>
> 👩‍🏫 **レイ**: 「そう。所有権がメソッドに移ったからね。これで『すでに破棄したインスタンスを使う』バグを防げるの。」

**比較:**

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| 読み取りメソッド | `def method(self)` | `method()` | `fn method(&self)` |
| 変更メソッド | `def method(self)` | `method()` | `fn method(&mut self)` |
| 消費メソッド | — | — | `fn method(self)` |

> 👩‍💻 **ユイ**: 「Python/TypeScriptだと区別ないんですね…。」
>
> 👩‍🏫 **レイ**: 「そう。Rustは**コンパイラが使い方を強制**してくれるの。だから安全なのよ。」

### 関連関数（Associated Functions）— 静的メソッド

> 👩‍🏫 **レイ**: 「`self`を取らない関数も`impl`ブロックに書けるわ。これを**関連関数**って呼ぶの。」

```rust
impl Rectangle {
    // 関連関数（selfなし）
    fn new(width: u32, height: u32) -> Self {
        Self { width, height }
    }

    fn square(size: u32) -> Self {
        Self { width: size, height: size }
    }
}

fn main() {
    // 関連関数は :: で呼び出す
    let rect = Rectangle::new(30, 50);
    let square = Rectangle::square(10);
}
```

> 👩‍💻 **ユイ**: 「`Self`と`::`って何ですか？」
>
> 👩‍🏫 **レイ**: 「`Self` = 『この`impl`ブロックの対象の型』を指すの。ここでは`Self` = `Rectangle`よ。」

### `Self`と`::`を読み解く

> 👩‍🏫 **レイ**: （ホワイトボードに書く）

```
┌─────────────────────────────────────────────────────────────────┐
│  . と :: の使い分け                                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  rect.area()           ← ドット: インスタンスメソッド呼び出し    │
│                          （selfを取る）                          │
│                                                                 │
│  Rectangle::new(30, 50) ← ダブルコロン: 関連関数呼び出し         │
│                          （selfを取らない）                      │
│                                                                 │
│  同様に:                                                         │
│  String::from("hello")  ← 関連関数                              │
│  Vec::new()             ← 関連関数                              │
│  Option::Some(5)        ← 列挙型のバリアント                     │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「なるほど！`String::from()`も関連関数だったんですね。」
>
> 👩‍🏫 **レイ**: 「その通り！」

---

## 列挙型（Enum）— データを持てる強力な型

> 👩‍🏫 **レイ**: 「次は**列挙型（enum）**よ。Rustのenumは**めちゃくちゃ強力**なの。」
>
> 👩‍💻 **ユイ**: 「enumって、定数の集合ですよね？TypeScriptにもありますけど…」
>
> 👩‍🏫 **レイ**: 「Rustのenumは違うわ。**各バリアントがデータを持てる**のよ。」

### 基本的な列挙型

```rust
enum Direction {
    Up,
    Down,
    Left,
    Right,
}

fn main() {
    let dir = Direction::Up;

    match dir {
        Direction::Up => println!("上"),
        Direction::Down => println!("下"),
        Direction::Left => println!("左"),
        Direction::Right => println!("右"),
    }
}
```

> 👩‍💻 **ユイ**: 「これはTypeScriptと同じですね。」

### データを持つ列挙型 — Rust独特の強力な機能

> 👩‍🏫 **レイ**: 「じゃあ、これはどう？」

```rust
enum Message {
    Quit,                       // データなし
    Move { x: i32, y: i32 },    // 名前付きフィールド（構造体のような形）
    Write(String),              // String型のデータを1つ持つ
    ChangeColor(i32, i32, i32), // 3つの値を持つ
}

fn main() {
    let msg1 = Message::Quit;
    let msg2 = Message::Move { x: 10, y: 20 };
    let msg3 = Message::Write(String::from("hello"));
    let msg4 = Message::ChangeColor(255, 0, 0);
}
```

> 👩‍💻 **ユイ**: 「えっ！？各バリアントが**違うデータ**を持ってる！？」
>
> 👩‍🏫 **レイ**: 「そう！これがRustのenumの真骨頂よ。実は、これ**タグ付きユニオン（tagged union）**って呼ばれる概念なの。」
>
> 👩‍💻 **ユイ**: 「タグ付きユニオン…？」
>
> 👩‍🏫 **レイ**: 「メモリレベルで見ると理解しやすいわ。ホワイトボードに描くわね。」

### enumのメモリレイアウト — 実はこうなっている

```
┌─────────────────────────────────────────────────────────────────┐
│  Message enumのメモリレイアウト                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  各バリアントのサイズ:                                           │
│  - Quit:         0 bytes（データなし）                           │
│  - Move:         8 bytes（i32 × 2）                              │
│  - Write:       24 bytes（Stringはptr+len+cap）                  │
│  - ChangeColor: 12 bytes（i32 × 3）                              │
│                                                                 │
│  enum全体のサイズ = 判別タグ + 最大バリアントのサイズ              │
│                  = 1 byte + 24 bytes = 25 bytes                  │
│                  （パディングで32 bytesになることも）              │
│                                                                 │
│  ┌────────┬────────────────────────────────────┐               │
│  │  tag   │           data                     │               │
│  │ 1 byte │         24 bytes                   │               │
│  └────────┴────────────────────────────────────┘               │
│                                                                 │
│  tag = 0: Quit                                                  │
│  tag = 1: Move { x, y }                                         │
│  tag = 2: Write(String)                                         │
│  tag = 3: ChangeColor(r, g, b)                                  │
│                                                                 │
│  → 実行時に「どのバリアントか」をtagで判別                        │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「あっ、なるほど！各バリアントが違うデータを持てるけど、メモリ上は**一番大きいバリアントのサイズ**を確保してるんですね！」
>
> 👩‍🏫 **レイ**: 「そう！だから`Quit`を格納しても24バイトの領域を使うの。ちょっとムダに見えるけど、実行時の型チェックなしでパフォーマンスが出せるのよ。」
>
> 👩‍💻 **ユイ**: 「matchでパターンマッチするとき、このtagを見てるってことですか？」
>
> 👩‍🏫 **レイ**: 「完璧！コンパイラがtagに基づいて分岐するコードを生成してくれるの。だから超高速なのよ。」

### 各バリアントの形式

> 👩‍🏫 **レイ**: （ホワイトボードに書く）

```
┌─────────────────────────────────────────────────────────────────┐
│  列挙型バリアントの形式                                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  1. ユニット型（データなし）                                     │
│     Quit                                                        │
│                                                                 │
│  2. 構造体型（名前付きフィールド）                               │
│     Move { x: i32, y: i32 }                                     │
│                                                                 │
│  3. タプル型（位置でアクセス）                                   │
│     Write(String)                                               │
│     ChangeColor(i32, i32, i32)                                  │
│                                                                 │
│  これらを1つのenum内に混在させられる！                           │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「すごい…！TypeScriptだとどう書くんですか？」
>
> 👩‍🏫 **レイ**: 「TypeScriptだと、判別可能なユニオン型で表現するわ。」

**TypeScript:**

```typescript
// TypeScriptでは判別可能なユニオン型で表現
type Message =
    | { type: "Quit" }
    | { type: "Move"; x: number; y: number }
    | { type: "Write"; content: string }
    | { type: "ChangeColor"; r: number; g: number; b: number };

// Rustの方がシンプルに書ける
```

> 👩‍💻 **ユイ**: 「確かに…Rustの方がシンプルですね！」

### 列挙型にもメソッドを実装できる

> 👩‍🏫 **レイ**: 「enumにも`impl`でメソッドを追加できるわよ。」

```rust
impl Message {
    fn call(&self) {
        match self {
            Message::Quit => println!("Quit"),
            Message::Move { x, y } => println!("Move to ({}, {})", x, y),
            Message::Write(text) => println!("Write: {}", text),
            Message::ChangeColor(r, g, b) => println!("Color: ({}, {}, {})", r, g, b),
        }
    }
}

fn main() {
    let msg = Message::Write(String::from("hello"));
    msg.call();  // Write: hello
}
```

> 👩‍💻 **ユイ**: 「enumにもメソッド追加できるんですね！」

---

## パターンマッチ（match）— 網羅性チェック付きswitch

> 👩‍🏫 **レイ**: 「`match`は`switch`文の強力版よ。**すべてのケースを網羅**しなければコンパイルエラーになるの。」

```rust
enum Coin {
    Penny,
    Nickel,
    Dime,
    Quarter,
}

fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        Coin::Dime => 10,
        Coin::Quarter => 25,
    }
}
```

> 👩‍💻 **ユイ**: 「普通のswitch文じゃないですか？」
>
> 👩‍🏫 **レイ**: 「じゃあ、これは？」

### なぜ網羅性チェックが重要か

```rust
fn value_in_cents(coin: Coin) -> u32 {
    match coin {
        Coin::Penny => 1,
        Coin::Nickel => 5,
        // Dime と Quarter を忘れている
    }
}
// ❌ コンパイルエラー: patterns `Dime` and `Quarter` not covered
```

> 👩‍💻 **ユイ**: 「おお！ケースを忘れたらエラーになるんですね！」
>
> 👩‍🏫 **レイ**: 「そう！この網羅性チェック、実務で何度も助けられてるわ。こんな話があってね…」

### 実務で助かった話 — 決済手段の追加

> 👩‍🏫 **レイ**: 「以前、決済システムに新しい支払い方法を追加したことがあるの。」

```rust
enum PaymentMethod {
    CreditCard { card_number: String },
    BankTransfer { account_id: String },
    // 新しく追加 ↓
    Cryptocurrency { wallet_address: String },
}
```

> 👩‍🏫 **レイ**: 「この変更をしたら、コンパイラが**5箇所**の`match`文でエラーを出してくれたの。『Cryptocurrencyのケースがない』って。」
>
> 👩‍💻 **ユイ**: 「全部の分岐を見つけてくれるんですね！」
>
> 👩‍🏫 **レイ**: 「そう。TypeScriptやPythonだと、grep検索してケースを探すか、本番で『想定外のエラー』になるまで気づかないことがあるの。」

**TypeScript:**

```typescript
// TypeScriptは設定次第でチェックが緩い
function valueInCents(coin: Coin): number {
    switch (coin) {
        case Coin.Penny: return 1;
        case Coin.Nickel: return 5;
        // Dime, Quarterを忘れてもエラーにならない場合がある
    }
}
```

> 👩‍💻 **ユイ**: 「確かに…！Rustの方が安全ですね。」
>
> 👩‍🏫 **レイ**: 「特にenumに新しいバリアントを追加したとき、**修正が必要な箇所をコンパイラが全部教えてくれる**のがめちゃくちゃ便利なのよ。」

### パターンマッチでの値の取り出し

```rust
fn process_message(msg: Message) {
    match msg {
        Message::Quit => {
            println!("終了します");
        }
        Message::Move { x, y } => {
            // 名前付きフィールドを分解
            println!("({}, {})に移動", x, y);
        }
        Message::Write(text) => {
            // タプルの値を取り出し
            println!("メッセージ: {}", text);
        }
        Message::ChangeColor(r, g, b) => {
            // 複数の値を取り出し
            println!("色をRGB({}, {}, {})に変更", r, g, b);
        }
    }
}
```

> 👩‍💻 **ユイ**: 「パターンマッチで値を取り出せるんですね！便利…！」

### `_` プレースホルダ — その他すべて

```rust
fn main() {
    let value = 3;

    match value {
        1 => println!("one"),
        2 => println!("two"),
        _ => println!("other"),  // それ以外すべて
    }
}
```

> 👩‍🏫 **レイ**: 「`_`は『その他すべて』を表すわ。でも、enumで使うと網羅性チェックが甘くなるから、慎重に使ってね。」

---

## Option型 — nullの代わり

> 👩‍🏫 **レイ**: 「さて、いよいよ**Option型**よ。これが**めちゃくちゃ重要**なの。」
>
> 👩‍💻 **ユイ**: 「Option…？」
>
> 👩‍🏫 **レイ**: 「Rustには`null`がないの。」
>
> 👩‍💻 **ユイ**: 「えっ！？nullないんですか！？じゃあ『値がない』をどう表すんですか？」
>
> 👩‍🏫 **レイ**: 「代わりに`Option<T>`を使うの。これもenumの一種よ。」

```rust
enum Option<T> {
    Some(T),  // 値がある
    None,     // 値がない
}
```

### なぜnullを廃止したのか — 10億ドルの間違い

> 👩‍🏫 **レイ**: 「nullって、1965年にトニー・ホーアがALGOL言語に導入したのよ。でも、彼自身がこう言ってるの。」

```
┌─────────────────────────────────────────────────────────────────┐
│  "I call it my billion-dollar mistake."                         │
│  「私はそれを10億ドルの過ちと呼んでいる」                         │
│                          — トニー・ホーア (2009年)               │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  nullが引き起こす問題:                                           │
│                                                                 │
│  1. NullPointerException / TypeError                            │
│     - 実行時エラーの最大の原因                                   │
│     - 「なぜnullになったか」のデバッグが困難                      │
│                                                                 │
│  2. 型システムの穴                                               │
│     - Java: String型でもnullになりうる                           │
│     - 「Stringを受け取る関数」がnullで壊れる                     │
│                                                                 │
│  3. 暗黙の前提                                                   │
│     - 「この変数はnullにならないはず」という仮定                  │
│     - ドキュメントに書くか、祈るしかない                         │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「そういえば、JavaScriptでも`Cannot read property 'x' of null`とかよく見ますね…」
>
> 👩‍🏫 **レイ**: 「でしょ？Rustは**型システムで『値がないかもしれない』を表現**することで、この問題を根本的に解決したの。」

### なぜOptionが必要か

> 👩‍🏫 **レイ**: （ホワイトボードに書く）

```
┌─────────────────────────────────────────────────────────────────┐
│  null vs Option                                                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  JavaScript/Python (null/None):                                 │
│  - どの変数でもnull/Noneになりうる                               │
│  - 「値がないかも」が型に現れない                                │
│  - 実行時にNullPointerException/TypeError                       │
│                                                                 │
│  Rust (Option<T>):                                              │
│  - Option<T>型の変数だけが「値がないかも」                       │
│  - 「値がないかも」が型に明示される                              │
│  - Noneの処理を忘れるとコンパイルエラー                          │
│                                                                 │
│  例:                                                             │
│  String         → 必ず文字列がある                               │
│  Option<String> → 文字列があるかもしれないし、ないかもしれない   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

> 👩‍💻 **ユイ**: 「型で『値がないかもしれない』を表現するんですね！」
>
> 👩‍🏫 **レイ**: 「その通り！だから『nullチェック忘れ』が**原理的に起きない**の。」

### Optionの使用例

```rust
fn main() {
    let some_number: Option<i32> = Some(5);
    let no_number: Option<i32> = None;

    // Optionから値を取り出すにはパターンマッチ
    match some_number {
        Some(n) => println!("Number: {}", n),
        None => println!("No number"),
    }
}
```

> 👩‍💻 **ユイ**: 「必ずパターンマッチで処理するから、Noneの場合を忘れないんですね！」
>
> 👩‍🏫 **レイ**: 「完璧！」

### 便利なOptionメソッド

> 👩‍🏫 **レイ**: 「Optionには便利なメソッドがたくさんあるわ。」

```rust
fn main() {
    let x: Option<i32> = Some(5);
    let none: Option<i32> = None;

    // unwrap: Someなら中身を返す、Noneならpanic
    let value = x.unwrap();  // 5
    // let crash = none.unwrap();  // ⚠️ panic!

    // expect: unwrap + エラーメッセージ付き
    let value = x.expect("値がありません");

    // unwrap_or: Noneの場合のデフォルト値
    let value = none.unwrap_or(0);  // 0

    // map: Someの中身を変換
    let doubled: Option<i32> = x.map(|n| n * 2);  // Some(10)
    let doubled_none: Option<i32> = none.map(|n| n * 2);  // None

    // is_some / is_none: 判定
    if x.is_some() {
        println!("値があります");
    }
}
```

> 👩‍💻 **ユイ**: 「めっちゃ便利ですね！`map`とかJavaScriptの配列メソッドみたいですね。」

### if let — 簡潔なパターンマッチ

> 👩‍🏫 **レイ**: 「1つのパターンだけ処理したい場合は、`if let`が便利よ。」

```rust
fn main() {
    let some_value: Option<i32> = Some(3);

    // matchを使う場合
    match some_value {
        Some(v) => println!("Value: {}", v),
        None => {},  // Noneは何もしない（冗長）
    }

    // if letを使う場合（簡潔）
    if let Some(v) = some_value {
        println!("Value: {}", v);
    }

    // elseも書ける
    if let Some(v) = some_value {
        println!("Value: {}", v);
    } else {
        println!("値がありません");
    }
}
```

> 👩‍💻 **ユイ**: 「短く書けますね！」

---

## 派生マクロ（derive）— 自動実装

> 👩‍🏫 **レイ**: 「最後に、**derive**マクロよ。よく使う機能を自動実装してくれるの。」

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: i32,
    y: i32,
}

fn main() {
    let p1 = Point { x: 1, y: 2 };
    let p2 = p1.clone();  // Clone

    println!("{:?}", p1);        // Debug: Point { x: 1, y: 2 }
    println!("{}", p1 == p2);    // PartialEq: true
}
```

> 👩‍💻 **ユイ**: 「`#[derive(...)]`って何ですか？」
>
> 👩‍🏫 **レイ**: 「これは**アトリビュート**って言って、コンパイラに『この機能を自動実装して』って指示するの。」

### よく使うderiveマクロ

| マクロ | 機能 | 使用例 |
|-------|-----|--------|
| `Debug` | `{:?}`でデバッグ出力 | `println!("{:?}", value)` |
| `Clone` | `.clone()`でコピー | `let copy = value.clone()` |
| `Copy` | 暗黙的コピー（小さい値向け） | 代入時に自動コピー |
| `PartialEq` | `==`で比較 | `if a == b { }` |
| `Eq` | 完全な等価性（NaN等を含まない） | HashMapのキーに必要 |
| `Hash` | ハッシュ値を計算 | HashSetやHashMapのキーに必要 |
| `Default` | デフォルト値を生成 | `Default::default()` |

> 👩‍💻 **ユイ**: 「めっちゃ便利ですね！自分で実装しなくていいんですね。」
>
> 👩‍🏫 **レイ**: 「そう。Rustはボイラープレート（定型コード）を減らす工夫がたくさんあるのよ。」

---

## まとめ — 今日わかったこと

> 👩‍🏫 **レイ**: 「じゃあ、今日学んだことを整理してみて。」
>
> 👩‍💻 **ユイ**: 「はい！えっと…」

（ユイがノートを見返しながら）

> 👩‍💻 **ユイ**: 「まず、Rustにはclassがない。最初は『え！？』って思ったけど、これは**継承の問題を避けるため**なんですよね。1994年にGoFが『継承より合成を』って言ってたのに、言語レベルで解決したのがRustだって。」
>
> 👩‍🏫 **レイ**: 「完璧。」
>
> 👩‍💻 **ユイ**: 「で、classの代わりに**structでデータを定義**して、**implでメソッドを追加**する。分離されてるけど、むしろ見通しが良いかも。」
>
> 👩‍🏫 **レイ**: 「うん、大きなプロジェクトだと特にね。」
>
> 👩‍💻 **ユイ**: 「あと、メソッドの`&self`、`&mut self`、`self`の違いは**所有権と直結**してるんですよね。読むだけ、変更する、消費する。これを間違えるとコンパイルエラーになるから、バグを防げる。」
>
> 👩‍🏫 **レイ**: 「そう！Python/TypeScriptだと全部`self`だから、間違った使い方しても気づけないことがあるのよね。」
>
> 👩‍💻 **ユイ**: 「一番衝撃だったのは**enumがデータを持てる**こと！TypeScriptだと判別可能なユニオン型を使うところを、Rustは言語機能としてサポートしてて、しかもメモリレイアウトまで見たら『タグ+データ』って超効率的に実装されてて…」
>
> 👩‍🏫 **レイ**: 「（嬉しそうに）メモリの話まで理解してくれてる…！」
>
> 👩‍💻 **ユイ**: 「それと**Option型**！nullがないって最初は不便だと思ったけど、『値がないかもしれない』を型で表現することで、**nullチェック忘れが原理的に起きない**んですよね。10億ドルの間違いを防いでくれてる。」
>
> 👩‍🏫 **レイ**: 「パーフェクト！」
>
> 👩‍💻 **ユイ**: 「あと先輩の**継承地獄の話**が印象的でした…。BaseEntityにフィールド追加したらテスト47個落ちたって…」
>
> 👩‍🏫 **レイ**: （苦笑い）「あれは今でもトラウマよ…。でもRustならあんなこと起きないから安心してね。」

| 概念 | Python | TypeScript | Rust |
|-----|--------|------------|------|
| データ構造 | `class`, `@dataclass` | `interface`, `class` | `struct` |
| メソッド | クラス内に定義 | クラス内に定義 | `impl`ブロック（分離） |
| 列挙型 | `Enum` | `enum`, ユニオン型 | `enum`（データ付き可） |
| null表現 | `None` | `null`/`undefined` | `Option<T>` |
| パターンマッチ | `match`文（3.10+） | `switch`（弱い） | `match`（網羅性チェック） |

> 👩‍💻 **ユイ**: 「最初は『クラスがない』って聞いて不安でしたけど、`struct`と`impl`で十分ですね！」
>
> 👩‍🏫 **レイ**: 「でしょ？むしろ、データとロジックが分離されてる方が見通しがいいのよ。」
>
> 👩‍💻 **ユイ**: 「enumがデータを持てるのも驚きました！Option型も、最初は面倒だと思ったけど、nullチェック忘れがなくなるのは安心ですね。」
>
> 👩‍🏫 **レイ**: 「その調子！特に`&self`, `&mut self`, `self`の使い分けは、Rustの所有権システムの核心だから、しっかり覚えておいてね。」

---

## ポイント

- **struct + impl** = クラスの代わり（データとメソッドが分離）
- **&self / &mut self / self** = メソッドがインスタンスをどう扱うか
- **Self** = impl対象の型を指す
- **::** = 関連関数やバリアントのアクセス
- **enum** = データを持てる強力な列挙型
- **Option** = null安全（値がないかもを型で表現）
- **match** = 網羅性チェック付きパターンマッチ
- **derive** = よく使う機能を自動実装

---

## 次のステップ

> 👩‍🏫 **レイ**: 「次は**エラーハンドリング**よ。`Result`型と`?`演算子を使った、try-catchとは違うアプローチを学ぶわ。」
>
> 👩‍💻 **ユイ**: 「Option型で学んだパターンマッチが活きそうですね！」
>
> 👩‍🏫 **レイ**: 「その通り！楽しみにしててね。」

[Chapter 04: エラーハンドリング](04-error-handling-dialogue.md) では、Rustのエラー処理パターンを学びます。`Result`型と`?`演算子を使った、try-catchとは異なるアプローチを解説します。Option型で学んだパターンマッチの知識が活きます。
