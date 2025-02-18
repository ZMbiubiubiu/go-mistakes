## 3.9 使用切片附加的意外副作用

本节将讨论使用 `append` 时的一个常见错误，在某些情况下可能会产生意想不到的副作用。

在以下示例中，我们将初始化 `s1` 切片，通过切片 `s1` 创建 `s2`，并通过将元素附加到 `s2` 来创建 `s3`：

```go
s1 := []int{1, 2, 3}
s2 := s1[1:2]
s3 := append(s2, 10)
```

我们初始化了一个包含三个元素的 `s1` 切片。`s2` 是由切片 `s1` 创建的。然后，我们在 `s3` 上调用 `append`。此代码末尾的这三个切片应该是什么状态？你能猜到吗？

在第二行之后，就在创建 `s2` 之后，以下是内存中两个切片的状态：

![](https://img.exciting.net.cn/16.png)

`s1` 是一个长度为3、容量为3的切片，而 `s2` 是长度为1、容量为2的切片，两者都由我们前面提到的同一个数组支持。

使用 `append` 添加元素会检查切片是否已满（长度==容量）。如果未满，`append` 函数将通过更新备份数组并返回长度增加1的切片来添加元素。

在本例中，`s2` 没有满；它可以接受另一个元素。因此，以下是这三个切片的最终状态：

![](https://img.exciting.net.cn/17.png)

在备份数组中，最后一个元素已更新为存储10。因此，如果我们打印所有切片，以下是输出：

```go
s1=[1 2 10], s2=[2], s3=[2 10]
```

尽管我们既没有直接更新 `s1[2]` 也没有直接更新 `s2[1]`，但 `s1` 切片的内容已被修改。因此，我们应该牢记这一点，以避免意外后果。

让我们在传递切片结果时看看这个原理的一个影响
对函数的操作。我们将初始化一个包含三个元素的切片
并调用仅包含前两个元素的函数：

```go
func main() {
        s := []int{1, 2, 3}

        f(s[:2])
        // Use s
}

func f(s []int) {
        // Update s
}
```

在此实现中，如果 `f` 更新前两个元素，更改将作用于 `main` 函数的切片上。但是，如果 `f` 调用 `append`，它将更新切片的第三个元素，即使我们只传递了两个元素：

```go
func main() {
        s := []int{1, 2, 3}

        f(s[:2])
        fmt.Println(s) // [1 2 10]
}

func f(s []int) {
        _ = append(s, 10)
}
```

如果我们出于防御原因想要保护第三个元素，这意味着确保 `f` 不会更新它，我们有两种选项。

第一个选项是传递切片的副本，然后构造生成的切片：

```go
func main() {
        s := []int{1, 2, 3}
        sCopy := make([]int, 2)
        copy(sCopy, s)

        f(sCopy)
        result := append(sCopy, s[2])
        // Use result
}

func f(s []int) {
        // Update s
}
```

当我们将副本传递给 `f` 时，即使此函数调用 `append`，也不会导致超出前两个元素范围的副作用。此解决方案的缺点是它使代码更难阅读，并添加了一个额外的副本，如果切片很大，这可能是一个问题。

另一种解决方案可用于将潜在副作用的范围仅限于前两个元素。该解决方案涉及所谓的全切片表达式：`s[low:high:max]`。此语句创建一个与使用 `s[low:high]` 创建的切片类似的切片，只是结果切片的容量将等于 `max - low`。这是调用 `f` 时的示例：

```go
func main() {
        s := []int{1, 2, 3}
        f(s[:2:2])
        // Use s
}

func f(s []int) {
        // Update s
}
```

在这里，传递给 `f` 的切片不是 `s[:2]` 而是 `s[:2:2]` 。因此，切片的容量为2 - 0 = 2，如下图所示：

![](https://img.exciting.net.cn/18.png)

传递 `s[:2:2]` 时，我们可以仅将效果范围限制为前两个元素。同时，它还避免了必须执行切片复制。

使用切片时，我们必须记住，我们可能会面临导致意外副作用的情况。实际上，如果结果切片的长度小于其容量，则 `append` 可以改变原始切片。如果我们想限制可能的副作用的范围，我们可以使用切片副本或完整的切片表达式，这会阻止我们进行复制。

在下一节中，我们将继续讨论切片，但在潜在内存泄漏的背景下。