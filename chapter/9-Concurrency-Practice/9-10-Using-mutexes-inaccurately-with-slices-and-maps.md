## 9.10 切片和 map 使用互斥锁引入的坑

在数据既可变又共享的并发上下文中工作时，我们通常必须使用互斥锁围绕数据结构实现受保护的访问。一个常见的错误是在使用切片和 map 时不准确地使用互斥锁。让我们看一个具体的例子，了解潜在的问题。

我们将实现一个 `Cache` 结构，用于处理客户余额的缓存。此结构将包含每个客户 ID 的余额 map 和保护并发访问的互斥锁：

```go
type Cache struct {
    mu       sync.RWMutex
    balances map[string]float64
}
```

> **Note** 这个解决方案使用一个 `sync.RWMutex` 来允许多读，但不能写入。

接下来，我们将添加一个将改变 `balances` 的 `AddBalance` 方法。互斥将在临界区完成（在互斥锁内完成）：

```go
func (c *Cache) AddBalance(id string, balance float64) {
    c.mu.Lock()
    c.balances[id] = balance
    c.mu.Unlock()
}
```

同时，我们还有一个需求，实现一个方法来计算所有客户的平均余额。我们可能用这种方式处理最小临界区：

```go
func (c *Cache) AverageBalance() float64 {
    c.mu.RLock()
    balances := c.balances
    c.mu.RUnlock()

    sum := 0.
    for _, balance := range balances {
        sum += balance
    }
    return sum / float64(len(balances))
}
```

首先，我们创建一个 map 到本地 `balances` 变量的副本。仅在临界区完成复制以迭代每个余额并计算临界区之外的平均值。但是，这个解决方案有效吗？

如果我们使用带有两个并发 goroutine 的 `-race` 标志运行测试，一个调用 `AddBalance`（因此改变 `balances` ），另一个调用 `AverageBalance`，它确实会导致数据竞争。那么这里出了什么问题？

在内部，map 是一个 `runtime.hmap` 结构，主要包含一些元数据（例如，一个计数器）和一个引用数据桶的指针。因此，`balances := c.balances` 不会复制实际数据。切片的原理相同：

```go
s1 := []int{1, 2, 3}
s2 := s1
s2[0] = 42
fmt.Println(s1)
```

打印 `s1` 返回 `[42 2 3]`，即使我们修改了 `s2`。原因是 `s2 := s1` 创建了一个新切片：`s2` 具有相同的长度、相同的容量，并且由与 `s1` 相同的数组支持。

回到我们的示例，我们为 `balances` 分配了一个引用与 `c.balances` 相同的数据桶的新 map 。同时，两个 goroutine 将对同一个数据集执行操作，其中一个正在对其进行变异。因此，这就是数据竞争。

我们如何解决数据竞争？我们主要有两种选择。

如果迭代操作不是很重（在我们执行增量操作时就是这种情况），我们应该保护整个函数：

```go
func (c *Cache) AverageBalance() float64 {
    c.mu.RLock()
    defer c.mu.RUnlock()

    sum := 0.
    for _, balance := range c.balances {
        sum += balance
    }
    return sum / float64(len(c.balances))
}
```

在这里，临界区现在包含了整个函数，其中包括迭代。这可以防止数据竞争。

如果迭代操作不是轻量级的，可以使用的另一个选项是处理数据的实际副本并仅保护副本：

```go
func (c *Cache) AverageBalance() float64 {
    c.mu.RLock()
    m := make(map[string]float64, len(c.balances))
    for k, v := range c.balances {
        m[k] = v
    }
    c.mu.RUnlock()

    sum := 0.
    for _, balance := range m {
        sum += balance
    }
    return sum / float64(len(m))
}
```

在这里，一旦我们做了一个深拷贝，我们就会释放互斥锁。迭代在临界区之外的副本上完成。

让我们考虑一下这个解决方案；在这里，我们必须对 map 值进行两次迭代：一次用于复制，一次用于执行操作（此处为增量）。然而，临界区只是 map 副本。因此，当且仅当操作不快时，此解决方案才可能非常适合。例如，如果一个操作需要调用一个外部数据库，这个解决方案可能会更有效。在选择一种或另一种解决方案时，不可能定义阈值，因为它取决于各种因素，例如元素数量和结构的平均大小。

总之，我们必须小心互斥锁的边界。在本节中，我们了解了为什么将现有map（或现有切片）分配给 map 不足以保护数据竞争。事实上，新变量，无论是 map 还是切片，都由相同的数据集支持。有两种主要的解决方案可以防止它，要么保护整个功能，要么处理实际数据的副本。在所有情况下，我们在设计临界区时都要小心谨慎，并确保准确定义边界。

现在让我们讨论使用 `sync.WaitGroup` 时的一个常见错误。