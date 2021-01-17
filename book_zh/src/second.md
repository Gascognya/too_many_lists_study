# 一个不错的Stack实现

在上一章中, 我们写了个最小实现的单向链表Stack. 然而, 也有一些设计决策使它变得非常糟糕. 让我们修缮它. 为此我们将:

* 不造轮子
* 使List能够处理任何元素
* 添加peek方法
* 使我们的List可迭代

在这个过程中我们将学会

* 使用高级的Option
* 泛型
* 生命周期
* 迭代器

将我们新建名为 `second.rs` 的rs文件:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
```

复制 `first.rs` 中的所有内容到其中.
