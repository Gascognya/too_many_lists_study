# 内存布局

好, 回到内存布局的设计上.

对于持久化链表最重要的是你可以自由操纵tail:

举个例子, 下列复用情况在持久化链表中是非常常见的:

```text
list1 = A -> B -> C -> D
list2 = tail(list1) = B -> C -> D
list3 = push(list2, X) = X -> B -> C -> D
```

但是我们希望它在内存中是这样的:

```text
list1 -> A ---+
              |
              v
list2 ------> B -> C -> D
              ^
              |
list3 -> X ---+
```

这里不能用 Box, 因为 `B` 的所有权是 *共享* 的. 
谁来负责释放它? 如果我 drop 掉 list2, 它应该释放掉 B 么? 对于 Box 来说我们当然希望如此!

函数式编程语言 -- 事实上, 几乎其他所有语言 -- 通过使用 *GC* (garbage collection, 垃圾回收) 来摆脱这个问题. 
有了GC的魔法, B 将在不被人需要的时候释放.

这些语言可以通过 tracing GC, 搜寻整个运行时内存, 并自动找出哪些是垃圾. 这是 Rust 所不具备的. 
作为替代, *引用计数* 是 Rust 至今所拥有的全部了. 引用计数可以被认为是一个非常简单的GC. 
在一些时候, 它比 tracing GC 更轻量级, 然而当你尝试 "循环引用" 时, 它会崩溃掉.
但幸运的是, 我们这个例子用不会出现循环引用 (你可以尝试证明一下 -- 当然我不会).

>(译者: 这里谈到了关于GC的一些概念, 也谈到了两类常见GC算法, Tracing GC 和 引用计数.)

>引用计数的优点的即时高效, 在计数归零的一瞬间便会释放掉. 但缺点是会发生循环引用, 即A中引用了B, B中引用了A. 
>会导致GC失灵. 常见的引用计数GC是CPython, 但CPython为了解决循环引用, GC中实际上后来也加入了一些Tracing算法.

>Tracing GC虽然不会发生循环引用. 但也存在一些问题. 例如并不能像引用计数那样及时释放, 或者会发生STW(Stop The World).

所以我们该如何使用引用计数GC? `Rc`! Rc 和 Box 类似, 但是我们可以复制它, 只有它的所有相关衍生都 drop 后, 它才会释放. 
不幸的是, 这灵活性也带来了严重的代价: 我们只能对其内部进行共享引用. 这意味着我们不能真正的从链表中获取到数据, 也不能改变它们.

我们的内存布局会是什么样子呢? 让我们先来看看之前的:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

我们可以直接把 Box 换成 Rc 么?

```rust ,ignore
// in third.rs

pub struct List<T> {
    head: Link<T>,
}

type Link<T> = Option<Rc<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

```text
cargo build

error[E0412]: cannot find type `Rc` in this scope
 --> src/third.rs:5:23
  |
5 | type Link<T> = Option<Rc<Node<T>>>;
  |                       ^^ not found in this scope
help: possible candidate is found in another module, you can import it into scope
  |
1 | use std::rc::Rc;
  |
```

炸了. 和我们在可变链表中的完全不一样, Rc 是如此差劲, 以至于没有被默认隐式导入到Rust作用域中. 多么失败啊.

```rust ,ignore
use std::rc::Rc;
```

```text
cargo build

warning: field is never used: `head`
 --> src/third.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

看起来似乎是允许的. Rust 仍旧写的很简单. 我打赌我们把所有 Box 换成 Rc 就可以手工!

...

不, 实际上不能.
