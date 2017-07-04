title: rust 笔记 - 结构体
author: cwen
date: 2017-07-05
update: 2017-07-05
tags:
    - 笔记
    - rust

---

入坑 rust   <!--more-->
## 结构体

```rust
// 定义
 struct User {
    username: String,
    email: String,
    sign_in_count: u64,
    active: bool,
}
// 声明使用
let user1 = User {
    email: String::from("someone@example.com"),
    username: String::from("someusername123"),
    active: true,
    sign_in_count: 1,
};
```
为了从结构体中获取某个值，可以使用点号。如果我们只想要用户的邮箱地址，可以用user1.email。

## 通过衍生 trait 增加实用功能

```rust
#[derive(Debug)]
struct Rectangle {
    length: u32,
    width: u32,
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!("rect1 is {:?}", rect1); // {} Display , {:?} Debug
}
```

## 方法

```rust
#[derive(Debug)]
struct Rectangle {
    length: u32,
    width: u32,
}

impl Rectangle {
    fn area(&self) -> u32 {
        self.length * self.width
    }
}

fn main() {
    let rect1 = Rectangle { length: 50, width: 30 };

    println!(
        "The area of the rectangle is {} square pixels.",
        rect1.area()
    );
}
```
方法的第一个参数可以是 &self / self / &mut self(可变引用)

## 关联函数

impl块的另一个好用的功能是：允许在impl块中定义不以self作为参数的函数。这被称为关联函数（associated functions），因为他们与结构体相关联。即便如此他们也是函数而不是方法，因为他们并不作用于一个结构体的实例。你已经使用过一个关联函数了：String::from。

关联函数经常被用作返回一个结构体新实例的构造函数。例如我们可以提供一个关联函数，它接受一个维度参数并且用来作为长和宽，这样可以更轻松的创建一个正方形Rectangle而不必指定两次同样的值：

```rust
impl Rectangle {
    fn square(size: u32) -> Rectangle {
        Rectangle { length: size, width: size }
    }
}
```

使用结构体名和::语法来调用这个关联函数：比如let sq = Rectangle::square(3);。这个方法位于结构体的命名空间中：::语法用于关联函数和模块创建的命名空间。

