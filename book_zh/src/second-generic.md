# 泛型支持

我们已经在 Option 和 Box 中接触到了一点关于泛型的支持. 
然而目前为止, 我们一直在避免声明泛型.
其实这很简单, 现在我们把所有类型都改用泛型:

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

你只需要把所有东西都加上这种 `<T>`, 你的代码就能变成泛型了. 
当然, 仅做这些实际上是不够的, 编译器会炸毛.


```text
> cargo test

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:14:6
   |
14 | impl List {
   |      ^^^^ expected 1 type argument

error[E0107]: wrong number of type arguments: expected 1, found 0
  --> src/second.rs:36:15
   |
36 | impl Drop for List {
   |               ^^^^ expected 1 type argument

```

问题十分清晰: 
我们讨论的 `List` 现在不再是指具体某一事物, 而是一个抽象的概念. 
就像 Option 和 Box, 现在我们要讨论的是 `List<Something>`.

但是我们在这些 impl 中用的 Something 是什么? 
就像 List, 我们想要我们实现的作为作用在 *所有* 的 T 中. 
所以, 和 List 一样, 让我们为 `impl` 加上尖括号:


```rust ,ignore
impl<T> List<T> {
    pub fn new() -> Self {
        List { head: None }
    }

    pub fn push(&mut self, elem: T) {
        let new_node = Box::new(Node {
            elem: elem,
            next: self.head.take(),
        });

        self.head = Some(new_node);
    }

    pub fn pop(&mut self) -> Option<T> {
        self.head.take().map(|node| {
            self.head = node.next;
            node.elem
        })
    }
}

impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

...就是这样!


```
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 2 tests
test first::test::basics ... ok
test second::test::basics ... ok

test result: ok. 2 passed; 0 failed; 0 ignored; 0 measured

```

我们的所有代码完成了泛型. Rust 非常 *简单*. 
特别提一下 `new` , 它甚至没有改变:

```rust ,ignore
pub fn new() -> Self {
    List { head: None }
}
```

我们在创建List实例时, 不会去写 `List<T>`, 而是`List<i32>`, `List<String>`这样具体的类型.

>(译者: 原文很拗口. 就换了个说法)

>原文: Bask in the Glory that is Self, guardian of refactoring and copy-pasta coding. Also of interest, we don't write `List<T>` when we construct an instance of list. That part's inferred for us based on the fact that we're returning it from a function that expects a `List<T>`.

好, 现在让我们来看看全新的*行为* !
