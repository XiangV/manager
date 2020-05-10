---
title: Rust语言
date: 2020-05-10 15:48:29
tags: rust-lang
categories: 编程语言
---



## Rust 初识

初次知道rust-lang要从PingCap开始，PingCap开源的TiKV使用了rust作为开发语言，底层是RocksDB，使用rust ffi调用RocksDB的接口。所以就顺着去了解了一下Rust语言。



看了看Rust的语言特性，确实比较吸引人：所有权系统、泛型、trait、闭包、并发，包括了一个现代语言应该具备的所有特性。



## 语言特性

### 所有权系统

Rust主打的特性是并发内存安全，为了避免c/c++中不安全的内存操作（空悬指针、内存泄漏）创建了所有权系统。所有权系统保证同一时刻不会有多个write，或者write+read，每一时刻数据只有一个owner。

rust编译器在编译阶段就能找出所有不符合rust标准的使用方式，从而避免运行期的不安全。也正是编译期的严格检查，让初学者在编译阶段比较头疼，熟悉之后，使用rust的方式去思考和coding可以避免一些常见的编译问题。

也正是编译期的严格检查，只要编译完成，二进制基本可以无内存问题的安全运行。

所有权系统实际上管理的是堆上内存，指向堆上内存的指针的owner问题。

rust默认声明的变量不可变，这个原则的基础是：多线程下read是安全的。如果变量可变，需要使用mut关键字。

这种所有权系统在参数传递是"复制"时会与c/c++编程经验不一样，比如：

```rust
fn just_print(source: String) {
    println!("{}", source);
}
fn main() {
    // "Rust-lang is pretty goog" 是 `static &str
    // 调用 to_string() 方法复制到堆上
    let a = "Rust-lang is pretty good".to_string();
    just_print(a);
    println!("{}", a)
}
```

这样是编译通过不了的，报错如下：

{% asset_img rust-owner.png Compile Error %}

因为调用`just_print`时`a`的所有权被move到参数中去了，在`main`中剩下的是一个无效的变量。修正方式是`just_print`参数修改为`source: &String`

### 泛型(generic types)和trait

rust支持泛型，这个与c++中的模板类似，泛型参数代表了满足某种trait的类型。

trait实际与java中的interface类似，定义了某些行为。在泛型中，通常需要指定泛型参数满足那些trait，因为你的泛型对象、泛型函数中一定是使用泛型参数对象的一些通用行为的。

```rust
fn template_example<T: Clone + Debug>(p1: &T) {
    println("{:?}", p1);
} 
```

比如上面的例子定义了一个泛型函数，泛型参数`T`需要实现了`Clone`和`Debug`两个trait。

### 闭包

闭包实际上在很多语言中有实现，c++语言从c++11开始支持lambda。rust中的闭包定义如下：

```rust
move |p1: Type1, p2: Type2,...| -> ReturnType {
}
```

闭包可以捕获上下文环境中的自由变量，`move`指定将捕获的自由变量所有权转移到闭包内。

### 并发

rust中实现并发非常简单

```rust
use std::thread;
thread::Spawn(||{})
```

引入`std::thread`，调用其`spawn`就创建出一个线程。并发的安全性由两个trait：`Send`  `Sync`

实现了`Send`就可以安全的在线程间传递，实现了`Sync`就可以安全的在线程间传递不可变借用。

## 总结

这里非常简单的介绍了Rust中一些特性，实际使用中还有很多细节，还有很多高级特性没有介绍（macro，unsafe）。总体来说rust还是很吸引人的，以后找些事情用rust实现尝试一下。