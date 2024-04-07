+++
title = "Rust所有権周り"
date = 2024-03-25
draft = true
+++

### 所有権
[4. 所有権を理解する](https://doc.rust-jp.rs/book-ja/ch04-00-understanding-ownership.html) より
```rust
// これだけでmoveされる
// moveはshallow copy
let a = b
```
これがだめで

```rust
fn main() {
    let x = String::from("Hello");
    let y = x;
    println!("{:?}", y);
    println!("{:?}", x);
}
```

```
[~/topic_tree]$cargo run
   Compiling topic_tree v0.1.0 (/Users/syureneko/topic_tree)
    Finished dev [unoptimized + debuginfo] target(s) in 0.58s
     Running `target/debug/topic_tree`
5
[~/topic_tree]$cargo run
   Compiling topic_tree v0.1.0 (/Users/syureneko/topic_tree)
    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running `target/debug/topic_tree`
5
5
[~/topic_tree]$cargo run
   Compiling topic_tree v0.1.0 (/Users/syureneko/topic_tree)
error[E0382]: borrow of moved value: `x`
 --> src/main.rs:5:22
  |
2 |     let x = String::from("Hello");
  |         - move occurs because `x` has type `String`, which does not implement the `Copy` trait
3 |     let y = x;
  |             - value moved here
4 |     println!("{:?}", y);
5 |     println!("{:?}", x);
  |                      ^ value borrowed here after move
  |
  = note: this error originates in the macro `$crate::format_args_nl` which comes from the expansion of the macro `println` (in Nightly builds, run with -Z macro-backtrace for more info)
help: consider cloning the value if the performance cost is acceptable
  |
3 |     let y = x.clone();
  |              ++++++++

For more information about this error, try `rustc --explain E0382`.
```
これは通る、

```rust
fn main() {
    let x = 5;
    let y = x;
    println!("{:?}", y);
    println!("{:?}", x);
}
```

> その理由は、整数のようなコンパイル時に既知のサイズを持つ型は、スタック上にすっぽり保持されるので、 実際の値をコピーするのも高速だからです。これは、変数yを生成した後にもxを無効化したくなる理由がないことを意味します。 換言すると、ここでは、shallow copyとdeep copyの違いがないことになり、 cloneメソッドを呼び出しても、一般的なshallow copy以上のことをしなくなり、 そのまま放置しておけるということです。


### 参照と借用
関数の引数に参照を使うことを借用させると呼ぶ。参照では所有権はムーブしない。
関数内で使用した参照は関数外に出せない。なぜなら、参照が宙に浮くので、回避策は参照を取らずに値の所有権ごと渡す。
可変参照例：
```rust
fn main() {
    let mut s = String::from("hello");

    change(&mut s);
}

fn change(some_string: &mut String) {
    some_string.push_str(", world");
}
```

### Rc<T>

[Rust book: 15.4. Rc<T>は、参照カウント方式のスマートポインタ](https://doc.rust-jp.rs/book-ja/ch15-04-rc.html)より

> 複数の所有権を可能にするため、RustにはRc<T>という型があり、これは、reference counting(参照カウント)の省略形です。 
Rc::cloneの実装は、多くの型のclone実装のように、全てのデータのディープコピーをすることではありません。 Rc::cloneの呼び出しは、参照カウントをインクリメントするだけであり、時間はかかりません。 


```rust
enum List {
    Cons(i32, Rc<List>),
    Nil,
}

use List::{Cons, Nil};
use std::rc::Rc;

fn main() {
    let a = Rc::new(Cons(5, Rc::new(Cons(10, Rc::new(Nil)))));
    let b = Cons(3, Rc::clone(&a));
    let c = Cons(4, Rc::clone(&a));
}

```

理解：Rcを使えばCloneで所有権を共有できる。内部ではReference Countを持っており0になった際に回収される。これはコンパイル時に判断される。ただし、Rc<T>は読み取り専用。

### Weak
https://doc.rust-jp.rs/book-ja/ch15-06-reference-cycles.html


### RefCall
