# defer关键字

defer和go一样都是Go语言提供的关键字。defer用于资源的释放，会在函数返回之前进行调用。一般采用如下模式：

```
f,err := os.Open(filename)
if err != nil {
    panic(err)
}
defer f.Close()
```

如果有多个defer表达式，调用顺序类似于栈，越后面的defer表达式越先被调用。

不过如果对defer的了解不够深入，使用起来可能会踩到一些坑，尤其是跟带命名的返回参数一起使用时。在讲解defer的实现之前先看一看使用defer容易遇到的问题。

## defer使用时的坑

先来看看几个例子。例1：

```
func f() (result int) {
    defer func() {
        result++
    }()
    return 0
}
```

例2：

```
func f() (r int) {
     t := 5
     defer func() {
       t = t + 5
     }()
     return t
}
```

例3：

```
func f() (r int) {
    defer func(r int) {
          r = r + 5
    }(r)
    return 1
}
```

请读者先不要运行代码，在心里跑一遍结果，然后去验证。

例1的正确答案不是0，例2的正确答案不是10，如果例3的正确答案不是6......

defer是在return之前执行的。这个在 [官方文档](http://golang.org/ref/spec#defer_statements)中是明确说明了的。要使用defer时不踩坑，最重要的一点就是要明白，**return xxx这一条语句并不是一条原子指令!**

函数返回的过程是这样的：先给返回值赋值，然后调用defer表达式，最后才是返回到调用函数中。

defer表达式可能会在设置函数返回值之后，在返回到调用函数之前，修改返回值，使最终的函数返回值与你想象的不一致。

其实使用defer时，用一个简单的转换规则改写一下，就不会迷糊了。改写规则是将return语句拆成两句写，return xxx会被改写成:

```
返回值 = xxx
调用defer函数
空的return
```

先看例1，它可以改写成这样：

```
func f() (result int) {
     result = 0  //return语句不是一条原子调用，return xxx其实是赋值＋ret指令
     func() { //defer被插入到return之前执行，也就是赋返回值和ret指令之间
         result++
     }()
     return
}
```

所以这个返回值是1。

再看例2，它可以改写成这样：

```
func f() (r int) {
     t := 5
     r = t //赋值指令
     func() {        //defer被插入到赋值与返回之间执行，这个例子中返回值r没被修改过
         t = t + 5
     }
     return        //空的return指令
}
```

所以这个的结果是5。

最后看例3，它改写后变成：

```
func f() (r int) {
     r = 1  //给返回值赋值
     func(r int) {        //这里改的r是传值传进去的r，不会改变要返回的那个r值
          r = r + 5
     }(r)
     return        //空的return
}
```

所以这个例子的结果是1。

defer确实是在return之前调用的。但表现形式上却可能不像。本质原因是return xxx语句并不是一条原子指令，defer被插入到了赋值 与 ret之间，因此可能有机会改变最终的返回值。

## defer的实现

defer关键字的实现跟go关键字很类似，不同的是它调用的是runtime.deferproc而不是runtime.newproc。

在defer出现的地方，插入了指令call runtime.deferproc，然后在函数返回之前的地方，插入指令call runtime.deferreturn。

普通的函数返回时，汇编代码类似：

```
add xx SP
return
```

如果其中包含了defer语句，则汇编代码是：

```
call runtime.deferreturn，
add xx SP
return
```

goroutine的控制结构中，有一张表记录defer，调用runtime.deferproc时会将需要defer的表达式记录在表中，而在调用runtime.deferreturn的时候，则会依次从defer表中出栈并执行。