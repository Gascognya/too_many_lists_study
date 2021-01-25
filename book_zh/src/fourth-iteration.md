# Iteration

让我们来试试 Iter 这个坏家伙.

## IntoIter

IntoIter, 总是很简单, 包装链表, 然后一直 `pop` 就好了:

```rust ,ignore
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<T> {
        self.0.pop_front()
    }
}
```

但我们有了一个有趣的新进展. 我们之前的链表都是单向的, 现在是一个, 而现在是双向的. 
从前到后有什么特别之处? 如果有人想要在另一个方向迭代呢?

Rust 提供了答案: `DoubleEndedIterator`. 
DoubleEndedIterator继承自 Iterator (意味着所有的 DoubleEndedIterator 都是迭代器) 需要一个新方法: `next_back`. 
它具有与 `next` 完全相同的特性, 但它应该从另一端产生元素. DoubleEndedIterator 的语义对我们来说非常方便: 
迭代器变成了deque. 您可以从前面和后面使用元素, 直到两端收敛, 此时迭代器为空.

就像 Iterator 和 `next` 一样, 事实证明 `next_back` 并不是 DoubleEndedIterator 的使用方真正该关心的问题. 
相反, 这个接口最好的地方在于它公开了 `rev` 方法, 该方法将迭代器封装起来, 生成一个新迭代器, 以倒序生成元素. 它的语义相当简单: 
在反向迭代器上调用 `next` 就是调用 `next_back`.

我们试着来实现这个API:

```rust ,ignore
impl<T> DoubleEndedIterator for IntoIter<T> {
    fn next_back(&mut self) -> Option<T> {
        self.0.pop_back()
    }
}
```

做个测试:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push_front(1); list.push_front(2); list.push_front(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next_back(), Some(1));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next_back(), None);
    assert_eq!(iter.next(), None);
}
```


```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 11 tests
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test fourth::test::into_iter ... ok
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test third::test::iter ... ok
test third::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 11 passed; 0 failed; 0 ignored; 0 measured

```

很好.

## Iter

Iter 就比较麻烦了. 因为涉及到 `Ref`! 因为 Ref 的存在, 我们不能存储 `&Node`.
相反, 我们可以试着存储 `Ref<Node>`s:

```rust ,ignore
pub struct Iter<'a, T>(Option<Ref<'a, Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.borrow()))
    }
}
```

```text
> cargo build

```

还行. 实现 `next` 有点麻烦, 但我认为它与之前的链表的 IterMut 的基本逻辑相同, 除了涉及到RefCell:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = Ref<'a, T>;
    fn next(&mut self) -> Option<Self::Item> {
        self.0.take().map(|node_ref| {
            self.0 = node_ref.next.as_ref().map(|head| head.borrow());
            Ref::map(node_ref, |node| &node.elem)
        })
    }
}
```

```text
cargo build

error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:155:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   -------- borrow is only valid in the closure body
    |             |
    |             reference to `node_ref` escapes the closure body here

error[E0505]: cannot move out of `node_ref` because it is borrowed
   --> src/fourth.rs:156:22
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- lifetime `'1` appears in the type of `self`
154 |         self.0.take().map(|node_ref| {
155 |             self.0 = node_ref.next.as_ref().map(|head| head.borrow());
    |             ------   -------- borrow of `node_ref` occurs here
    |             |
    |             assignment requires that `node_ref` is borrowed for `'1`
156 |             Ref::map(node_ref, |node| &node.elem)
    |                      ^^^^^^^^ move out of `node_ref` occurs here
```

不.

`node_ref` 活得不够久. 不像普通引用, Rust 不让我们们将 Ref 分离开. 
从 `head.borrow()` 中得到的Ref只允许比 `node_ref` 活得久, 但是我们在 `Ref::map` 调用中drop了它.

比较巧的是, 在我写这个的时候, 我们想要的函数实际上两天前就稳定下来了. 这意味着它还需要几个月的时间才能发布稳定版. 
所以让我们使用最新的 nightly 进行 build:

```rust ,ignore
pub fn map_split<U, V, F>(orig: Ref<'b, T>, f: F) -> (Ref<'b, U>, Ref<'b, V>) where
    F: FnOnce(&T) -> (&U, &V),
    U: ?Sized,
    V: ?Sized,
```

让我们试试 ...

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = next.as_ref().map(|head| head.borrow());

        elem
    })
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0521]: borrowed data escapes outside of closure
   --> src/fourth.rs:159:13
    |
153 |     fn next(&mut self) -> Option<Self::Item> {
    |             --------- `self` is declared here, outside of the closure body
...
159 |             self.0 = next.as_ref().map(|head| head.borrow());
    |             ^^^^^^   ---- borrow is only valid in the closure body
    |             |
    |             reference to `next` escapes the closure body here
```

哦. 我们需要再次 `Ref::Map` 活得正确的生命周期. 但是 `Ref::Map`
返回一个 `Ref` 而我们需要 `Option<Ref>`, 但是我们需要通过 Ref 来映射我们的 Option...


??????

```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.0.take().map(|node_ref| {
        let (next, elem) = Ref::map_split(node_ref, |node| {
            (&node.next, &node.elem)
        });

        self.0 = if next.is_some() {
            Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
        } else {
            None
        };

        elem
    })
}
```

```text
error[E0308]: mismatched types
   --> src/fourth.rs:162:22
    |
162 |                 Some(Ref::map(next, |next| &**next.as_ref().unwrap()))
    |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `fourth::Node`, found struct `std::cell::RefCell`
    |
    = note: expected type `std::cell::Ref<'_, fourth::Node<_>>`
               found type `std::cell::Ref<'_, std::cell::RefCell<fourth::Node<_>>>`
```

Oh. Right. There's multiple RefCells. The deeper we walk into the list, the more
nested we become under each RefCell. We would need to maintain, like, a stack of
Refs to represent all the outstanding loans we're holding, because if we stop
looking at an element we need to decrement the borrow-count on every RefCell that
comes before it.................

I don't think there's anything we can do here. It's a dead end. Let's try
getting out of the RefCells.

What about our `Rc`s. Who said we even needed to store references?
Why can't we just Clone the whole Rc to get a nice owning handle into the middle
of the list?

```rust
pub struct Iter<T>(Option<Rc<Node<T>>>);

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter(self.head.as_ref().map(|head| head.clone()))
    }
}

impl<T> Iterator for Iter<T> {
    type Item =
```

Uh... Wait what do we return now? `&T`? `Ref<T>`?

No, none of those work... our Iter doesn't have a lifetime anymore! Both `&T`
and `Ref<T>` require us to declare some lifetime up front before we get into
`next`. But anything we manage to get out of our Rc would be borrowing the
Iterator... brain... hurt... aaaaaahhhhhh

Maybe we can... map... the Rc... to get an `Rc<T>`? Is that a thing? Rc's docs
don't seem to have anything like that. Actually someone made [a crate][own-ref]
that lets you do that.

But wait, even if we do *that* then we've got an even bigger problem: the
dreaded spectre of iterator invalidation. Previously we've been totally immune
to iterator invalidation, because the Iter borrowed the list, leaving it totally
immutable. However if our Iter was yielding Rcs, they wouldn't borrow the list
at all! That means people can start calling `push` and `pop` on the list while
they hold pointers into it!

Oh lord, what will that do?!

Well, pushing is actually fine. We've got a view into some sub-range of the
list, and the list will just grow beyond our sights. No biggie.

However `pop` is another story. If they're popping elements outside of our
range, it should *still* be fine. We can't see those nodes so nothing will
happen. However if they try to pop off the node we're pointing at... everything
will blow up! In particular when they go to `unwrap` the result of the
`try_unwrap`, it will actually fail, and the whole program will panic.

That's actually pretty cool. We can get tons of interior owning pointers into
the list and mutate it at the same time *and it will just work* until they
try to remove the nodes that we're pointing at. And even then we don't get
dangling pointers or anything, the program will deterministically panic!

But having to deal with iterator invalidation on top of mapping Rcs just
seems... bad. `Rc<RefCell>` has really truly finally failed us. Interestingly,
we've experienced an inversion of the persistent stack case. Where the
persistent stack struggled to ever reclaim ownership of the data but could get
references all day every day, our list had no problem gaining ownership, but
really struggled to loan our references.

Although to be fair, most of our struggles revolved around wanting to hide the
implementation details and have a decent API. We *could* do everything fine
if we wanted to just pass around Nodes all over the place.

Heck, we could make multiple concurrent IterMuts that were runtime checked to
not be mutable accessing the same element!

Really, this design is more appropriate for an internal data structure that
never makes it out to consumers of the API. Interior mutability is great for
writing safe *applications*. Not so much safe *libraries*.

Anyway, that's me giving up on Iter and IterMut. We could do them, but *ugh*.

[own-ref]: https://crates.io/crates/owning_ref
