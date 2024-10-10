# 基础语法

Rust 的语法和其他语言的语法有着很多的不同，可以说是复杂很多。虽然写一篇将语法的文档非常枯燥（看也是如此），但我还是写在这里，作为字典来查询吧。Rust 的语法，肯定是需要经常回顾的，特别是在现实中，你可能很少会用 Rust 去写一个生产级的项目。

## Hello World {id="hello-world"}

免不了俗，还是从 `Hello World` 的示例开始吧, 先贴代码:
```rust
fn main(){
    let greeting = "Hello World";
    println!("{}", greeting);
}
```
和大部分编译型语言一样，都是从 `main` 函数开始执行的。我们定义了一个字符串类型的变量名为 `greeting`，值为 `Hello World`。然后利用 `println!()` 宏打印 `greeting` 这个变量的值。

## 变量的可变性 {id="mut-variable"} ##

需要注意的是，在 Rust 中，变量默认是不可变的，如果要声明为可变，需要加上 `mut` 关键字:
```rust
let mut x = 5;
x = 6;
printf!("x is {}", x);
```

编译上面的代码，编译器会给出如下警告:
```rust
warning: value assigned to `x` is never read
 --> src/main.rs:2:13
  |
2 |     let mut x = 5;
  |             ^
  |
  = help: maybe it is overwritten before being read?
  = note: `#[warn(unused_assignments)]` on by default
```
这是告诉我们，`x` 的初始值 `5` 在被读取之前就已经被覆写为 `6`。可以把 `x` 的初始值直接写为 `6`。由此可见，Rust 的编译器给出了保姆级的提示，并且还会给出代码的修改意见。

## 函数 {id="function"} ##

接下来我们来介绍函数的写法。

### 闭包函数 {id="closure"} ##

闭包函数是一种匿名函数，可以捕获周围环境中的便利那个的匿名函数，通常用在需要传递行为的场合比如说线程、迭代器和异步编程中。

比如在一个应用程序中，我需要初始化配置。但是整个应用的生命周期中，我只需要初始化一次，就可以使用如下的写法，其包含了一个匿名函数:
```rust
use std::sync::Once;
use dotenv::dotenv;

static INIT: Once = Once::new();

pub fn init() {
    INIT.call_once(|| {
        dotenv().ok();
    });
}
```

闭包语法有几种写法:
```rust
// 无参数且不指定返回类型
|| {}
// 有参数但不指定返回类型
|x, y| {}
// 有参数且指定返回类型
|x, y| -> i32 {}
// 有参数但是不使用
|_| {}
```