# Peek 方法

好, 我们完成了 `push` 和 `pop` 方法. 我不想撒谎, 这有点情绪化. 
编译时的正确性是一种地狱般的毒品.

咱们做点简单的事冷静一下吧: 让我们只实现下 `peek_front`.
以前实现这个一直非常容易, 对吧?

是吧?

事实上, 我认为复制粘贴就可以了!

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

等下, 这次有点不一样.

```rust ,ignore
pub fn peek_front(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        // BORROW!!!!
        &node.borrow().elem
    })
}
```


```text
cargo build

error[E0515]: cannot return value referencing temporary value
  --> src/fourth.rs:66:13
   |
66 |             &node.borrow().elem
   |             ^   ----------^^^^^
   |             |   |
   |             |   temporary value created here
   |             |
   |             returns a value referencing data owned by the current function
```

我想砸了我的电脑.

这与单向链表逻辑完全一样. 会什么结果不同.

其中的答案, 便是我们这章的用意: RefCells 让一切都很糟糕. 直到现在, RefCells 还只是个讨厌的东西. 现在它将成为噩梦.

到底是为什么呢? 要理解内容, 我们要回到 `borrow` 的定义:

```rust ,ignore
fn borrow<'a>(&'a self) -> Ref<'a, T>
fn borrow_mut<'a>(&'a self) -> RefMut<'a, T>
```

在布局章节, 我们说过:

> RefCell 是运行时检查, 而不是静态检查.
> 如果你打破了规则, RefCell 会抛出 panic 让程序崩溃.
> 为什么它返回 Ref 和 RefMut ? 嗯, 它们基本和 `Rc` 一样, 仅仅为了出借. 
> 它们还会保持 RefCell 借用直到离开作用域. 我们稍后会讲到.

现在到时候了.

`Ref` 和 `RefMut` 分别实现了 `Deref` 和 `DerefMut` . 
所以大多数时间它们的行为与 `&T` 和 `&mut T` 完全一样. 
然而因为 trait 的工作原理, 返回的引用关联的是 Ref 的生命周期, 并不是 RefCell. 
这意味着你得保持Ref的存在和你对Ref引用一样长.

那我们该如何是好? 我们只想返回一个引用, 但是我们必须得保持 Ref 存活. 
但当我们从 `peek` 返回的时候, 函数结束了, `Ref` 超出了作用域.

😖

这貌似是个死局. 你不能完全封装 RefCells 的所有使用.

但... 如果我们放弃隐藏实现细节呢? 直接把 Ref 返回?

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        node.borrow()
    })
}
```

```text
> cargo build

error[E0412]: cannot find type `Ref` in this scope
  --> src/fourth.rs:63:40
   |
63 |     pub fn peek_front(&self) -> Option<Ref<T>> {
   |                                        ^^^ not found in this scope
help: possible candidates are found in other modules, you can import them into scope
   |
1  | use core::cell::Ref;
   |
1  | use std::cell::Ref;
   |
```

哦. 应该导入一下.


```rust ,ignore
use std::cell::{Ref, RefCell};
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/fourth.rs:64:9
   |
64 | /         self.head.as_ref().map(|node| {
65 | |             node.borrow()
66 | |         })
   | |__________^ expected type parameter, found struct `fourth::Node`
   |
   = note: expected type `std::option::Option<std::cell::Ref<'_, T>>`
              found type `std::option::Option<std::cell::Ref<'_, fourth::Node<T>>>`
```

Hmm... 没错. 我们拥有 `Ref<Node<T>>`, 但是需要 `Ref<T>`. 我们可以放弃全部的封装直接将它返回. 
我们也可以让事情变得复杂点, 将 `Ref<Node<T>>` 封装在新类型中, 只公开对 `&T` 的访问.

两种选择都不怎么好.

相反, 让我们找点有趣的:

```rust ,ignore
map<U, F>(orig: Ref<'b, T>, f: F) -> Ref<'b, U>
    where F: FnOnce(&T) -> &U,
          U: ?Sized
```

> 创造一个新的 Ref

是的: 就像 map 一个 Option, 你也可以 map 一个 Ref.

```rust ,ignore
pub fn peek_front(&self) -> Option<Ref<T>> {
    self.head.as_ref().map(|node| {
        Ref::map(node.borrow(), |node| &node.elem)
    })
}
```

```text
> cargo build
```

很好

让我们来做一些测试.

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert!(list.peek_front().is_none());
    list.push_front(1); list.push_front(2); list.push_front(3);

    assert_eq!(&*list.peek_front().unwrap(), &3);
}
```


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 10 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::basics ... ok
test fourth::test::peek ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test third::test::basics ... ok
test second::test::peek ... ok
test second::test::iter ... ok
test third::test::iter ... ok

test result: ok. 10 passed; 0 failed; 0 ignored; 0 measured

```

很棒!
