title: golang 编程基础 - Hello,Word
author: cwen
date: 2015-12-02 
update: 2015-12-02 
tags:
    - 编程 
    - golang   

--- 

最近越发觉得自己的golang基础还是不够扎实,所有决定再从头捋一遍golang的基础知识
同时也为golang的爱好者们提供点入门材料.  
<!--more-->     
我们就从这个经典的 `Hello Word` 案例开始吧!(先上代码)  
```
//HelloWord.go
package main

import "fmt"

func main() {
    fmt.Println("Hello, Word")
}
```  
接着我们打开终端
```  
$ cd $GOPATH/src/***    //进入你的文件目录 
$ go run helloWord.go  
```  
毫不意外,命令会输出  
```  
Hello,Word
```

同时我们还可以这样来干 
```
$ go build helloWord.go  
```  
这样你会在当前目录下找到一个可执行的二进制文件,不需要任何其他处理下,你就可以在任何时间来运行这个二进制文件了(注：因为是静态编译，所以也不用担心在系统库更新的时候冲突，幸福感满满)  

```  
$ ./helloWord
Helllo,Word
```  


#### 代码详解  

看到代码的第一行,熟悉 Java,Python 的同学会觉得很熟悉,没错golang也是使用 `package` 的来组织代码的, 一个 `package` 会包含一个或多个`.go`结束的源代码文件。每一个源文件都是以一个 `package xxx`的声明开头的，比如我们的例子里就是 `package main` 。这行声明表示该文件是属于哪一个 `package`，紧跟着是一系列 `import` 的 `package` 名，表示这个文件中引入的 `package` 。再之后是本文件本身的代码   

> main.main 为函数的入口(main包 main函数)       

代码的第二句相信大家也都猜到了,导入 `fmt` 包,    
`fmt` 包是干什么的呢?    
`fmt` 包实现了类似 `C` 语言 `printf` 和 `scanf` 的格式化 `I/O` 。格式化动作（'verb'）源自C语言但更简单。  

接下让我们来看 `main` 函数, `main` 必须存在与 `main` 包内, 这是我们整个程序的入口(注：其实c系语言差不多都是这样)。main函数所做的事情就是我们程序做的事情。当然了，`main` 函数一般完成的工作是调用其它 `packge` 里的函数来完成自己的工作，比如 `fmt.Println` 。  

`main` 函数内我们调用了 `fmt` 包里面定义的函数 `Println`。大家可以看到，这个函数是通过`<pkgName>.<funcName>`的方式调用的.  
`Println` 函数类似与 c 语言的 `printf` ,只是在 `printf` 函数的基础上在输出最后加上一个格式化换行符.  

> 到此我们的 `helloWord.go` 代码分析结束  
> 下一篇博文让我真正的走进golang `<<goalng编程基础-基本语法>>`   
