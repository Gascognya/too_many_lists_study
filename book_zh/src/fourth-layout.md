# 内存布局

我们设计中的关键在于 `RefCell` 类型. RefCell 的核心是一对方法:

```rust ,ignore
fn borrow(&self) -> Ref<'_, T>;
fn borrow_mut(&self) -> RefMut<'_, T>;
```

`borrow` 和 `borrow_mut` 的正如 `&` 和 `&mut`:
你可以多次调用 `borrow`, 但 `borrow_mut` 需要独占.

RefCell 是运行时检查, 而不是静态检查 (意味着多次调用borrow_mut虽然不允许, 但不会有静态提示).
如果你打破了规则, RefCell 会抛出 panic 让程序崩溃.
为什么它返回 Ref 和 RefMut ? 嗯, 它们基本和 `Rc` 一样, 仅仅为了出借. 它们还会保持 RefCell 借用直到离开作用域. 
我们稍后会将到.

现在通过 Rc 和 RefCell 我们可以造就... 
一种令人难以置信的冗长、无处不在的可变垃圾收集语言且不能处理循环引用! Y-yaaaaay...

好, 我们想要双链. 这意味着每个 node 需要指向前一个和后一个 node. 
此外, 链表也应该包含指向头和尾的指针. 这让我们能在双端快速插入和删除.

所以我们也许需要:

```rust ,ignore
use std::rc::Rc;
use std::cell::RefCell;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>,
}

type Link<T> = Option<Rc<RefCell<Node<T>>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
    prev: Link<T>,
}
```

```text
> cargo build

warning: field is never used: `head`
 --> src/fourth.rs:5:5
  |
5 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fourth.rs:6:5
  |
6 |     tail: Link<T>,
  |     ^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fourth.rs:12:5
   |
12 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/fourth.rs:13:5
   |
13 |     next: Link<T>,
   |     ^^^^^^^^^^^^^

warning: field is never used: `prev`
  --> src/fourth.rs:14:5
   |
14 |     prev: Link<T>,
   |     ^^^^^^^^^^^^^
```

很多 dead code warnings, 但是它成功构建了! 让我们是这样用一下.
