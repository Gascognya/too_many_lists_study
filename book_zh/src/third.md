# 一个可持久的Stack实现

好了, 我们已经掌握了 可变单链表 的艺术.

让我们的眼光从 *单* 所有权移向 *共享* 所有权, 通过写一个 *可持久的* 不可变单向链表. 
这恰好是函数式编程熟悉的 list. 你可以获得一个 head *或* tail , 或者把某一链表的 head 放到另一个链表的 tail...
再或者... 基本上是这样. 

通过这个过程, 我们将逐渐熟悉 Rc 和 Arc, 这将有助于我们的下一个链表实现.

让我们创建 `third.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
```

这次我们不需要复制. 让我们从零开始.
