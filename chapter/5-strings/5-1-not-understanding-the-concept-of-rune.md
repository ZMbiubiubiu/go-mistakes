# 5 字符串

> **本章概要**
> * 理解 Go 中符文的基本概念
> * 防止字符串迭代和修剪的常见错误 
> * 避免因字符串连接或无用转换而导致的低效代码 
> * 使用子字符串避免内存泄漏

在 Go 中，字符串是一种不可变的数据结构，表示：

* 指向不可变字节切片的指针 
* 此数组中的总字节数

我们将在本章中看到 Go 有一种非常独特的方式来处理字符串，并引入了一个名为 runes 的概念。这个概念对于理解是必不可少的，并且可能会使新手感到困惑。一旦我们了解了字符串的管理方式，我们就可以在迭代字符串时避免常见错误。我们还将看到 Go 开发人员在使用或生成字符串时所犯的常见错误。此外，我们会看到有时我们可以直接使用 `[]byte`，避免额外的分配。最后，我们将看到如何避免从子字符串中创建泄漏的常见错误。本章的主要目标是通过介绍常见的字符串错误来帮助您理解 Go 中字符串的工作原理。

## 5.1 不被理解的符文概念

如果不讨论 Go 中符文的概念，我们就无法开始本章关于字符串的内容。正如您将在本章的下一节中看到的，这个概念是彻底理解字符串处理方式和避免常见错误的关键。但首先，在深入研究 Go 符文之前，我们需要确保我们在一些基本编程方面保持一致概念。

我们应该明白 charset 和 encoding 的区别：

* 顾名思义，字符集是一组字符。例如，Unicode 字符集包含 2^21 个字符。
* 编码是二进制字符列表的翻译。例如，UTF‑8 是一种编码标准，能够将所有 Unicode 字符编码为可 变字节数：从 1 到 4 个字节。

我们提到了字符来简化字符集定义。然而，在 Unicode 中，我们使用 *码位(code point)* 的概念来指代由单个值表示的项目。例如，`汉` 字符由 U+6C49 码位标识。使用 UTF‑8，`汉` 使用三个字节进行编码：0xE6、0xB1 和 0x89。

它为什么如此重要？因为在 Go 中，符文是一个 Unicode代码点。

同时，我们提到 UTF‑8 将字符编码为 1 到 4 个字节。因此，最多 32 位。这就是为什么在 Go 中，符文是 `int32`：

```go
type rune = int32
```

关于 UTF‑8 需要强调的其他一点：有些人认为 Go 字符串始终是 UTF‑8，但事实并非如此。让我们考虑以下示例：

```go
s := "hello"
```

我们为 `s` 分配了一个字符串文字（一个字符串常量）。在 Go 中，源代码以 UTF‑8 编码。因此，所有字符串文字都将使用 UTF‑8 编码为字节序列。但是，字符串是任意字节的序列；它不一定基于 UTF‑8。因此，当我们操作一个不是从字符串文字初始化的变量（例如，从文件系统读取）时，我们不一定期望它使用 UTF‑8 编码。

> **Note** `golang.org/x` 一个为标准库提供扩展的存储库，包含用于 UTF-16 和 UTF-32 的包.

让我们回到 `hello` 的例子。我们有一个由五个字符组成的字符串：h、e、l、l 和 o。这些简单的字符每个都使用一个字节进行编码。这就是为什么获取 `s` 的长度返回 5 的原因：

```go
s := "hello"
fmt.Println(len(s)) // 5
```

然而，一个字符并不总是被编码成一个字节。回到字符，我们提到使用 UTF‑8，这个字符被编码分成三个字节。我们可以通过以下示例对其进行验证：

```go
s := "汉" 
fmt.Println(len(s)) // 3
```

此示例打印 3 而不是打印 1。事实上，应用于字符串的 `len` 内置函数不返回字符数；它返回字节数。

相反，我们可以从字节列表中创建一个字符串。我们提到 `汉` 字符是使用三个字节编码的，0xE6、0xB1 和 0x89：

```go
s := string([]byte{0xE6, 0xB1, 0x89})
fmt.Printf("%s\n", s)
```

在这里，我们构建了一个由这 3 个字节组成的字符串。当我们打印字符串时，不是打印三个字符，而是只打印一个 `汉`。

总之：

* 字符集是一组字符，而编码描述了如何将字符集转换为二进制。
* 在 Go 中，字符串引用任意字节的不可变切片。
* Go 源代码使用 UTF‑8 编码。因此，所有字符串文字都是 UTF‑8 字符串。然而，由于字符串可以包含任意字节，如果它是从其他地方（不是源代码）获得的，则不能保证它基于 UTF‑8 编码。
* 符文对应于 Unicode 代码点概念，表示由单个值表示的项目。
* 使用 UTF‑8，Unicode 代码点可以编码为一到四个字节。
* 在 Go 中对字符串使用 `len` 返回字节数，不是符文的数量。

牢记这些概念是必不可少的，就像在 Go 中无处不在的符文一样。让我们看看这个知识的具体应用，其中包含与字符串迭代相关的常见错误。