# Chapter 06: コレクション

> **この章で学ぶこと**: Vec、HashMap、イテレータ、クロージャ — データの集合を扱う方法

---

## コレクション概要

Rustの主要なコレクション型：

| 型 | 説明 | Python相当 | JS/TS相当 |
|---|------|-----------|----------|
| `Vec<T>` | 可変長配列 | `list` | `Array` |
| `HashMap<K, V>` | キーバリュー | `dict` | `Map`/`Object` |
| `HashSet<T>` | 集合 | `set` | `Set` |
| `VecDeque<T>` | 両端キュー | `collections.deque` | — |
| `BTreeMap<K, V>` | 順序付きマップ | — | — |

---

## Vec<T> — 可変長配列

最もよく使うコレクションです。

### 作成と初期化

```rust
fn main() {
    // 空のVecを作成
    let mut v1: Vec<i32> = Vec::new();
    
    // マクロで初期化
    let v2 = vec![1, 2, 3, 4, 5];
    
    // 容量を事前確保
    let mut v3: Vec<i32> = Vec::with_capacity(100);
}
```

🔄 **比較**:
```python
# Python
v1 = []
v2 = [1, 2, 3, 4, 5]
```

```typescript
// TypeScript
const v1: number[] = [];
const v2 = [1, 2, 3, 4, 5];
```

### 要素の追加・削除

```rust
fn main() {
    let mut v = vec![1, 2, 3];
    
    v.push(4);           // 末尾に追加 → [1, 2, 3, 4]
    v.pop();             // 末尾を削除 → [1, 2, 3]
    v.insert(1, 10);     // 指定位置に挿入 → [1, 10, 2, 3]
    v.remove(1);         // 指定位置を削除 → [1, 2, 3]
    
    v.extend([4, 5, 6]); // 複数追加 → [1, 2, 3, 4, 5, 6]
    v.clear();           // 全削除 → []
}
```

### 要素へのアクセス

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // インデックスアクセス（範囲外でpanic）
    let third = v[2];  // 3
    
    // getメソッド（Optionを返す、安全）
    let third: Option<&i32> = v.get(2);      // Some(&3)
    let tenth: Option<&i32> = v.get(10);     // None
    
    // 最初と最後
    let first = v.first();  // Some(&1)
    let last = v.last();    // Some(&5)
}
```

⚠️ **注意**: インデックスアクセス`v[i]`は範囲外でpanicします。安全に処理したい場合は`get()`を使いましょう。

### スライス

Vecの一部を参照として取り出せます。

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    let slice: &[i32] = &v[1..4];  // [2, 3, 4]
    let all: &[i32] = &v[..];      // [1, 2, 3, 4, 5]
    let from_2: &[i32] = &v[2..];  // [3, 4, 5]
    let to_3: &[i32] = &v[..3];    // [1, 2, 3]
}
```

🔄 **比較（Python）**:
```python
v = [1, 2, 3, 4, 5]
slice = v[1:4]  # [2, 3, 4]（コピーが作成される）
```

Rustのスライスは**参照**なのでコピーは発生しません。

---

## HashMap<K, V> — キーバリュー

### 作成と操作

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    
    // 挿入
    scores.insert(String::from("Blue"), 10);
    scores.insert(String::from("Yellow"), 50);
    
    // 取得
    let blue_score = scores.get("Blue");  // Some(&10)
    
    // 存在確認
    if scores.contains_key("Blue") {
        println!("Blueチームのスコアがあります");
    }
    
    // 削除
    scores.remove("Blue");
}
```

🔄 **比較**:
```python
# Python
scores = {}
scores["Blue"] = 10
blue_score = scores.get("Blue")  # 10 or None
```

```typescript
// TypeScript
const scores = new Map<string, number>();
scores.set("Blue", 10);
const blueScore = scores.get("Blue");
```

### Entry API — 条件付き挿入

```rust
use std::collections::HashMap;

fn main() {
    let mut scores = HashMap::new();
    
    // キーがなければ挿入
    scores.entry(String::from("Blue")).or_insert(50);
    scores.entry(String::from("Blue")).or_insert(100);  // 既にあるので無視
    
    println!("{:?}", scores);  // {"Blue": 50}
    
    // 既存値を更新
    let count = scores.entry(String::from("Blue")).or_insert(0);
    *count += 10;  // 50 + 10 = 60
}
```

### ワードカウントの例

```rust
use std::collections::HashMap;

fn main() {
    let text = "hello world wonderful world";
    let mut word_count = HashMap::new();
    
    for word in text.split_whitespace() {
        let count = word_count.entry(word).or_insert(0);
        *count += 1;
    }
    
    println!("{:?}", word_count);
    // {"hello": 1, "world": 2, "wonderful": 1}
}
```

🔄 **比較（Python）**:
```python
from collections import Counter
word_count = Counter(text.split())
```

---

## イテレータ

Rustのイテレータは**遅延評価**で、チェーンして使います。

### 基本的な使い方

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // forループ（暗黙的にイテレータを使用）
    for x in &v {
        println!("{}", x);
    }
    
    // 明示的にイテレータを取得
    let mut iter = v.iter();
    println!("{:?}", iter.next());  // Some(&1)
    println!("{:?}", iter.next());  // Some(&2)
}
```

### 3種類のイテレータ

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // 不変参照（&T）
    for x in v.iter() { }
    for x in &v { }  // 同じ意味
    
    // 可変参照（&mut T）
    let mut v = vec![1, 2, 3];
    for x in v.iter_mut() {
        *x *= 2;
    }
    
    // 所有権を取得（T）
    for x in v.into_iter() { }
    for x in v { }  // 同じ意味（vはムーブされる）
}
```

---

## イテレータアダプタ

イテレータを変換するメソッドです。**遅延評価**なので、最後に`collect()`等で消費するまで実行されません。

### map — 変換

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    let doubled: Vec<i32> = v.iter()
        .map(|x| x * 2)
        .collect();
    
    println!("{:?}", doubled);  // [2, 4, 6]
}
```

🔄 **比較**:
```python
# Python
doubled = [x * 2 for x in v]
doubled = list(map(lambda x: x * 2, v))
```

```typescript
// TypeScript
const doubled = v.map(x => x * 2);
```

### filter — 絞り込み

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5, 6];
    
    let evens: Vec<i32> = v.iter()
        .filter(|x| *x % 2 == 0)
        .copied()  // &i32 → i32
        .collect();
    
    println!("{:?}", evens);  // [2, 4, 6]
}
```

### filter_map — フィルタ + 変換

```rust
fn main() {
    let strings = vec!["1", "two", "3", "four", "5"];
    
    let numbers: Vec<i32> = strings.iter()
        .filter_map(|s| s.parse().ok())  // パース成功したものだけ
        .collect();
    
    println!("{:?}", numbers);  // [1, 3, 5]
}
```

### fold / reduce — 畳み込み

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // fold: 初期値あり
    let sum: i32 = v.iter().fold(0, |acc, x| acc + x);
    println!("Sum: {}", sum);  // 15
    
    // sum: 合計の専用メソッド
    let sum: i32 = v.iter().sum();
    
    // product: 積の専用メソッド
    let product: i32 = v.iter().product();  // 120
}
```

🔄 **比較（JavaScript）**:
```javascript
const sum = v.reduce((acc, x) => acc + x, 0);
```

### その他の便利なメソッド

```rust
fn main() {
    let v = vec![1, 2, 3, 4, 5];
    
    // take: 最初のn個
    let first_three: Vec<_> = v.iter().take(3).collect();  // [1, 2, 3]
    
    // skip: 最初のn個をスキップ
    let after_two: Vec<_> = v.iter().skip(2).collect();  // [3, 4, 5]
    
    // enumerate: インデックス付き
    for (i, x) in v.iter().enumerate() {
        println!("{}: {}", i, x);
    }
    
    // zip: 2つのイテレータを結合
    let a = vec![1, 2, 3];
    let b = vec!["a", "b", "c"];
    let zipped: Vec<_> = a.iter().zip(b.iter()).collect();
    // [(1, "a"), (2, "b"), (3, "c")]
    
    // find: 条件に合う最初の要素
    let first_even = v.iter().find(|x| *x % 2 == 0);  // Some(&2)
    
    // any / all: 条件判定
    let has_even = v.iter().any(|x| x % 2 == 0);  // true
    let all_positive = v.iter().all(|x| *x > 0);  // true
    
    // count: 要素数
    let count = v.iter().filter(|x| *x % 2 == 0).count();  // 2
}
```

---

## クロージャ

クロージャは**環境をキャプチャできる無名関数**です。

```rust
fn main() {
    let x = 4;
    
    // クロージャ（xをキャプチャ）
    let add_x = |n| n + x;
    
    println!("{}", add_x(5));  // 9
}
```

🔄 **比較**:
```python
# Python
x = 4
add_x = lambda n: n + x
```

```typescript
// TypeScript
const x = 4;
const addX = (n: number) => n + x;
```

### クロージャの型推論

```rust
fn main() {
    // 型注釈なし（推論される）
    let add = |a, b| a + b;
    
    // 型注釈あり
    let add: fn(i32, i32) -> i32 = |a, b| a + b;
    
    // 複数行
    let complex = |x| {
        let y = x * 2;
        y + 1
    };
}
```

### キャプチャの種類

クロージャが環境をどう捕捉するかは3種類あります：

```rust
fn main() {
    let s = String::from("hello");
    
    // 1. 不変借用（&T）
    let print = || println!("{}", s);
    print();
    println!("{}", s);  // ✅ まだ使える
    
    // 2. 可変借用（&mut T）
    let mut s = String::from("hello");
    let mut append = || s.push_str(" world");
    append();
    
    // 3. ムーブ（T）
    let s = String::from("hello");
    let consume = move || println!("{}", s);
    consume();
    // println!("{}", s);  // ❌ sはムーブされた
}
```

`move`キーワードで強制的に所有権を取得できます（スレッドに渡す場合などに必要）。

---

## 実践パターン

### パターン1: チェーンで処理

```rust
struct User {
    name: String,
    age: u32,
    active: bool,
}

fn get_active_adult_names(users: &[User]) -> Vec<String> {
    users.iter()
        .filter(|u| u.active && u.age >= 18)
        .map(|u| u.name.clone())
        .collect()
}
```

🔄 **比較（Python）**:
```python
def get_active_adult_names(users):
    return [u.name for u in users if u.active and u.age >= 18]
```

### パターン2: グルーピング

```rust
use std::collections::HashMap;

fn group_by_age(users: &[User]) -> HashMap<u32, Vec<&User>> {
    let mut groups: HashMap<u32, Vec<&User>> = HashMap::new();
    
    for user in users {
        groups.entry(user.age).or_default().push(user);
    }
    
    groups
}
```

### パターン3: 最大・最小

```rust
fn main() {
    let numbers = vec![3, 1, 4, 1, 5, 9, 2, 6];
    
    let max = numbers.iter().max();           // Some(&9)
    let min = numbers.iter().min();           // Some(&1)
    
    // 条件付き最大
    let max_even = numbers.iter()
        .filter(|x| *x % 2 == 0)
        .max();  // Some(&6)
    
    // カスタム比較
    let users = vec![
        User { name: "Alice".into(), age: 30, active: true },
        User { name: "Bob".into(), age: 25, active: true },
    ];
    let oldest = users.iter().max_by_key(|u| u.age);
}
```

---

## パフォーマンスの考慮

### イテレータは高速

Rustのイテレータは**ゼロコスト抽象化**です。forループと同等の速度で動作します。

```rust
// この2つは同じ速度
let sum1: i32 = (0..1000).sum();

let mut sum2 = 0;
for i in 0..1000 {
    sum2 += i;
}
```

### collect時の型ヒント

```rust
fn main() {
    let v = vec![1, 2, 3];
    
    // 型を明示
    let doubled: Vec<i32> = v.iter().map(|x| x * 2).collect();
    
    // ターボフィッシュ記法
    let doubled = v.iter().map(|x| x * 2).collect::<Vec<i32>>();
    
    // 部分的な型ヒント
    let doubled: Vec<_> = v.iter().map(|x| x * 2).collect();
}
```

---

## まとめ

| 概念 | Python | JavaScript | Rust |
|-----|--------|------------|------|
| 可変長配列 | `list` | `Array` | `Vec<T>` |
| マップ | `dict` | `Map`/`Object` | `HashMap<K, V>` |
| イテレータ | ジェネレータ | Array methods | `Iterator`トレイト |
| 変換 | リスト内包表記 | `.map()` | `.iter().map()` |
| 畳み込み | `functools.reduce` | `.reduce()` | `.fold()` |
| 無名関数 | `lambda` | アロー関数 | クロージャ `\|x\| x` |

🎯 **ポイント**:
- **Vec** = 最も使うコレクション、可変長配列
- **HashMap** = キーバリュー、`entry()`APIが便利
- **イテレータ** = 遅延評価、チェーンで処理を組み立て
- **クロージャ** = 環境をキャプチャ、`move`で所有権を取得

---

## 次のステップ

[Chapter 07: モジュール](07-modules.md) では、コードを整理するためのモジュールシステムを学びます。`mod`、`use`、`pub`、クレートの構造を解説します。
