# 拆解

`pop_front` 基础逻辑和 `push_front` 类似, 但是相反. 让我们试试:

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    // need to take the old head, ensuring it's -2
    self.head.take().map(|old_head| {                         // -1 old
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {                               // -1 new
                // not emptying list
                new_head.borrow_mut().prev.take();            // -1 old
                self.head = Some(new_head);                   // +1 new
                // total: -2 old, +0 new
            }
            None => {
                // emptying list
                self.tail.take();                             // -1 old
                // total: -2 old, (no new)
            }
        }
        old_head.elem
    })
}
```

```text
> cargo build

error[E0609]: no field `elem` on type `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>`
  --> src/fourth.rs:64:22
   |
64 |             old_head.elem
   |                      ^^^^ unknown field
```

是 *RefCells*. 我又要 `borrow_mut` 了...

```rust ,ignore
pub fn pop_front(&mut self) -> Option<T> {
    self.head.take().map(|old_head| {
        match old_head.borrow_mut().next.take() {
            Some(new_head) => {
                new_head.borrow_mut().prev.take();
                self.head = Some(new_head);
            }
            None => {
                self.tail.take();
            }
        }
        old_head.borrow_mut().elem
    })
}
```

```text
cargo build

error[E0507]: cannot move out of borrowed content
  --> src/fourth.rs:64:13
   |
64 |             old_head.borrow_mut().elem
   |             ^^^^^^^^^^^^^^^^^^^^^^^^^^ cannot move out of borrowed content
```

*sigh*

> cannot move out of borrowed content

哦... 看来Box *真的* 宠坏我们了. `borrow_mut` 只会给我们一个 `&mut Node<T>`, 但我们不能 move 它!

我们需要能传入 `RefCell<T>` 获得 `T`. 让我们看看 
[文档][refcell] 里的有关内容:

> `fn into_inner(self) -> T`
>
> 消耗掉 RefCell, 返回其中的值.

这个看起来靠谱!

```rust ,ignore
old_head.into_inner().elem
```

```text
> cargo build

error[E0507]: cannot move out of an `Rc`
  --> src/fourth.rs:64:13
   |
64 |             old_head.into_inner().elem
   |             ^^^^^^^^ cannot move out of an `Rc`
```

坏了. `into_inner` 想要将 RefCell move, 但我们做不到, 因为它在 `Rc` 中. 
正如我们前一章所说, `Rc<T>` 只能让我们获取其内部的共享引用. 
这是有道理的, 因为这是 *所有引用计数指针* 的要点: 它们是共享的!

当我们想要为引用计数列表实现Drop时也是这个问题, 解决方式是: `Rc::try_unwrap`, 它将移出Rc中的内容, 当引用计数为1时.

```rust ,ignore
Rc::try_unwrap(old_head).unwrap().into_inner().elem
```

`Rc::try_unwrap` 返回一个 `Result<T, Rc<T>>`. 
Results 基本上是一个广义的 `Option`, 它的 `None` 情况有与之相关的数据, 当你试图解开 `Rc`. 
因为我们不关心它失败的情况(如果我们正确地编写了程序, 它 *必须* 成功), 我们只对它调用 `unwrap` .

无论如何, 让我们看看接下来我们会遇到什么编译错误(让我们面对它, 肯定会有一个).

```text
> cargo build

error[E0599]: no method named `unwrap` found for type `std::result::Result<std::cell::RefCell<fourth::Node<T>>, std::rc::Rc<std::cell::RefCell<fourth::Node<T>>>>` in the current scope
  --> src/fourth.rs:64:38
   |
64 |             Rc::try_unwrap(old_head).unwrap().into_inner().elem
   |                                      ^^^^^^
   |
   = note: the method `unwrap` exists but the following trait bounds were not satisfied:
           `std::rc::Rc<std::cell::RefCell<fourth::Node<T>>> : std::fmt::Debug`
```

 Result 的 `unwrap` 需要你可以打印错误情况下的debug信息.
`RefCell<T>` 只有当 `T` 实现了Debug时, 它才能实现. `Node` 没有实现Debug.

我们不这么做, 而是将结果转换为带 `ok` 的 Option:

```rust ,ignore
Rc::try_unwrap(old_head).ok().unwrap().into_inner().elem
```

求你了.

```text
cargo build

```

好.

我们做到了.

我们实现了 `push` 和 `pop`.

让我们把之间链表的basic test 抄过来 (因为它的内容我们都实现了):

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop_front(), None);

        // Populate list
        list.push_front(1);
        list.push_front(2);
        list.push_front(3);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(3));
        assert_eq!(list.pop_front(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push_front(4);
        list.push_front(5);

        // Check normal removal
        assert_eq!(list.pop_front(), Some(5));
        assert_eq!(list.pop_front(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop_front(), Some(1));
        assert_eq!(list.pop_front(), None);
    }
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 9 tests
test first::test::basics ... ok
test fourth::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::basics ... ok
test fifth::test::iter_mut ... ok
test third::test::basics ... ok
test second::test::iter ... ok
test third::test::iter ... ok
test second::test::into_iter ... ok

test result: ok. 9 passed; 0 failed; 0 ignored; 0 measured

```


现在我们可以从链表中移除东西, 我们可以实现 Drop.
这一次 Drop 在概念上更有趣一些. 之前的 Drop 是为了处理无限递归, 但这次不同.

`Rc` 不能处理循环引用. 如果存在循环引用, 所有东西都会让其他东西保持存活. 一个双向链表, 事实证明它是由小循环组成的长链!
所以当我们删除链表时, 仅仅是两端的引用变为 1... 然后什么也不会发生.如果我们的链表只包含一个节点, 那并没有问题. 但如果链表包含多个元素. 它也理应正常工作.

正如我们看到的, 移除元素比较痛苦. 所以最简单的办法是 `pop` 到没有任何东西:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        while self.pop_front().is_some() {}
    }
}
```

```text
cargo build

```

(我们其实可以使用可变堆栈来实现这一点, 但捷径是为那些理解事物的人准备的!)

我们可以看看实现 `_back` 版的 `push` 和 `pop`, 但它们仅仅是复制粘贴. 我们将先考虑更有趣的小心!


[refcell]: https://doc.rust-lang.org/std/cell/struct.RefCell.html
[multirust]: https://github.com/brson/multirust
[downloads]: https://www.rust-lang.org/install.html
