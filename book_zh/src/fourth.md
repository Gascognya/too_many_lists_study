# 一个BadSafe的Deque实现

现在, 你已经看过了 Rc 也听过了内部可变性, 这给了我们一些有趣的启示... 
也许我们可以把它应用到 Rc. 在这种情况下, 或许我们可以完全安全的实现一个双向链表!

在这个过程中, 我们将不断熟悉 *内部可变现*, 并可能艰难的认识到, 安全并不代表正确. 
双向链表很难, 我们可能会经常犯错.

建一个新文件 `fourth.rs`:

```rust ,ignore
// in lib.rs

pub mod first;
pub mod second;
pub mod third;
pub mod fourth;
```

声明: 这一章将是一个非常糟糕的思路, 请抱着批判的态度来学习.
