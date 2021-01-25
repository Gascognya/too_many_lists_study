# 一个Unsafe的Queue实现

引用计数器的内部可变性有些失控了. Rust 肯定不喜欢你这样做吧? 嗯, 是 也 不是. 
Rc 和 Refcell 可以处理简单情况, 但它们也可能非常笨拙. 
特别是当你想隐藏实现细节. 我觉得一定有更好的解决方法!

在这一章, 我们将回到单向链表, 实现一个单链队列, 这将涉及到 *裸指针* 和 *Unsafe*.

让我们建一个 `fifth.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
pub mod fifth;
```

我们的代码由 second.rs 派生, 因为在链表的世界中, 队列是堆栈的扩充. 不过, 我们还是要从头开始, 因为我们还需要解决一些关于布局之类的基本问题.
