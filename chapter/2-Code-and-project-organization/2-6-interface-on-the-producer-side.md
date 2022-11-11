## 2.6 在生产者端的接口

我们在上一节中已经看到，接口被认为是有价值的。然而，Go 开发人员经常误解的一个问题是：接口应该放在哪里？

在深入研究该主题之前，让我们确保我们将在本节中使用的术语清晰：

* 生产者端：与具体实现在同一个包中定义的接口。

![](https://img.exciting.net.cn/27.png)

* 消费者端：定义在外部包中的接口。

![](https://img.exciting.net.cn/28.png)

开发人员在生产者端创建接口以及具体实现是很常见的。这种设计可能是具有 C# 或 Java 背景的开发人员的一种习惯。然而，在 Go 中，在大多数情况下，这不是我们应该做的。

让我们讨论以下示例。我们创建一个特定的包来存储和检索客户数据。同时，我们决定，仍然在同一个包中，所有调用都必须通过以下接口：

```go
package store

type CustomerStorage interface {
        StoreCustomer(customer Customer) error
        GetCustomer(id string) (Customer, error)
        UpdateCustomer(customer Customer) error
        GetAllCustomers() ([]Customer, error)
        GetCustomersWithoutContract() ([]Customer, error)
        GetCustomersWithNegativeBalance() ([]Customer, error)
}
```

我们可能认为我们有很好的理由在生产者端创建和公开这个接口。也许这是将客户端代码与实际实现分离的好方法？也许我们预见到它将帮助客户创建测试替身？但是，这不是 Go 中的最佳实践。

正如我们所提到的，与具有显式实现的语言相比，Go 中隐式地满足了接口，这往往会改变游戏规则。在大多数情况下，要遵循的方法类似于我们在上一节中描述的：应该发现抽象，而不是创建抽象。这意味着生产者不能为所有客户强制一个给定的抽象。

相反，由客户决定他是否需要某种形式的抽象，然后确定最适合他的需要的抽象级别。

在前面的示例中，也许一个调用方不会对解耦他的代码感兴趣。 也许另一个客户想要解耦他的代码，但只对 `GetAllCustomers` 方法感兴趣。 在这种情况下，它可以使用单个方法创建一个接口，从外部包中引用 `Customer` 结构：

```go
package client

type customersGetter interface {
        GetAllCustomers() ([]store.Customer, error)
}
```

从包组织中，它将导致以下结果：

![](https://img.exciting.net.cn/29.png)

有几点需要注意：
* 由于 `customersGetter` 接口仅在 `client`包中被使用，它可以保持未导出。
* 从视觉上看，它看起来像循环依赖。 但是，由于隐式满足接口，因此 `store` 与 `client` 之间没有依赖关系。这就是为什么这种方法在具有显式实现的语言中并不总是可行的原因。

要点是 `client` 包现在可以根据需要定义最准确的抽象（这里只有一个方法）。它与接口隔离原则（SOLID中的I）的概念有关，它指出不应强迫任何调用方依赖它不使用的方法。

因此，在这种情况下，最好的方法是在生产者端公开具体实现，让客户端决定如何使用它以及是否需要抽象。

为了完整起见，让我们提一下，这种方法，生产者端的接口，有时在标准库中使用。例如，`encoding` 包定义了其他子包实现的接口，例如 `encoding/json` 或 `encoding/binary`。那么编码包错了吗？当然不。在这种情况下，编码包中定义的抽象在标准库中使用，语言设计者知道预先创建这种抽象是有价值的。我们回到上一节的讨论：如果您认为抽象可能对想象的未来有所帮助，或者至少如果您无法证明该抽象是有效的，请不要创建抽象。

因此，在大多数情况下，接口应该位于消费者端。但是，在特定情况下，例如，当我们知道（未预见）抽象将对消费者有所帮助时，我们可能希望将其放在生产者端。如果我们这样做了，我们应该努力让它尽可能地最小化，增加它的可重用潜力并使其更容易组合。

让我们继续讨论函数上下文中的接口定义。