## 6.1 接收器应该定义为值类型还是指针类型？

为方法选择接收器类型并不总是那么简单。我们什么时候应该使用值接收器？我们什么时候应该使用指针接收器？我们将在本节中看到明智的选择方法。

在优化章节中，我们将彻底讨论值与指针。因此，本节将仅涉及表面性能。此外，在许多情况下，使用值或指针接收器不应该由性能决定，而是由我们将讨论的其他条件决定。但首先，让我们重新思考一下接收器是如何工作的。

在 Go 中，我们可以将值或指针接收器附加到方法。使用值接收器，Go 复制该值并将其传递给方法。

作为说明，以下示例更改了值接收器：

```go
type customer struct {
        balance float64
}

func (c customer) add(v float64) {
        c.balance += v
}

func main() {
        c := customer{balance: 100.}
        c.add(50.)
        fmt.Printf("balance: %.2f\n", c.balance)
}			
```

由于我们使用了值接收器，因此在 `add` 方法中增加余额不会改变原始 `customer` 结构的 `balance` 字段：

```shell
100.00
```

另外，使用指针接收器，Go 将对象的地址传递给方法。从本质上讲，它仍然是一个副本，但我们只会复制一个指针，而不是对象本身（通过引用传递在 Go 中不存在）。对接收器的任何修改都将在原始对象上完成。这是相同的示例，但现在接收器是一个指针：

```go
type customer struct {
        balance float64
}

func (c *customer) add(operation float64) {
        c.balance += operation
}

func main() {
        c := customer{balance: 100.0}
        c.add(50.0)
        fmt.Printf("balance: %.2f\n", c.balance)
    }
```

当我们使用指针接收器时，增加余额会改变原始 `customer` 结构的 `balance` 字段：

```shell
150.00
```

在值和指针接收器之间进行选择并不总是那么直截了当。现在让我们讨论帮助我们做出选择的主要条件：

接收器 *必须* 是一个指针：

* 如果方法需要改变接收器。如果接收器是切片并且方法需要附加元素，则此规则也有效：

```go
type slice []int

func (s *slice) add(element int) {
        *s = append(*s, element)
}
```

* 如果方法接收器包含无法复制的字段；例如，`sync` 包的类型部分（我们将在 *复制 `sync` 类型* 中讨论这一点）。

接收器 *应该* 是一个指针：

* 如果接收器是一个大对象。实际上，使用指针可以使调用更有效，因为它可以防止进行大量复制。不确定到底有多大，基准测试可以成为解决方案。事实上，几乎不可能说出具体的尺寸，因为它取决于许多因素。

接收器 *必须* 是一个值：

* 如果我们必须强制接收器的不变性。
* 如果接收器是 map、函数或通道；否则会导致编译错误。

接收器 *应该* 是一个值：

* 如果接收器是一个不必变异的切片。
* 如果接收器是一个小数组或结构，它自然是没有可变字段的值类型，例如：`time.Time`。
* 如果接收器是基本类型，例如 `int`、`float64` 或 `string`。

有一个案例需要更多讨论。假设我们设计了一个不同的 `customer` 结构，其可变字段不是直接属于结构的一部分，而是在另一个结构中：

```go
type customer struct {
        data *data
}

type data struct {
        balance float64
}

func (c customer) add(operation float64) {
        c.data.balance += operation
}

func main() {
        c := customer{data: &data{
                balance: 100,
        }}
        c.add(50.)
        fmt.Printf("balance: %.2f\n", c.data.balance)
}
```

即使接收器是一个值，调用 `add` 最终也会改变实际余额：

```shell
150.00
```

在这种情况下，我们不需要接收器成为改变 `balance` 的指针。然而，为了清楚起见，人们可能倾向于使用指针接收器来突出 `customer` 作为一个整体对象是可变的。

> **Note** 是否允许混合接收器类型？例如，一个包含多个方法的结构，有些带有指针接收器，有些带有值接收器。共识似乎倾向于禁止它。但是，我们可以注意到标准库中的一些反例；特别是一个例子：`time.Time`。
> 
> 事实上，设计者想要强制执行 `time.Time` 是不可变的。因此，大多数方法，如 `After` 、 `IsZero` 或 `UTC` 都有一个值接收器。然而，为了符合现有的接口，例如 `encoding.TextUnmarshaler`，`time.Time` 必须实现 `UnmarshalBinary([]byte)` 错误方法，该方法在给定字节片的情况下改变接收器。因此，对于这种方法，接收器是一个指针。
> 
> 因此，通常应避免混合接收器类型，但在100%的情况下不应禁止。

我们现在应该对使用值接收器还是指针接收器有一个很好的理解。当然，不可能详尽无遗，因为总会有边缘情况。然而，本节的目标是为涵盖大多数情况提供指导。默认情况下，我们可以选择使用价值接收器，除非有充分的理由不这样做。有疑问，我们应该使用指针接收器。

在下一节中，我们将讨论命名结果参数：提醒我们它们是什么以及何时使用它们。