# Basics

我们已经知道了 Rust 的很多基础知识, 所以我们可以先做些简单的事.

构造器的话, 我们可以直接复制:

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }
}
```

`push` 和 `pop` 已经没意义了. 取而代之的是 `append` 和 `tail`, 它们提供了类似的功能.

让我们从 append 开始. 它接受一个链表和一个元素, 然后返回一个链表. 例如在可变链表的情况下, 我们想要创建一个新 node, 旧的链表将会变成 `next` 的值. 唯一新奇的事情是如何获得下一个值, 因为我们不允许改变任何东西.

我们祈祷的答案是 Clone trait. Clone 被很多类型实现了, 它提供了一种获取副本的通用方法. 它就像 C++ 中的 copy constructor, 但是它从来不会被隐式调用.

Rc 经常使用 Clone 的方式增加引用计数. 我们不需要将 head move 到子链表中, 我们只需要克隆旧链表的 head. 
Option 提供了一个我们想要的 Clone 实现.

让我们来试试看:

```rust ,ignore
pub fn append(&self, elem: T) -> List<T> {
    List { head: Some(Rc::new(Node {
        elem: elem,
        next: self.head.clone(),
    }))}
}
```

```text
> cargo build

warning: field is never used: `elem`
  --> src/third.rs:10:5
   |
10 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/third.rs:11:5
   |
11 |     next: Link<T>,
   |     ^^^^^^^^^^^^^
```

哇哦, Rust 在字段是否使用方面真的很严格. 它告诉我们这些字段从未被使用! 目前为止看起来还不错.

`tail` 是这个的逻辑逆操作. 它接受一个链表, 然后返回这个链表去掉第一个元素后的部分. 我们需要做的仅仅是, clone 链表的第二个元素(如果存在的话). 让我们来试试:

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().map(|node| node.next.clone()) }
}
```

```text
cargo build

error[E0308]: mismatched types
  --> src/third.rs:27:22
   |
27 |         List { head: self.head.as_ref().map(|node| node.next.clone()) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `std::rc::Rc`, found enum `std::option::Option`
   |
   = note: expected type `std::option::Option<std::rc::Rc<_>>`
              found type `std::option::Option<std::option::Option<std::rc::Rc<_>>>`
```

哦, 我们搞砸了. `map` 期待我们的闭包返回一个 Y, 但这里实际上返回了一个 `Option<Y>`.幸运的是我们可以使用 `and_then` 来获取 Option.

```rust ,ignore
pub fn tail(&self) -> List<T> {
    List { head: self.head.as_ref().and_then(|node| node.next.clone()) }
}
```

```text
> cargo build

```

很好.

现在我们有了 `tail`, 我们也许需要提供 `head`, 它返回第一个元素的引用. 好比可变链表中的 `peek` :

```rust ,ignore
pub fn head(&self) -> Option<&T> {
    self.head.as_ref().map(|node| &node.elem )
}
```

```text
> cargo build

```

很好.

这些功能足够了, 我们来测试一下:


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;

    #[test]
    fn basics() {
        let list = List::new();
        assert_eq!(list.head(), None);

        let list = list.append(1).append(2).append(3);
        assert_eq!(list.head(), Some(&3));

        let list = list.tail();
        assert_eq!(list.head(), Some(&2));

        let list = list.tail();
        assert_eq!(list.head(), Some(&1));

        let list = list.tail();
        assert_eq!(list.head(), None);

        // Make sure empty tail works
        let list = list.tail();
        assert_eq!(list.head(), None);

    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::into_iter ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test third::test::basics ... ok

test result: ok. 5 passed; 0 failed; 0 ignored; 0 measured

```

很好!

Iter 也与我们在可变列表时的实现方式相同:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &**node);
            &node.elem
        })
    }
}
```

```rust ,ignore
#[test]
fn iter() {
    let list = List::new().append(1).append(2).append(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 7 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 6 passed; 0 failed; 0 ignored; 0 measured

```

注意, 我们不能为这种类型实现IntoIter或IterMut, 我们只有对元素的共享访问.
