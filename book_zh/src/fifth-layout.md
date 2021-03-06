# Layout

那么单链队列长什么样? 很好, 当我们有一个单向链表, 从一端压入, 从另一端弹出. 堆栈和普通单链表的唯一去表就是, 从 "另一端" 弹出:

```text
build:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

push:
[Some(ptr)] -> (X, Some(ptr)) -> (A, Some(ptr)) -> (B, None)

pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

To make a queue, we just need to decide which operation to move to the
end of the list: push, or pop? Since our list is singly-linked, we can
actually move *either* operation to the end with the same amount of effort.

为了创建一个队列，我们只需要决定将哪个操作移动到列表的末尾:push，还是pop?因为我们的列表是单链的，所以我们可以用相同的努力将*任一*操作移动到最后。

To move `push` to the end, we just walk all the way to the `None` and set it
to Some with the new element.

为了将' push '移动到末尾，我们只需一路走到' None '，并使用新元素将其设置为Some。

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)

flipped push X:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)
```

To move `pop` to the end, we just walk all the way to the node *before* the
None, and `take` it:

要将' pop '移动到最后，我们只需走到* None之前的节点*，然后' take '它:

```text
input list:
[Some(ptr)] -> (A, Some(ptr)) -> (B, Some(ptr)) -> (X, None)

flipped pop:
[Some(ptr)] -> (A, Some(ptr)) -> (B, None)
```

We could do this today and call it quits, but that would stink! Both of these
operations walk over the *entire* list. Some would argue that such a queue
implementation is indeed a queue because it exposes the right interface. However
I believe that performance guarantees are part of the interface. I don't care
about precise asymptotic bounds, just "fast" vs "slow". Queues guarantee
that push and pop are fast, and walking over the whole list is definitely *not*
fast.

我们可以今天就做，然后算了，但那样太恶心了!这两种操作都遍历*整个*列表。有些人会说这样的队列实现确实是队列，因为它公开了正确的接口。然而，我相信性能保证是接口的一部分。我不关心精确的渐近界限，只关心“快”与“慢”的区别。队列保证了push和pop是快速的，而遍历整个列表肯定是“不”快速的。

One key observation is that we're wasting a ton of work doing *the same thing*
over and over. Can we memoize this work? Why, yes! We can store a pointer to
the end of the list, and just jump straight to there!

一个重要的发现是，我们在反复做“同样的事情”上浪费了大量的工作。我们能记住这项工作吗?为什么,是的!我们可以存储一个指向列表末尾的指针，然后直接跳到那里!

It turns out that only one inversion of `push` and `pop` works with this.
To invert `pop` we would have to move the "tail" pointer backwards, but
because our list is singly-linked, we can't do that efficiently.
If we instead invert `push` we only have to move the "head" pointer
forwards, which is easy.

结果是，只有一个“push”和“pop”的倒置与此工作。要反转' pop '，我们必须将“尾部”指针向后移动，但由于我们的列表是单链的，我们无法有效地做到这一点。如果我们反推“push”，我们只需要向前移动“头”指针，这很简单。

让我们试一下:

```rust ,ignore
use std::mem;

pub struct List<T> {
    head: Link<T>,
    tail: Link<T>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // swap the old tail to point to the new tail
        let old_tail = mem::replace(&mut self.tail, Some(new_tail));

        match old_tail {
            Some(mut old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
            }
        }
    }
}
```

I'm going a bit faster with the impl details now since we should be pretty
comfortable with this sort of thing. Not that you should necessarily expect
to produce this code on the first try. I'm just skipping over some of the
trial-and-error we've had to deal with before. I actually made a ton of mistakes
writing this code that I'm not showing. You can only see me leave off a `mut` or
`;` so many times before it stops being instructive. Don't worry, we'll see
plenty of *other* error messages!

现在我将更快地处理impl细节，因为我们应该对这类事情感到很舒服。并不是说您应该期望在第一次尝试时就生成这些代码。我只是略过了一些我们之前经历过的反复试验。实际上，我在写这段代码时犯了很多错误。你只能看到我把“mut”或“;”省略很多次，然后这些词就不再有意义了。不要担心，我们将看到大量的*其他*错误消息!

```text
> cargo build

error[E0382]: use of moved value: `new_tail`
  --> src/fifth.rs:38:38
   |
26 |         let new_tail = Box::new(Node {
   |             -------- move occurs because `new_tail` has type `std::boxed::Box<fifth::Node<T>>`, which does not implement the `Copy` trait
...
33 |         let old_tail = mem::replace(&mut self.tail, Some(new_tail));
   |                                                          -------- value moved here
...
38 |                 old_tail.next = Some(new_tail);
   |                                      ^^^^^^^^ value used here after move
```

Shoot!

> use of moved value: `new_tail`

Box doesn't implement Copy, so we can't just assign it to two locations. More
importantly, Box *owns* the thing it points to, and will try to free it when
it's dropped. If our `push` implementation compiled, we'd double-free the tail
of our list! Actually, as written, our code would free the old_tail on every
push. Yikes! 🙀

Box没有实现复制，所以我们不能把它分配给两个位置。更重要的是，Box *拥有*它指向的东西，并将尝试在它被丢弃时释放它。如果我们的“push”实现编译完成，我们将双倍释放列表的尾部!实际上，正如所写的，我们的代码将在每次push时释放old_tail。呵!🙀

Alright, well we know how to make a non-owning pointer. That's just a reference!

我们知道如何创建一个非所有者指针。这只是一个引用!

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: Option<&mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_mut().map(|node| &mut **node)
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_mut().map(|node| &mut **node)
            }
        };

        self.tail = new_tail;
    }
}
```

Nothing too tricky here. Same basic idea as the previous code, except we're
using some of that implicit return goodness to extract the tail reference from
wherever we stuff the actual Box.


这没什么难的。基本思想与前面的代码相同，只是我们使用了一些隐式的return good来从填充实际框的任何地方提取尾部引用。

```text
> cargo build

error[E0106]: missing lifetime specifier
 --> src/fifth.rs:3:18
  |
3 |     tail: Option<&mut Node<T>>, // NEW!
  |                  ^ expected lifetime parameter
```

Oh right, we need to give references in types lifetimes. Hmm... what's the
lifetime of this reference? Well, this seems like IterMut, right? Let's try
what we did for IterMut, and just add a generic `'a`:

对了，我们需要引用生命周期中的类型。嗯…这个引用的生命周期是多少?这看起来像IterMut，对吧?让我们尝试一下我们对IterMut所做的，只是添加一个通用的“a”:

```rust ,ignore
pub struct List<'a, T> {
    head: Link<T>,
    tail: Option<&'a mut Node<T>>, // NEW!
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}

impl<'a, T> List<'a, T> {
    pub fn new() -> Self {
        List { head: None, tail: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_tail = Box::new(Node {
            elem: elem,
            // When you push onto the tail, your next is always None
            next: None,
        });

        // Put the box in the right place, and then grab a reference to its Node
        let new_tail = match self.tail.take() {
            Some(old_tail) => {
                // If the old tail existed, update it to point to the new tail
                old_tail.next = Some(new_tail);
                old_tail.next.as_mut().map(|node| &mut **node)
            }
            None => {
                // Otherwise, update the head to point to it
                self.head = Some(new_tail);
                self.head.as_mut().map(|node| &mut **node)
            }
        };

        self.tail = new_tail;
    }
}
```

```text
cargo build

error[E0495]: cannot infer an appropriate lifetime for autoref due to conflicting requirements
  --> src/fifth.rs:35:27
   |
35 |                 self.head.as_mut().map(|node| &mut **node)
   |                           ^^^^^^
   |
note: first, the lifetime cannot outlive the anonymous lifetime #1 defined on the method body at 18:5...
  --> src/fifth.rs:18:5
   |
18 | /     pub fn push(&mut self, elem: T) {
19 | |         let new_tail = Box::new(Node {
20 | |             elem: elem,
21 | |             // When you push onto the tail, your next is always None
...  |
39 | |         self.tail = new_tail;
40 | |     }
   | |_____^
note: ...so that reference does not outlive borrowed content
  --> src/fifth.rs:35:17
   |
35 |                 self.head.as_mut().map(|node| &mut **node)
   |                 ^^^^^^^^^
note: but, the lifetime must be valid for the lifetime 'a as defined on the impl at 13:6...
  --> src/fifth.rs:13:6
   |
13 | impl<'a, T> List<'a, T> {
   |      ^^
   = note: ...so that the expression is assignable:
           expected std::option::Option<&'a mut fifth::Node<T>>
              found std::option::Option<&mut fifth::Node<T>>


```

Woah, that's a really detailed error message. That's a bit concerning, because it
suggests we're doing something really messed up. Here's an interesting part:

哇，这是一条非常详细的错误信息。这有点令人担忧，因为这表明我们正在做一些非常糟糕的事情。这里有一个有趣的部分:

> the lifetime must be valid for the lifetime `'a` as defined on the impl

> 生命周期必须在impl上定义的生命周期“a”内有效


We're borrowing from `self`, but the compiler wants us to last as long as `'a`,
what if we tell it `self` *does* last that long..?

我们借用了' self '，但是编译器想让我们持续的时间和'a '一样长，如果我们告诉它' self '确实持续那么长…

```rust ,ignore
    pub fn push(&'a mut self, elem: T) {
```
'a
```text
cargo build

warning: field is never used: `elem`
 --> src/fifth.rs:9:5
  |
9 |     elem: T,
  |     ^^^^^^^
  |
  = note: #[warn(dead_code)] on by default
```

Oh, hey, that worked! Great!

Let's just do `pop` too:

我们也来看看pop:

```rust ,ignore
pub fn pop(&'a mut self) -> Option<T> {
    // Grab the list's current head
    self.head.take().map(|head| {
        let head = *head;
        self.head = head.next;

        // If we're out of `head`, make sure to set the tail to `None`.
        if self.head.is_none() {
            self.tail = None;
        }

        head.elem
    })
}
```

And write a quick test for that:

```rust ,ignore
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
    }
}
```

```text
cargo test

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:68:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
68 |         list.push(1);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:69:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
69 |         list.push(2);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here

error[E0499]: cannot borrow `list` as mutable more than once at a time
  --> src/fifth.rs:70:9
   |
65 |         assert_eq!(list.pop(), None);
   |                    ---- first mutable borrow occurs here
...
70 |         list.push(3);
   |         ^^^^
   |         |
   |         second mutable borrow occurs here
   |         first borrow later used here


....

** WAY MORE LINES OF ERRORS **

....

error: aborting due to 11 previous errors
```

🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀🙀

Oh my goodness.

The compiler's not wrong for vomiting all over us. We just committed a
cardinal Rust sin: we stored a reference to ourselves *inside ourselves*.
Somehow, we managed to convince Rust that this totally made sense in our
`push` and `pop` implementations (I was legitimately shocked we did). I believe
the reason is that Rust can't yet tell that the reference is into ourselves
from just `push` and `pop` -- or rather, Rust doesn't really have that notion
at all. Reference-into-yourself failing to work is just an emergent behaviour.

编译器吐了我们一身并没有错。我们刚刚犯了一个严重的错误:我们在我们自己的内部存储了一个对我们自己的引用。我们设法说服Rust相信这在我们的“推送”和“pop”实现中是有意义的。我相信原因是Rust还不能从“push”和“pop”这两个词中分辨出这个词对我们的影响——或者说，Rust根本没有这个概念。不工作只是一种紧急的行为。

As soon as we tried to *use* our list, everything quickly fell apart.
When we call `push` or `pop`, we promptly store a reference to ourselves in
ourselves and become *trapped*. We are literally borrowing ourselves.

当我们试图“使用”我们的列表时，一切很快就崩溃了。当我们调用“push”或“pop”时，我们立即在“ourselves”中存储了一个对我们自己的引用，并陷入“陷阱”。我们实际上是在借用自己。


Our `pop` implementation hints at why this could be really dangerous:

我们的“pop”实现暗示了为什么这是非常危险的:

```rust ,ignore
// ...
if self.head.is_none() {
    self.tail = None;
}
```

What if we forgot to do this? Then our tail would point to some node *that
had been removed from the list*. Such a node would be instantly freed, and we'd
have a dangling pointer which Rust was supposed to protect us from!

And indeed Rust is protecting us from that kind of danger. Just in a very...
**roundabout** way.

So what can we do? Go back to `Rc<RefCell>>` hell?

Please. No.

No instead we're going to go off the rails and use *raw pointers*.
Our layout is going to look like this:

要是我们忘了呢?然后，我们的尾部将指向从列表中删除的某个节点*。这样的节点将立即被释放，我们将有一个悬浮的指针，Rust应该保护我们不受它的影响!
事实上，锈蚀正在保护我们免受这种危险。只是在一个非常…* *的* *。
那么我们能做什么呢?回到地狱?
请。不。
不，相反，我们将偏离轨道，使用原始指针*。我们的布局是这样的:

```rust ,ignore
pub struct List<T> {
    head: Link<T>,
    tail: *mut Node<T>, // DANGER DANGER
}

type Link<T> = Option<Box<Node<T>>>;

struct Node<T> {
    elem: T,
    next: Link<T>,
}
```

And that's that. None of this wimpy reference-counted-dynamic-borrow-checking
nonsense! Real. Hard. Unchecked. Pointers.

那就是了。这些没用的引用计数动态借阅检查!真实的。困难的。无节制的。指针。

Let's be C everyone. Let's be C all day.

I'm home. I'm ready.

Hello `unsafe`.



