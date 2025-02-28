## 9.3 goroutine 和循环变量，一不小心就出错

错误处理 goroutine 和循环变量可能是 Go 开发人员在编写并发应用程序时最常见的问题之一。

让我们看一个具体的例子；然后，我们将定义此类错误的条件以及如何防止它。

让我们思考以下示例。我们将初始化一个切片；然后，在作为新 goroutine 执行的闭包中，我们将访问这个元素：

```go
s := []int{1, 2, 3}
    
for _, i := range s {
    go func() {
        fmt.Print(i)
    }()
}
```

我们可能希望此代码以没有特定顺序的方式打印 `123`（因为无法保证首先创建的 goroutine 将首先执行完成）。但是，此代码的输出不是确定性的。例如，有时它可以打印 `233`，有时可以打印 `333`。什么原因？

在这个例子中，我们从一个闭包创建新的 goroutine。提醒一下，闭包是一个从函数体外部引用变量的函数值。这里是 `i` 变量。我们必须知道，当一个闭包 goroutine 被执行时，它并没有捕获 goroutine 创建时的值。相反，所有的 goroutine 都引用了相同的变量。当一个 goroutine 运行时，它会在 `fmt.Println` 执行时打印 `i` 的值。因此，自 goroutine 启动以来，`i` 可能已被修改。

让我们看看代码打印 `233` 时可能的执行情况：

![](https://img.exciting.net.cn/57.png)

随着时间的推移，`i` 的值从 1、2 到 3 不等。在每次迭代中，我们启动一个新的 goroutine。由于无法保证每个 goroutine 何时开始和完成，因此结果也会有所不同。在这个例子中，第一个 goroutine 在 `i` 等于 2 时打印它。然后，其他 goroutine 在值已经等于 3 时打印它。因此，这个例子打印 `233`。此代码的行为不是确定性的。

如果我们希望每个闭包在创建 goroutine 时访问 `i` 的值，有什么解决方案？

第一个，如果我们想继续使用闭包，需要创建一个新变量：

```go
for _, i := range s {
    val := i
    go func() {
        fmt.Print(val)
    }()
}
```

为什么这段代码有效？在每次迭代中，我们都会创建一个新的局部 `val` 变量。此变量在创建 goroutine 之前捕获 `i` 的当前值。因此，当每个闭包 goroutine 执行 print 语句时，它会使用预期值执行它，这段代码打印 `123`（同样，没有特定的顺序）。

第二种选择不再依赖闭包，而是依赖实际函数：

```go
for _, i := range s {
    go func(val int) {
        fmt.Print(val)
    }(i)
}
```

我们仍然用一个 goroutine 执行一个匿名函数（例如，我们不运行 `go f(i)`），但这次它不是闭包。该函数不会将 `val` 作为其主体外部的变量进行引用；`val` 现在是函数输入的一部分。通过这样做，我们在每次迭代中修复 `i` 并使我们的应用程序按预期工作。

我们必须谨慎使用 goroutine 和循环变量。如果 goroutine 是一个访问从其主体外部声明的迭代变量的闭包，那么这是一个问题。我们可以通过创建一个局部变量来解决它（例如，正如我们在执行 goroutine 之前使用 `val:= i` 所看到的那样）或使函数不再是闭包。两种选择都有效，我们不应该偏爱另一种。人们可能会发现闭包方法更方便，而函数方法可能更具表现力。

多个通道上的 `select` 语句会发生什么？下一章我们一起聊聊。