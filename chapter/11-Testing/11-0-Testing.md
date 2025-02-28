# 11 测试

> **本章概要**
> * 深入研究有用的 Go 选项以对测试进行分类或使其更健壮
> * 通过了解如何防止使用睡眠调用并了解如何使用时间 API 测试函数，使 Go 测试具有确定性
> * 深入研究实用程序包，例如 `httptest` 和 `iotest`
> * 讨论导致错误假设的常见基准错误
> * 提出一组改进测试过程的技巧

测试是项目生命周期的一个重要方面。它有无数的好处，例如建立对应用程序的信心、充当代码文档或使重构更容易。与其他一些语言相比，Go 具有强大的原语来编写测试。在本节中，我们将深入研究导致测试过程脆弱、效率降低和准确性降低的常见错误。

