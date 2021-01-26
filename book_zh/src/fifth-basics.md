# Basics

Alright, back to basics. How do we construct our list?

好了，回到最基本的。我们如何构造我们的列表?

Before we just did:

之前我们做过

```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }
}
```

But we're not using Option for the `tail` anymore:

但我们不再为“尾巴”使用Option:

```text
> cargo build

error[E0308]: mismatched types
  --> src/fifth.rs:15:34
   |
15 |         List { head: None, tail: None }
   |                                  ^^^^ expected *-ptr, found enum `std::option::Option`
   |
   = note: expected type `*mut fifth::Node<T>`
              found type `std::option::Option<_>`
```

We *could* use an Option, but unlike Box, `*mut` *is* nullable. This means it
can't benefit from the null pointer optimization. Instead, we'll be using `null`
to represent None.

我们*可以*使用一个选项，但不像Box， ' *mut ' *是*空的。这意味着它不能从空指针优化中受益。相反，我们将使用' null '来表示None。

So how do we get a null pointer? There's a few ways, but I prefer to use
`std::ptr::null_mut()`. If you want, you can also use `0 as *mut _`, but that
just seems so *messy*.

那么我们如何得到一个空指针呢?有几种方法，但我更喜欢使用' std::ptr::null_mut() '。如果你愿意，你也可以使用' 0作为*mut _ '，但这看起来是如此的*混乱*。

```rust ,ignore
use std::ptr;

// defns...

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: ptr::null_mut() }
    }
}
```

```text
cargo build

warning: field is never used: `head`
 --> src/fifth.rs:4:5
  |
4 |     head: Link<T>,
  |     ^^^^^^^^^^^^^
  |
  = note: #[warn(dead_code)] on by default

warning: field is never used: `tail`
 --> src/fifth.rs:5:5
  |
5 |     tail: *mut Node<T>,
  |     ^^^^^^^^^^^^^^^^^^

warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^

warning: field is never used: `head`
  --> src/fifth.rs:12:5
   |
12 |     head: Link<T>,
   |     ^^^^^^^^^^^^^
```

*shush* compiler, we will use them soon.

*嘘*编译器，我们很快就会使用它们。

Alright, let's move on to writing `push` again. This time, instead of grabbing
an `Option<&mut Node<T>>` after we insert, we're just going to grab a
`*mut Node<T>` to the insides of the Box right away. We know we can soundly do
this because the contents of a Box has a stable address, even if we move the
Box around. Of course, this isn't *safe*, because if we just drop the Box we'll
have a pointer to freed memory.

好了，我们继续写push。这一次，我们不再在插入之后抓取“Option&lt;&mut no&lt;T&gt;&gt;”，而是直接在盒子内部抓取一个“*mut no&lt;&gt;”。我们知道我们可以这样做，因为盒子的内容有一个稳定的地址，即使我们移动盒子。当然，这并不是“安全”的，因为如果我们只是丢弃盒子，我们就会有一个指向已释放内存的指针。

How do we make a raw pointer from a normal pointer? Coercions! If a variable
is declared to be a raw pointer, a normal reference will coerce into it:

我们如何使一个原始指针从一个普通的指针?强迫!如果变量声明为原始指针，则普通引用将强制转换为该变量:

```rust ,ignore
let raw_tail: *mut _ = &mut *new_tail;
```

We have all the info we need. We can translate our code into, approximately,
the previous reference version:

我们需要的信息都有了。我们可以把我们的代码近似地翻译成前面的参考版本:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // .is_null checks for null, equivalent to checking for None
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        self.tail.next = Some(new_tail);
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             self.tail.next = Some(new_tail);
   |             ----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

Huh? We have a pointer to a Node, why can't we get the `next` field?

嗯?我们有一个指向节点的指针，为什么我们不能得到' next '字段?

Rust is kinda a jerk when you use raw pointers. To access the contents of a
raw pointer, it insists that we manually deref them, because it's such an unsafe
operation. So let's do that:


当你用原始指针的时候，拉斯特有点混蛋。为了访问原始指针的内容，它坚持要手动对其进行解归，因为这是一个非常不安全的操作。让我们这样做:

```rust ,ignore
*self.tail.next = Some(new_tail);
```

```text
> cargo build

error[E0609]: no field `next` on type `*mut fifth::Node<T>`
  --> src/fifth.rs:31:23
   |
31 |             *self.tail.next = Some(new_tail);
   |             -----------^^^^
   |             |
   |             help: `self.tail` is a raw pointer; try dereferencing it: `(*self.tail).next`
```

Uuuugh 运算符优先级.

```rust ,ignore
(*self.tail).next = Some(new_tail);
```

```text
> cargo build

error[E0133]: dereference of raw pointer is unsafe and requires unsafe function or block
  --> src/fifth.rs:31:13
   |
31 |             (*self.tail).next = Some(new_tail);
   |             ^^^^^^^^^^^^^^^^^ dereference of raw pointer
   |
   = note: raw pointers may be NULL, dangling or unaligned; they can violate aliasing rules and cause data races: all of these are undefined behavior
```

这应该不会这么难.

Remember how I said Unsafe Rust is like an FFI language for Safe Rust? Well, the
compiler wants us to explicitly delimit where we're doing this FFI-ing. We have
two options. First, we can mark our *entire* function as unsafe, in which case
it becomes an Unsafe Rust function and can only be called in an `unsafe`
context. This isn't great, because we want our list to be safe to use. Second,
we can write an `unsafe` block inside our function, to delimit the FFI boundary.
This declares the overall function to be safe. Let's do that one:

还记得我说不安全的锈蚀就像一个FFI语言的安全锈蚀吗?编译器想让我们显式地界定我们在哪里做这个ffi。我们有两个选择。首先，我们可以将我们的*整个*函数标记为不安全的，在这种情况下，它成为一个不安全的Rust函数，只能在“不安全”的环境中调用。这并不好，因为我们希望我们的列表是安全使用的。其次，我们可以在函数内部编写一个“不安全”块，来划定FFI边界。这声明了整个函数是安全的。让我们来做这个:

```rust ,ignore
pub fn push(&mut self, elem: T) {
    let mut new_tail = Box::new(Node {
        elem: elem,
        next: None,
    });

    let raw_tail: *mut _ = &mut *new_tail;

    // Put the box in the right place, and then grab a reference to its Node
    if !self.tail.is_null() {
        // If the old tail existed, update it to point to the new tail
        unsafe {
            (*self.tail).next = Some(new_tail);
        }
    } else {
        // Otherwise, update the head to point to it
        self.head = Some(new_tail);
    }

    self.tail = raw_tail;
}
```

```text
> cargo build
warning: field is never used: `elem`
  --> src/fifth.rs:11:5
   |
11 |     elem: T,
   |     ^^^^^^^
   |
   = note: #[warn(dead_code)] on by default
```

Yay!

It's kind've interesting that that's the *only* place we've had to write an
unsafe block so far. We do raw pointer stuff all over the place, what's up with
that?

有趣的是，到目前为止，这是我们唯一写不安全块的地方。我们到处都是原始指针，这是怎么回事?

It turns out that Rust is a massive rules-lawyer pedant when it comes to
`unsafe`. We quite reasonably want to maximize the set of Safe Rust programs,
because those are programs we can be much more confident in. To accomplish this,
Rust carefully carves out a minimal surface area for unsafety. Note that all
the other places we've worked with raw pointers has been *assigning* them, or
just observing whether they're null or not.

事实证明，Rust是一个庞大的规则——当涉及到“不安全”时，律师学究。我们很有理由最大化安全的Rust程序集，因为我们对这些程序更有信心。为了做到这一点，铁锈小心翼翼地把不安全的表面面积缩小到最小。注意，我们处理原始指针的所有其他地方都一直在*赋值*它们，或者只是观察它们是否为空。

If you never actually dereference a raw pointer *those are totally safe things
to do*. You're just reading and writing an integer! The only time you can
actually get into trouble with a raw pointer is if you actually dereference it.
So Rust says *only* that operation is unsafe, and everything else is totally
safe.

如果你从来没有真正地解引用一个原始指针*那是完全安全的事情。您只是读写一个整数!使用原始指针唯一可能遇到麻烦的情况是对它进行解引用。所以Rust只说操作是不安全的，其他一切都是完全安全的。

Super. Pedantic. But technically correct.

超级。迂腐的。但技术上正确的。

However this raises an interesting problem: although we're supposed to delimit
the scope of the unsafety with the `unsafe` block, it actually depends on
state that was established outside of the block. Outside of the function, even!

然而，这引发了一个有趣的问题:尽管我们应该用“不安全”块来划分不安全块的范围，但它实际上取决于在块之外建立的状态。在函数之外，甚至!

This is what I call unsafe *taint*. As soon as you use `unsafe` in a module,
that whole module is tainted with unsafety. Everything has to be correctly
written in order to make sure that invariants are upheld for the unsafe code.

这就是我所说的不安全的*污染*。一旦你在一个模块中使用了“unsafe”，整个模块就会被“unsafe”污染。为了确保不安全的代码支持不变量，必须正确地编写所有内容。

This taint is manageable because of *privacy*. Outside of our module, all of our
struct fields are totally private, so no one else can mess with our state in
arbitrary ways. As long as no combination of the APIs we expose causes bad stuff
to happen, as far as an outside observer is concerned, all of our code is safe!
And really, this is no different from the FFI case. No one needs to care
if some python math library shells out to C as long as it exposes a safe
interface.

这种污染是可控的，因为*隐私*。在我们的模块之外，所有的struct字段都是完全私有的，所以其他人不能以任意的方式干扰我们的状态。只要我们公开的api组合没有导致糟糕的事情发生，只要外部观察者关心，我们所有的代码都是安全的!实际上，这和FFI的情况没有什么不同。只要有一个安全的接口，没有人需要关心python数学库是否会外壳到C语言。

Anyway, let's move on to `pop`, which is pretty much verbatim the reference
version:

不管怎样，让我们来看看“pop”，它几乎和参考版本一模一样:

```rust ,ignore
pub fn pop(&mut self) -> Option<T> {
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        if self.head.is_none() {
            self.tail = ptr::null_mut();
        }

        head.elem
    })
}
```

Again we see another case where safety is stateful. If we fail to null out the
tail pointer in *this* function, we'll see no problems at all. However
subsequent calls to `push` will start writing to the dangling tail!

我们再次看到了安全是有状态的另一种情况。如果在*this*函数中未能将尾部指针置空，我们就不会看到任何问题。然而，对“push”的后续调用将开始写入悬垂尾巴!

Let's test it out:

```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    #[test]
    fn basics() {
        let mut list = List::new();

        // Check empty list behaves right
        assert_eq!(list.pop(), None);

        // Populate list
        list.push(1);
        list.push(2);
        list.push(3);

        // Check normal removal
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), None);

        // Check the exhaustion case fixed the pointer right
        list.push(6);
        list.push(7);

        // Check normal removal
        assert_eq!(list.pop(), Some(6));
        assert_eq!(list.pop(), Some(7));
        assert_eq!(list.pop(), None);
    }
}
```

This is just the stack test, but with the expected `pop` results flipped around.
I also added some extra steps at the end to make sure that tail-pointer
corruption case in `pop` doesn't occur.

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 12 tests
test fifth::test::basics ... ok
test first::test::basics ... ok
test fourth::test::basics ... ok
test fourth::test::peek ... ok
test second::test::basics ... ok
test fourth::test::into_iter ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured
```

Gold Star!


