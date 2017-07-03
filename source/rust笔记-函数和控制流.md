title: rust 笔记 - 函数 && 控制流
author: cwen
date: 2017-07-03
update: 2017-07-02
tags:
    - 笔记
    - rust

---

入坑 rust   <!--more-->
## 函数
Rust 代码使用 snake case 作为函数和变量名称的规范风格，在 snake case 中，所有字母都是小写并使用下划线分隔单词。Rust 中的函数定义以fn开始并在函数名后跟一对括号。大括号告诉编译器哪里是函数体的开始和结尾。

```rust
fn main() {
    another_function(5);
}

fn another_function(x: i32) {
    println!("The value of x is: {}", x);
}
```

## 语句和表达式

* 语句执行一些操作但不返回的指令

```rust
fn main() {
    let x = 4;
}
```

* 表达式计算并产生一个值。函数调用是一个表达式。宏调用是一个表达式。我们用来创新建作用域的大括号（代码块），{}，也是一个表达式

```rust
fn main() {
    let x = 5;

    let y = {
        let x = 3;
        x + 1
    };

    println!("The value of y is: {}", y);
}
```

## 函数返回值

可以向调用它的代码返回值。并不对返回值命名，不过会在一个箭头（->）后声明它的类型。在 Rust 中，函数的返回值等同于函数体最后一个表达式的值。

```rust
fn five() -> i32 {
    5
}

fn main() {
    let x = five();

    println!("The value of x is: {}", x);
}
```
## if 表达式

与大多数编程语言类似，if 用来控制分支，代码中的条件必须是bool

```rust
fn main() {
    let number = 6;
    if number % 4 == 0 {
        println!("number is divisible by 4");
    } else if number % 3 == 0 {
        println!("number is divisible by 3");
    } else if number % 2 == 0 {
        println!("number is divisible by 2");
    } else {
        println!("number is not divisible by 4, 3, or 2");
    }
    // 当 else if 过多的时候就该尝试重构代码了(match替换)
}
```
来点不一样的，let 语句中使用 if

```rust
fn main() {
    let condition = true;
    let number = if condition {
        5
    } else {
        6
    };
    println!("The value of number is: {}", number);
}

```
注意点： if的每个分支的可能的返回值都必须是相同类型；

## 循环

Rust 有三种循环类型：loop、while和for。

### loop 循环

loop关键字告诉 Rust 一遍又一遍的执行一段代码直到你明确要求停止(比如break)。

```rust
fn main() {
    let mut index = 0;
    loop {
        println!("again!");
        index = index + 1;
        if index >= 10 {
            println!("done");
            break;
        }
    }
}
```
### while 循环 && for 循环

```rust
fn main() {
    let a = [10, 20, 30, 40, 50];
    let mut index = 0;
    // while 循环
    while index < 5 {
        println!("the value is: {}", a[index]);

        index = index + 1;
    }
    // for 循环
    for element in a.iter() {
        println!("the value is: {}", element);
    }
}
```


