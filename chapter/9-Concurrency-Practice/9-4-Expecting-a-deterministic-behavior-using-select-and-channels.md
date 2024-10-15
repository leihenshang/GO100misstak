## 9.4 期望使用 select 和 channel 的确定性执行

Go 开发人员在使用通道时常犯的一个错误是对 select 在多个通道中的行为方式做出错误的假设。这种错误的假设可能会导致难以识别和重现的细微错误。

假设我们要实现一个需要从两个通道接收的 goroutine：

* `messageCh` 用于处理新消息。
* `disconnectedCh` 接收传达一些断开连接的通知。在那种情况下，我们想从父函数返回功能。

在这两个通道之间，我们要优先考虑 `messageCh`。例如，如果发生断开连接，我们要确保在返回之前已经收到所有消息。

我们可能会决定像这样处理优先级：

```go
for {
    select {
    case v := <-messageCh:
        fmt.Println(v)
    case <-disconnectCh:
        fmt.Println("disconnection, return")
        return
    }
}
```

我们使用 `select` 从多个通道接收。由于我们想要确定 `messageCh` 的优先级，因此可以假设先编写 `messageCh` case，然后再编写 `disconnectedCh` case。但是这段代码是否有效？让我们通过编写一个虚拟生产者 goroutine 来尝试一下，该 goroutine 将发送十条消息，然后发送断开连接通知：

```go
for i := 0; i < 10; i++ {
    messageCh <- i
}
disconnectCh <- struct{}{}
```

如果我们运行上面的示例，如果 `messageCh` 被缓冲，这将是一个可能的输出：

```shell
0
1
2
3
4
disconnection, return
```

因此，我们没有消费这十条消息，而是只收到了其中的五条；什么原因？它在于具有多个通道的 `select` 语句的规范：

> 如果一个或多个通信可以进行，则通过统一的伪随机选择选择一个可 以进行的通信。
>
> -- 编程语言规范

与第一个匹配的情况获胜的 `switch` 语句不同，如果可能有多个选项，则 `select` 语句将随机选择一个。

这种行为一开始可能看起来很奇怪，但有一个很好的理由：防止可能的饥饿。实际上，假设选择的第一个可能的通信是基于源顺序的。在这种情况下，我们可能会陷入这样一种情况，例如，由于发送者速度快，我们只能从一个通道接收。为了防止这种情况，语言设计者决定使用随机选择。

回到我们的示例，即使 `case v := <-messageCh` 是源顺序中的第一个，如果 `messageCh` 和 `disconnectCh` 中都有一条消息，则无法保证将选择哪种情况。因此，前面示例的行为不是确定性的。我们可以收到零条消息，或者五条消息，甚至十条消息：对此无法保证。

那么我们如何才能尝试克服这种情况呢？如果我们想在断开连接的情况下返回之前接收所有消息，则有不同的选项。

如果只有一个生产者 goroutine，我们有两种选择：

* 使 `messageCh` 成为无缓冲通道而不是缓冲通道。由于发送方 goroutine 阻塞直到接收方 goroutine 准备就绪，它会保证在收到来自 `disconnectedCh` 的断开连接之前收到来自 `messageCh` 的所有消息。
* 或者使用单个通道而不是两个。例如，我们可以定义一个 `struct` 来传达新消息或断开连接。由于通道保证发送消息的顺序与接收消息的顺序相同，因此我们可以确保最终收到断开连接。

如果我们遇到有多个生产者 goroutine 的情况，可能无法保证哪个先写。因此，无论是无缓冲的 `messageCh` 通道还是单个通道，都会导致生产者 goroutine 之间的竞争条件。在这种情况下，我们可以实现以下解决方案:

* 从 `messageCh` 或 `disconnectCh` 接收
* 如果收到 来自 `disconnectCh` 通道的消息 ：
    - 接收 `messageCh` 中的所有现有消息（如果有）
    - 然后返回

这将是解决方案：
```go
for {
    select {
    case v := <-messageCh:
        fmt.Println(v)
    case <-disconnectCh:
        for {
            select {
            case v := <-messageCh:
                fmt.Println(v)
            default:
                fmt.Println("disconnection, return")
                return
            }
        }
    }
}
```

此解决方案使用具有两种情况的内部 `for/select`，一种在 `messageCh` 情况下，另一种在 `default` 情况下。仅当其他情况都不匹配时，才会选择在 `select` 语句中使用 `default`。在这种情况下，这意味着我们只有在收到 `messageCh` 中的所有剩余消息后才会返回。

让我们通过一个可视化示例看看这段代码是如何应用的。我们将考虑在 `messageCh` 中有两条消息，在 `disconnectCh` 中有一条断开连接的情况：

![](https://img.exciting.net.cn/58.png)

在这种情况下，正如我们所说，`select` 将随机选择一种或另一种情况。假设 `select` 选择了第二种情况：

![](https://img.exciting.net.cn/59.png)

因此，我们收到断开连接，现在进入内部 `select`：

![](https://img.exciting.net.cn/60.png)

在这里，只要消息保留在 `messageCh` 中，`select` 就会始终
第一种情况优先于 `default` 情况：

![](https://img.exciting.net.cn/61.png)

然后，一旦我们收到来自 `messageCh` 的所有消息，`select` 将不会阻止而是选择 `default` 情况：

![](https://img.exciting.net.cn/62.png)

因此，我们将返回并停止这个 goroutine。

这是一种确保我们从具有多个通道上的接收器的通道接收所有剩余消息的方法。当然，如果在 goroutine 返回后发送 `messageCh`（例如，如果我们有多个生产者 goroutine），我们将错过此消息。

当对多个通道使用 `select` 时，我们必须记住，如果可能有多个选项，则不是源顺序中的第一种情况会获胜。相反，Go 会随机选择它，因此无法保证会选择哪一个。为了克服这种行为，在单个生产者 goroutine 的情况下，我们可以使用无缓冲通道或使用单个通道。在多个生产者 goroutine 的情况下，我们可以使用内部选择和 `default` 值来处理优先级。

以下部分将讨论一种常见的通道类型：通知通道。

错误示范：

```go
package main

  

import (

    "fmt"

    "time"

)

  

var ch1 = make(chan int, 10)

var discountCh1 = make(chan int)

  

func main() {

  

    go func() {

        for {

            select {

            case v := <-ch1:

                time.Sleep(time.Millisecond * 500)

                fmt.Println("ch1", v)

            case <-discountCh1:

                fmt.Println("discountCh1")

                return

            }

        }

    }()

  

    go func() {

        for i := 0; i < 10; i++ {

            ch1 <- i

        }

        fmt.Println("send values to chan1 is completed")

    }()

  

    go func() {

        time.Sleep(time.Second * 3)

        discountCh1 <- 1

    }()

  

    time.Sleep(time.Second * 5)

    fmt.Println("the end")

}
```

如何修复？
```go
package main

  

import (

    "fmt"

    "time"

)

  

var ch1 = make(chan int, 10)

var discountCh1 = make(chan int)

  

func main() {

  

    go func() {

        for {

            select {

            case v := <-ch1:

                time.Sleep(time.Millisecond * 500)

                fmt.Println("ch1", v)

            case <-discountCh1:

                fmt.Println("discountCh1")

                for {

                    select {

                    case v := <-ch1:

                        time.Sleep(time.Millisecond * 500)

                        fmt.Println("ch1 in the disconnect", v)

                    default:

                        fmt.Println("disconnect default")

                        return

                    }

                }

  

            }

        }

    }()

  

    go func() {

        var i = 0

        for ; i < 10; i++ {

            ch1 <- i

        }

        fmt.Println("send values to chan1 is completed", i)

    }()

  

    go func() {

        time.Sleep(time.Second * 3)

        discountCh1 <- 1

    }()

  

    time.Sleep(time.Second * 10)

    fmt.Println("the end")

}

```