# 创建短网址系统

> 原文：[Create a TinyURL System](http://blog.gainlo.co/index.php/2016/03/08/system-design-interview-question-create-tinyurl-system/)

> 译者：[飞龙](https://github.com/wizardforcel)

> 协议：[CC BY-NC-SA 4.0](http://creativecommons.org/licenses/by-nc-sa/4.0/)

> 自豪地采用[谷歌翻译](https://translate.google.cn/)

如果你已经开始准备系统设计面试，你一定听说过一个最流行的问题 - 创建一个短网址系统。

本周，Gainlo 团队就这个面试问题集思广益。我想总结一下，如何用概要思想来分析这个问题。

如果你是系统设计面试新手，我强烈建议你在系统设计面试之前首先了解一下你需要了解的八件事情，其中​​提供了可帮助你入门的一般性策略。

## 问题

如何创建一个短网址系统？

如果你对短网址不熟悉，我在这里简要解释一下。基本上，短网址是 URL 缩短服务，一个 Web 服务，提供简短别名来重定向长 URL 。还有很多其他类似的服务，比如 Google URL Shortener，Bitly 等等。

例如，URL <http://blog.gainlo.co/index.php/2015/10/22/8-things-you-need-to-know-before-system-design-interviews/> 很长，很难记住，短网址可以为它创建一个别名 - <http://tinyurl.com/j7ve58y>。如果你点击别名，它会将你重定向到原始网址。

所以如果你设计这个系统，允许人们输入 URL，生成较短的别名 URL，你会怎么做？

## 概要思想

让我们以基础和概要解决方案开始，然后再继续优化。

乍一看，每个长 URL 和相应的别名形成一个键值对。 我希望你能马上思考与哈希有关的东西。

因此，问题可以这样简化 - 给定一个 URL，我们如何找到哈希函数`F`，它将 URL 映射到一个简短别名：

```
F(URL) = alias
```

并满足以下条件：

每个 URL 只能映射到唯一的别名
每个别名都可以轻松地映射回唯一的 URL

第二个条件是运行其实的核心，系统应该通过别名查找并快速重定向到相应的 URL。

## 基本解决方案

为了方便起见，我们可以假设别名是`http://tinyurl.com/<alias_hash>`，`alias_hash`是固定长度的字符串。

如果长度为 7，包含`[A-Z, a-z, 0-9]`，则可以提供`62 ^ 7 ~= 3500 亿`个 URL。 据说在写这篇文章的时候，有大约 6.44 亿个网址。

首先，我们将所有的映射存储在一个数据库中。 一个简单的方法是使用`alias_hash`作为每个映射的 ID，可以生成一个长度为 7 的随机字符串。

所以我们可以先存储`<ID，URL>`。 当用户输入一个长 URL `http://www.gainlo.co`时，系统创建一个随机的 7 个字符的字符串，如`abcd123`作为 ID，并插入条目`<"abcd123", "http://www.gainlo.co">`进入数据库。

在运行期间，当有人访问`http://tinyurl.com/abcd123`时，我们通过 ID `abcd123`查找并重定向到相应的 URL `http://www.gainlo.co`。

## 性能 VS 灵活性

这个问题有很多后续问题。我想在这里进一步讨论的一件事是，通过使用GUID（全局唯一标识符）作为条目 ID，在这个问题中，与自增 ID 相比利弊是什么？

如果你深入了解插入/查询过程，你会注意到使用随机字符串作为 ID 可能会牺牲一点点性能。更具体地说，当你已经有数百万条记录时，插入可能是很昂贵的。由于 ID 不是连续的，因此每次插入新记录时，数据库都需要查看该 ID 的正确页面。但是，使用增量 ID 时，插入可以更容易 - 只需转到最后一页。

所以优化这个的一种方法是使用增量 ID。每次插入一个新的 URL 时，我们都会为新条目增加 1。我们还需要一个散列函数，将每个整数 ID 映射到一个 7 个字符的字符串。如果我们把每个字符串看作一个 62 进制的数字，映射应该很容易（当然，还有其他方法）。

另一方面，使用增量 ID 将使得映射更不灵活。例如，如果系统允许用户设置自定义短网址，显然 GUID 解决方案更容易，因为对于任何自定义短网址，我们可以计算相应的哈希作为条目 ID。

注意：在这种情况下，我们可能不使用随机生成的键，而是使用更好的散列函数将任何短网址映射为 ID，例如，一些传统的散列函数，如 CRC32，SHA-1 等。

## 开销

我很少询问如何评估系统的开销。对于插入/查询，我们已经在上面讨论过了。所以我会更关注存储开销。

每个条目存储为`<ID，URL>`，其中 ID 是一个 7 个字符的字符串。假设最大 URL 长度为 2083 个字符，则每个条目需要`7 * 4 + 2083 * 4 = 8.4 KB`。如果我们存储一百万个 URL 映射，我们需要大约 8.4G 的存储空间。

如果我们考虑数据库索引的大小，我们也可能存储其他信息，比如用户 ID，日期等等，这肯定需要更多的存储空间。

## 多台机器

显然，当系统发展到一定规模时，单台机器不能存储所有的映射。我们如何扩展多个实例？

更普遍的问题是如何在多个机器上存储散列映射。如果你知道分布式键值存储，你应该知道这可能是一个非常复杂的问题。我只会在这里讨论概要思想，如果你对所有这些细节感兴趣，我建议你阅读 [Dynamo 文章：亚马逊的高可用键值存储](http://www.allthingsdistributed.com/files/amazon-dynamo-sosp2007.pdf)。

简而言之，如果要在多个实例中存储大量的键值对，则需要设计查找算法，以便为给定的查找键寻找相应的机器。

例如，如果传入的短名称是`http://tinyurl.com/abcd123`，则基于键`abcd123`，系统应该知道哪个机器存储了数据库，它包含这个键的条目。这与数据库分片完全一样。

一种常见的方法是让机器充当代理，负责根据查找键，将请求分派到相应的后端存储器。后端存储器是实际上存储映射的数据库。他们可以通过各种方式拆分，如使用`hash(key) % 1024`将映射划分到 1024 个存储器中。

有很多细节可以使系统变得复杂，我只是在这里举几个例子：

+   复制。数据存储可能会因各种随机原因而崩溃，因此常见的解决方案是每个数据库都有多个副本。这里可能有很多问题：如何复制实例？如何快速恢复？如何保持读/写一致？
+   重新分片。当系统扩展到另一个级别时，原始分片算法可能无法正常工作。我们可能需要使用新的散列算法来对系统重新分片。如何在保持系统运行的同时，对数据库重新分片，可能是一个极其困难的问题。
+   并发。可以有多个用户同时插入相同的 URL 或编辑相同的别名。用一台机器，你可以用一个锁来控制它。但是，当你扩展到多个实例时，情况会变得更加复杂。
 

## 总结

我觉得你已经意识到这个问题乍一看并不复杂，但是当你深入挖掘更多的细节，特别是考虑规模问题的时候，会遇到很多后续问题。

我们在之前的文章中说过，使用的具体技术并不重要。真正重要的是如何解决这个问题的概要思想。这就是我不提 Redis，RebornDB 等东西的原因。

再次，有无限的方式用于进一步扩大这个问题。我想用这篇文章作为起点，来使你熟悉系统设计面试。

如果你觉得这篇文章有帮助，请分享给你的朋友，我会很感谢。 你也可以在这里查看更多的系统设计面试问题和分析。
