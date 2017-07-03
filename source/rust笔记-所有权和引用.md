title: rust 笔记 - 所有权 && 引用
author: cwen
date: 2017-07-03
update: 2017-07-03
tags:
    - 笔记
    - rust

---

入坑 rust   <!--more-->

## 所有权
规则：
1. 每一个值都被它的所有者（owner）变量拥有。
2. 值在任意时刻只能被一个所有者拥有。
3. 当所有者离开作用域，这个值将被丢弃。

### 变量作用域

1. 当变量进入作用域，它就是有效的。
2. 这一直持续到它离开作用域为止。

```rust
{                      // s is not valid here, it’s not yet declared
    let s = "hello";   // s is valid from this point forward

    // do stuff with s
}                      // this scope is now over, and s is no longer valid
```

### String 类型

基础类型大多数都是存储在栈上并且在离开作用域的时候被移出栈，String 类型是存储在堆上的类型，以 String 类型研究 Rust 如何清理数据都。

先说 String 数据移动

```rust
let s1 = String::from("hello");
let s2 = s1;
```

![内存操作图](https://kaisery.github.io/trpl-zh-cn/img/trpl04-02.svg)

String::form 从堆上分配一块内存存储数据，然后s1指针指向数据内存，当将 s1 赋值给 s2 的时候，不会复制数据，只是将 s2 的指针同样指向之前分配的数据块，同事s1将数据的控制权移交给s2, s1也就无法被继续操作。
但是对于栈上的数据，就是会直接copy数据，并不会存在这样的控制权移交的情况。栈上的数据实际是拥有copy trait，如果一个类型拥有Copy trait，一个旧的变量在（重新）赋值后仍然可用。Rust 不允许自身或其任何部分实现了Drop trait 的类型使用Copy trait。如果我们对其值离开作用域时需要特殊处理的类型使用Copy注解，将会出现一个编译时错误。

### 所有权与函数

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope.

    takes_ownership(s);             // s's value moves into the function...
                                    // ... and so is no longer valid here.
    let x = 5;                      // x comes into scope.

    makes_copy(x);                  // x would move into the function,
                                    // but i32 is Copy, so it’s okay to still
                                    // use x afterward.

} // Here, x goes out of scope, then s. But since s's value was moved, nothing
  // special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope.
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
  // memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope.
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.
```

### 返回值与作用域

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
                                        // value into s1.

    let s2 = String::from("hello");     // s2 comes into scope.

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
                                        // takes_and_gives_back, which also
                                        // moves its return value into s3.
} // Here, s3 goes out of scope and is dropped. s2 goes out of scope but was
  // moved, so nothing happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
                                             // return value into the function
                                             // that calls it.

    let some_string = String::from("hello"); // some_string comes into scope.

    some_string                              // some_string is returned and
                                             // moves out to the ```callfn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
                                             // function.
}

// takes_and_gives_back will take a String and return one.
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
                                                      // scope.

    a_string  // a_string is returned and moves out to the calling function.
}
```

## 引用

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize {
    s.len()
}
```

![](https://kaisery.github.io/trpl-zh-cn/img/trpl04-05.svg)

### 可变引用
使用 mut 修饰，不过可变引用有一个很大的限制：在特定作用域中的特定数据有且只有一个可变引用，我们也不能在拥有不可变引用的同时拥有可变引用。不可变引用的用户可不希望在它的眼皮底下值突然就被改变了！然而，多个不可变引用是没有问题的因为没有哪个读取数据的人有能力影响其他人读取到的数据。

### 引用的规则

简要的概括一下对引用的讨论：

1. 在任意给定时间，只能拥有如下中的一个:一个可变引用;任意数量的不可变引用。
2. 引用必须总是有效的。

