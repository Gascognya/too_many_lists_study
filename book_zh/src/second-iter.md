# Iter

Alright, let's try to implement Iter. This time we won't be able to rely on
List giving us all the features we want. We'll need to roll our own. The
basic logic we want is to hold a pointer to the current node we want to yield
next. Because that node may not exist (the list is empty or we're otherwise
done iterating), we want that reference to be an Option. When we yield an
element, we want to proceed to the current node's `next` node.

Alright, let's try that:

好吧, 我们来尝试实现下 Iter. 这次我们不能指望 List 为我们提供便利. 我们要自己动手. 我们想要的基本逻辑是用一个指针指向 我们想要输出其 next 的 node. 因为 node 可能不存在 (list 为空 或者迭代已完成), 我们需要引用一个 Option. 当我们生成一个元素, 我们需要到当前 node 的`next` node.

好, 我们来试试:


```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:72:18
   |
72 |     next: Option<&Node<T>>,
   |                  ^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:82:17
   |
82 |     type Item = &T;
   |                 ^ expected lifetime parameter
```

完了. 生命周期. 我听过这玩意. 大家都说它是一个梦魇.

让我们尝试些新东西: 看到这个 `error[E0106]` 了么? 这是编译器错误码. 我们可以看看rustc的解释, 好, `--explain`:

```text
> rustc --explain E0106
This error indicates that a lifetime is missing from a type. If it is an error
inside a function signature, the problem may be with failing to adhere to the
lifetime elision rules (see below).

Here are some simple examples of where you'll run into this error:

struct Foo { x: &bool }        // error
struct Foo<'a> { x: &'a bool } // correct

enum Bar { A(u8), B(&bool), }        // error
enum Bar<'a> { A(u8), B(&'a bool), } // correct

type MyStr = &str;        // error
type MyStr<'a> = &'a str; //correct
...

```

嗯... 这并没有讲清很多问题 (这文档假设我们已经了解rust了). 但看起来我们应该加个 `'a` 在我们的结构体中? 试一下吧.

```
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}
```

```text
> cargo build

error[E0106]: missing lifetime specifier
  --> src/second.rs:83:22
   |
83 | impl<T> Iterator for Iter<T> {
   |                      ^^^^^^^ expected lifetime parameter

error[E0106]: missing lifetime specifier
  --> src/second.rs:84:17
   |
84 |     type Item = &T;
   |                 ^ expected lifetime parameter

error: aborting due to 2 previous errors
```

好吧, 让我们给所有东西都加上试试:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<'a, T> List<T> {
    pub fn iter(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &'a node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&'a mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &'a node);
            &'a node.elem
        })
    }
}
```

```text
> cargo build

error: expected `:`, found `node`
  --> src/second.rs:77:47
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ---- while parsing this struct        ^^^^ expected `:`

error: expected `:`, found `node`
  --> src/second.rs:85:50
   |
85 |             self.next = node.next.map(|node| &'a node);
   |                                                  ^^^^ expected `:`

error[E0063]: missing field `next` in initializer of `second::Iter<'_, _>`
  --> src/second.rs:77:9
   |
77 |         Iter { next: self.head.map(|node| &'a node) }
   |         ^^^^ missing `next`
```

完了, 搞砸了.

可能我们要彻底理解 生命周期 `'a` 这玩意究竟是什么意思.

生命周期吓跑了很多人, 因为它改变了我们自编程之初就知道和喜欢的东西. 截至目前为止, 我们成功避开了生命周期, 虽然我们始终没去直面它.

在有 GC 的语言中, 生命周期没有意义, 因为 GC 确保所有东西都可以正确的产生与销毁. Rust 中的大多数数据是 *手动* 管理的, 所以需要其他的解决方案. C 和
C++ 向我们清楚地展示了, 用户拿指针随意指向堆栈上的数据会发生什么: 无处不在的危险. 
错误大体可以分为两类:

* 指向超出作用域之外的东西
* 指向的东西被改变

生命周期在99%的时间解决了这个问题, 用一个易懂的方式.

所以什么是生命周期?

很简单, 一个生命周期是程序中一段(块/作用域)代码的名称 .
就是这样. 当引用被标记上生命周期时, 我们说它必须在 *整个* 区域中都有效. 
不同的地方对引用可以或必须有效有不同的需求. 
整个生命周期系统反过来就是一个约束求解系统, 它试图将每个引用的区域最小化. 
如果它成功地找到了一组满足所有约束的生存期, 程序将会顺利编译! 
否则就会返回一个错误，说某样东西存活的时间不够长.

通常在函数中我们是不讨论生命周期的, 而且也不想这么做. 编译器有足够多的信息来供他推断出所有的约束, 进而推断出最小生命周期. 然而在 类型 和 API级 上, 编译器 *不能* 拥有所有信息. 
它要求你告诉它不同生命周期之间的关系, 这样它才能知道你要做什么.

原则上讲, 这些生命周期其实也可以不考虑, 但检查所有借用将是一个庞大的整个程序分析，将产生令人难以置信的非局部错误. Rust的系统应当所有的借阅检查都可以在每个函数体中独立完成，所有的错误都应该是局部的 (或者你的类型定义有错误的签名).

But we've written references in function signatures before, and it was fine!
这是因为有些情况非常常见，Rust会自动为你挑选生命周期. 这叫做 *生命周期省略*.

特殊情况:

```rust ,ignore
// 输入只有一个引用, 所以输出一定衍生自输入
fn foo(&A) -> &B; // sugar for:
fn foo<'a>(&'a A) -> &'a B;

// 很多输入, 假设它们都是各自独立的
fn foo(&A, &B, &C); // sugar for:
fn foo<'a, 'b, 'c>(&'a A, &'b B, &'c C);

// 方法, 假设所有输出的生命周期都衍生自 `self`
fn foo(&self, &B, &C) -> &D; // sugar for:
fn foo<'a, 'b, 'c>(&'a self, &'b B, &'c C) -> &'a D;
```

所以 `fn foo<'a>(&'a A) -> &'a B` 是什么意思? 在实践中, 它意味着输入项(A)一定活得比输出项(B)久. 所以如果你想保持输出(B)长时间存在, 这将扩大输入(A)的必须存在时间. 一旦你不再需要使用输出(B), 编译器会知道这时候可以允许输入(A)下班了.

通过这个系统, Rust 可以确保没有任何东西会在释放后被调用(因为直到没有需要这个东西的时候, 才会允许它被释放. 它总是比所有需要它的东西活得久), 并且当外界引用存在时, 没有任何东西会被改变. 它确保所有约束条件都成立!

好, 现在回到Iter.

让我们回滚到没有生命周期的状态:

```rust ,ignore
pub struct Iter<T> {
    next: Option<&Node<T>>,
}

impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

impl<T> Iterator for Iter<T> {
    type Item = &T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

我们只需要在函数和类型定义上, 加入生命周期:

```rust ,ignore
// Iter is generic over *some* lifetime, it doesn't care
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

// 这不写生命周期, List 不关联任何生命周期
impl<T> List<T> {
    // 我们为产生 Iter 的借用创建一个生命周期.
    // 现在 &self 需要活得比 Iter 更久.
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &node) }
    }
}

// 我们需要在这写生命周期, 因为 Iter 有一个我们需要定义的
impl<'a, T> Iterator for Iter<'a, T> {
    // 这也需要, 这是一个类型声明, 代表着返回类型, 而返回类型中是需要标明的
    type Item = &'a T;

    // 这不需要更改, 由上面处理.
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &node);
            &node.elem
        })
    }
}
```

好, 再来试一下.

```text
cargo build

error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`
```

(╯°□°)╯︵ ┻━┻

好吧. 我们修复了生命周期错误, 但是却得到了新种类的错误.

我们想要储存 `&Node`'s, 但是获得了 `&Box<Node>`s. 好吧, 这个简单, 我们仅需要在引用之间, 对 Box 解引用:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;
    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:77:43
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                                           ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.map(|node| &*node) }
   |                      ^^^^^^^^^ cannot move out of borrowed content

error[E0515]: cannot return reference to local data `*node`
  --> src/second.rs:85:46
   |
85 |             self.next = node.next.map(|node| &*node);
   |                                              ^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.map(|node| &*node);
   |                         ^^^^^^^^^ cannot move out of borrowed content
```

(ﾉಥ益ಥ）ﾉ ┻━┻

我们忘记了 `as_ref`, 所以我们把 Box move 进了 `map`, 这意味着它将被销毁, 我们的指针将会悬空:

```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &*node) }
    }
}

impl<'a, T> Iterator for Iter<'a, T> {
    type Item = &'a T;

    fn next(&mut self) -> Option<Self::Item> {
        self.next.map(|node| {
            self.next = node.next.as_ref().map(|node| &*node);
            &node.elem
        })
    }
}
```

```text
cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0308]: mismatched types
  --> src/second.rs:77:22
   |
77 |         Iter { next: self.head.as_ref().map(|node| &*node) }
   |                      ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

error[E0308]: mismatched types
  --> src/second.rs:85:25
   |
85 |             self.next = node.next.as_ref().map(|node| &*node);
   |                         ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ expected struct `second::Node`, found struct `std::boxed::Box`
   |
   = note: expected type `std::option::Option<&'a second::Node<T>>`
              found type `std::option::Option<&std::boxed::Box<second::Node<T>>>`

```

😭

`as_ref` 添加了一层引用, 我们需要移除掉:


```rust ,ignore
pub struct Iter<'a, T> {
    next: Option<&'a Node<T>>,
}

impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
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

```text
cargo build

```

🎉 🎉 🎉

你可能在想 "`&**` 是真的麻烦", 不必担心.
Rust 通常非常擅长通过 *自动解引用* 这种隐式转换, 它可以在你的代码中插入 \* 来进行类型检查. 它能做到这一点是因为我们有借出检查器来确保我们不会弄乱指针!

但是在这种情况下, 闭包加上我们用 `Option<&T>` 替换 `&T` 对于它来说实在是太复杂了, 所以我们需要做点什么. 幸运的是, 在我的经验中, 这种情况很少见.

为了完整起见, 我们可以给它一个不一样的提示, 通过 *Turbofish符号*:

```rust ,ignore
self.next = node.next.as_ref().map::<&Node<T>, _>(|node| &node);
```

看, map其实是一个泛型函数.

```rust ,ignore
pub fn map<U, F>(self, f: F) -> Option<U>
```

turbofish 符号, `::<>`, 让我们告诉编译器我们认为这些泛型的类型应该是什么. 
在当前情况下 `::<&Node<T>, _>` 表明 "它应该返回一个`&Node<T>`, 且我并不了解/关心其他类型".

进而让编译器直到应该对 `&node` 进行强制解引用, 所以我们不需要手动加所有 \*'s!

但在这种情况下, 我不认为这是真正的改进, 这只是一个掩饰不住的借口, 以炫耀强制解引用和有时有用的turbofish. 😅

让我们编写一个测试，以确保我们没有对它执行空操作或其他操作:

```rust ,ignore
#[test]
fn iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.iter();
    assert_eq!(iter.next(), Some(&3));
    assert_eq!(iter.next(), Some(&2));
    assert_eq!(iter.next(), Some(&1));
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 5 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

欧耶.

最后, 实际上我们可以省略这的生命周期:

```rust ,ignore
impl<T> List<T> {
    pub fn iter<'a>(&'a self) -> Iter<'a, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```

它相当于:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```

好, 生命周期更少了!

或者, 如果你不喜欢隐藏生命周期,
你可以试试 Rust 2018 "显示省略生命周期" 语法,  `'_`:

```rust ,ignore
impl<T> List<T> {
    pub fn iter(&self) -> Iter<'_, T> {
        Iter { next: self.head.as_ref().map(|node| &**node) }
    }
}
```
