## 3.11 低效的map初始化

本节将讨论我们已经在切片初始化中看到的类似问题，但这次使用的是 map 。但首先，我们需要了解有关如何在 Go 中实现 map 的基础知识，以了解为什么调整如何初始化 map 很重要。

### 3.11.1 概念

 map 提供了键/值对的无序集合，其中所有键都是不同的。

在 Go 中，map 是基于 hashtable 数据结构的。在内部，哈希表是一个桶数组，每个桶是一个指向键/值对数组的指针，如下图所示：

![](../images/21.png)

一个由四个元素组成的数组支持这个哈希表。如果我们深入研究数组索引，我们会注意到一个由单个键/值对（元素）组成的桶：“two”：2。每个桶的大小固定为 8 个元素。

每个操作（读取、更新、插入、删除）都是通过将键关联到数组索引来完成的。此步骤依赖于哈希函数。这个函数是稳定的，因为我们希望它在给定相同键的情况下始终返回相同的桶。在前面的示例中，`hash("two")` 返回 0；因此，该元素存储在数组索引 0 引用的存储桶中。

如果我们插入另一个元素并且散列键返回相同的索引，它会将另一个元素添加到同一个桶中：

![](../images/22.png)

如果插入一个已经满的桶（桶溢出），Go 将创建另一个包含八个元素的桶并将前一个桶链接到它：

![](../images/23.png)

关于读取、更新和删除，也需要计算对应的数组索引。然后，Go将依次遍历所有键，直到找到提供的键。因此，这三个操作的最坏情况时间复杂度是 O(p)，其中 p 是桶中元素的总数（默认为一个桶，溢出时为多个桶）。

现在让我们讨论一下为什么有效地初始化 map 很重要。

### 3.11.2 初始化

为了理解与 map 初始化效率低下相关的问题，让我们创建一个包含三个元素的 `map[string]int` 类型：

```go
m := map[string]int{
    "1": 1,  
    "2": 2,  
    "3": 3,  
}
```

在内部， map 将由一个由单个条目组成的数组支持；因此，一个桶。现在，如果我们添加一百万个元素会发生什么？在这种情况下，单个条目是不够的，因为在最坏的情况下，找到一个密钥意味着要遍历数千个存储桶。这就是为什么 map 应该能够自动增长以应对元素数量的原因。

当 map 增长时，它的桶数将增加一倍。 map 成长的条件是什么？

* 如果桶中的平均项目数（称为负载因子）大于一个常数值。这个常数等于6.5（但在未来的版本中可能会改变，因为它是Go内部的）。
* 如果溢出的桶太多（包含超过八个元素）。

当map增长时，所有的键将再次分派到所有的桶。这就是为什么在最坏的情况下，插入一个键可能是一个 _O(n)_ 操作，其中n是 map 中元素的总数。

我们已经看到使用切片，如果我们预先知道要添加到切片中的元素数量，我们可以用给定的大小或容量对其进行初始化。它避免了必须不断重复昂贵的切片增长操作。这个想法类似于 map 。实际上，我们可以使用 `make` 内置函数在创建 map 时提供初始大小。例如，如果我们要初始化一个包含一百万个元素的 map ，可以这样完成：

```go
m := make(map[string]int, 1_000_000)
```

使用 map ，我们可以只提供初始大小，而不是像切片那样的容量；因此，只有一个参数。

通过指定大小，我们可以提示预期进入 map 的元素数量。在内部，将使用适当数量的桶创建 map 以存储一百万个元素。因此，它将节省大量计算时间，因为 map 不必动态创建存储桶并处理存储桶重新平衡。

此外，指定大小n并不意味着制作具有最多n个元素的 map ：如果需要，我们 仍然可以添加超过n个元素。相反，这意味着要求Go运行时为至少n个元素分配一个空间，如果我们预先知道 大小，这将很有帮助。

为了理解为什么指定大小很重要，让我们运行两个基准测试。前者将在不设置初始大小的 map 中插入一百万个元素，而后者在使用大小初始化的 map 中。

```go
BenchmarkMapWithoutSize-4 6 227413490 ns/op
 BenchmarkMapWithSize-4 13 91174193 ns/op
```

具有初始大小的第二个版本大约快60%。实际上，通过提供大小，我们可以防止 map 不断增长以应对插入的元素。

因此，就像切⽚⼀样，如果我们预先知道 map 将包含的元素数量，我们应该通过提供初始⼤⼩来创建它。这样做可以避免潜在的 map 增⻓，这在计算⽅⾯⾮常繁重，因为它需要重新分配⾜够的空间并重新平衡所有元素。

让我们继续讨论 map ，看看导致内存泄漏的常⻅错误。