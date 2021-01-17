# 使用Option

敏锐的读者可能已经意识到了, 我们实际上实现了一个很烂的Option:

```rust ,ignore
enum Link {
    Empty,
    More(Box<Node>),
}
```

Link 就是 `Option<Box<Node>>`. 不用到处写 `Option<Box<Node>>` 也挺好, 并且不像 `pop`, 我们不用把它向外暴露, 所以可能还不错. 然而Option 有一些 *真的很棒* 的方法不用我们自己手动实现. 让我们把所有Link全换成Option. 首先, 我们将简单重命名每样东西使用 Some 和 None:

```rust ,ignore
use std::mem;

pub struct List {
    head: Link,
}

// 定义类型别名
type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: mem::replace(&mut self.head, None),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match mem::replace(&mut self.head, None) {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, None);
        while let Some(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, None);
        }
    }
}
```

这将稍微好一点, 但是最大的优点是来自Option的方法.

第一步, `mem::replace(&mut option, None)` 是一种非常常见的习惯用法, Option实际上把它变成了一个方法: `take`.

```rust ,ignore
pub struct List {
    head: Link,
}

type Link = Option<Box<Node>>;

struct Node {
    elem: i32,
    next: Link,
}

impl List {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: i32) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<i32> {
        match self.head.take() {
            None => None,
            Some(node) => {
                self.head = node.next;
                Some(node.elem)
            }
        }
    }
}

impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

第二步, `match option { None => None, Some(x) => Some(y) }` 常见说法为 `map`. `map` 获取一个函数, 应用于 `Some(x)` 中的 `x`  来产生 `Some(y)` 中的 `y`. 我们可以写一个合适的 `fn` , 然后将它传递给 `map`, 但是我们更愿意写一个 *内联*.

>(译者: map就是指高阶函数, map(func) -> return func(x))

我们可以通过 *闭包* 来实现. 闭包是具有额外功能的匿名函数: 它可以引用 *闭包之外* 的局部变量!
这使得它们对于处理各种条件逻辑非常有用. 我们唯一一处使用 `match` 是在 `pop` 中, 我们可以将这里重写一下:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    self.head.take().map(|node| {
        self.head = node.next;
        node.elem
    })
}
```

啊, 好多了. 让我们确认下没有弄坏什么:

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

太棒了! 让我们继续来改进代码.