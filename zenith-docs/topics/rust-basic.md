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