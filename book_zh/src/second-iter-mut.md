# IterMut

说实话, IterMut 很狂野. 虽然说这句话本身就挺狂野的; 它和 Iter 差不多!

从语义上来讲, 是的, 但是共享和可变引用的本质意味着Iter是微不足道的.

先来看看我们如何为 Iter 实现 Iterator:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> { /* stuff */ }
}
```

让我们观察一下如果标上生命周期后的样子:

```rust ,ignore
impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next<'b>(&'b mut self) -> Option<&'a T> { /* stuff */ }
}
```

`next` 的签名 在输入和输出的生命周期之间 没有建立约束! 这代表着什么? 这代表着我们可以无限次的调用 `next`!


```rust ,ignore
let mut list = List::new();
list.push(1); list.push(2); list.push(3);

let mut iter = list.iter();
let x = iter.next().unwrap();
let y = iter.next().unwrap();
let z = iter.next().unwrap();
```

好!

对于共享引用来说这是 *绝对* 没问题的, 因为你可以同时拥有大量共享引用. 
然而可变引用 *不能* 共存, 它们是独享的.

其导致的结果是, 使用安全的代码来实现 IterMut 显然更加困难 (我们甚至还没有了解它的含义……). 
令人惊讶的是, IterMut 实际上有多重安全的实现方式!

我们将 把 Iter 代码中所有东西都改成可变的:

```rust ,ignore
pub struct IterMut<'a, T> {
    next: Option<&'a mut Node<T>>,
}

impl<T> List<T> {
    pub fn iter_mut(&self) -> IterMut<'_, T> {
        IterMut { next: self.head.as_mut().map(|node| &mut **node) }
    }
}

impl<'a, T> Iterator for IterMut<'a, T> {
    type Item = &'a mut T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_mut().map(|node| &mut **node);
            &mut node.elem
        })
    }
}
```

```text
> cargo build
error[E0596]: cannot borrow `self.head` as mutable, as it is behind a `&` reference
  --> src/second.rs:95:25
   |
94 |     pub fn iter_mut(&self) -> IterMut<'_, T> {
   |                     ----- help: consider changing this to be a mutable reference: `&mut self`
95 |         IterMut { next: self.head.as_mut().map(|node| &mut **node) }
   |                         ^^^^^^^^^ `self` is a `&` reference, so the data it refers to cannot be borrowed as mutable

error[E0507]: cannot move out of borrowed content
   --> src/second.rs:103:9
    |
103 |         self.next.map(|node| {
    |         ^^^^^^^^^ cannot move out of borrowed content
```

这里标了两种不同的错误. 第一个比较明确, 它告诉了我们如何修复! 你不能把不可变引用升级为可变引用, 所以 `iter_mut` 需要获取 `&mut self`. 复制粘贴下就好了.

```rust ,ignore
pub fn iter_mut(&mut self) -> IterMut<'_, T> {
    IterMut { next: self.head.as_mut().map(|node| &mut **node) }
}
```

那另一个是什么?

哦! 实际上上一章在写 `iter` 实现的时候犯了一个错误, 很幸运它居然能正常工作!

这是我们第一次遇到Copy的魔法. 当我们引入 [所有权][ownership] 时, 
我们说过当你移动某些东西时, 你不能使用它. 对于某些类型来说, 这很有道理. 
我们的好朋友Box为我们管理堆上的一个分配, 我们当然不希望有两段代码认为它们需要释放它的内存.

对于某些类型来说就不是这么回事了. 整数没有所有权的语义; 它们仅是毫无意义的数字! 
这也是为什么整数标记为 Copy. Copy 类型是可以按位赋值的.
所以, 当我们移动它的时候, 旧值依旧可以使用.
因此, 你可以在不替换的情况下, 从引用中 move 一个 Copy 类型!

rust 中所有的数字原语 (i32, u64, bool, f32, char, etc...) 都实现了 Copy.
你也可以为你的自定义类型实现 Copy, 只要它所有的成员都实现了 Copy.

这段代码之所以能工作, 共享引用是可以 Copy的!
因为 `&` 可以 copy, `Option<&>` 也可以 Copy. 所以当我们使用 `self.next.map` 没出现问题. 因为Option进行了 copy. 
但现在我们做不到, 因为 `&mut` 不具有 Copy (如果你复制一个&mut, 你会有两个指向同一位置的&mut, 这是不允许的). 
作为代替, 我们应该适当使用 Option 的 `take` 来获取它.


```rust ,ignore
fn next(&mut self) -> Option<Self::Item> {
    self.next.take().map(|node| {
        self.next = node.next.as_mut().map(|node| &mut **node);
        &mut node.elem
    })
}
```

```text
> cargo build

```

啊... IterMut 终于工作了!

让我们来测试一下它:


```rust ,ignore
#[test]
fn iter_mut() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter_mut();
    assert_eq!(iter.next(), Some(&mut 3));
    assert_eq!(iter.next(), Some(&mut 2));
    assert_eq!(iter.next(), Some(&mut 1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 6 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::iter_mut ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 7 passed; 0 failed; 0 ignored; 0 measured

```

好, 它工作了.


好吧我的意思是它应该能工作了, 但是还是有些问题! 这里要说清一下:

我们刚刚实现了一段代码, 它接受一个单向链表, 并最多返回一次, 对列表中每个元素的可变引用.
这是经过静态验证的. 它是完全安全的. 我们不需要做任何狂野的事.

要我说, 这可是件大事. 有一下几个原因:

* 我们 `take` 拿到 `Option<&mut>` , 所以我们对可变引用有独占访问权. 不用担心有人再看它.
* Rust 知道可以分享一个指定结构体字段可变引用, 因为它们之间不会互相影响.

您也可以使用数组或树为基本逻辑来获得安全的IterMut! 你甚至可以让迭代器双端化, 这样你就可以同时从前面 *和后面* 使用迭代器了!

[ownership]: first-ownership.md
