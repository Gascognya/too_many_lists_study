# Pop

与 `push` 一样, `pop` 也想要对链表进行改变. 与 `push` 不同的是,  `pop` 需要返回一些东西. 但 `pop` 需要处理一些棘手的情况: 如果List为空该怎办? 为了描述这个情况, 我们使用可靠的 `Option` 类型:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    // TODO
}
```

`Option<T>` 是一个描述值 `可能存在` 的枚举类型. 他可以是
`Some(T)` 或 `None`. 我们可以构造一个枚举, 就像Link那样, 但是我们希望我们的用户能够理解我们return的东西是什么, Option 无处不在且 *人尽皆知* . 事实上, 它真的非常基础, 隐藏在每个rust文件的作用域中, 所以我们可以直接使用 `Some` 和 `None` (一般我们不用 `Option::None` 这种复杂的形式).

`Option<T>` 的尖括号部分表示这是一个 *泛型*. 这意味着用户可以在Option中配置 *加入* 类型!

所以, 我们该怎样推断出 `Link` 表示 Empty 还是 More? 不如让我们来使用 `match` 进行匹配吧!

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
}
```

```text
> cargo build

error[E0308]: mismatched types
  --> src/first.rs:27:30
   |
27 |     pub fn pop(&mut self) -> Option<i32> {
   |            ---               ^^^^^^^^^^^ expected enum `std::option::Option`, found ()
   |            |
   |            this function's body doesn't return
   |
   = note: expected type `std::option::Option<i32>`
              found type `()`
```

哦呦, `pop` 需要返回一个值, 但我们现在还什么都没做. 我们 *可以*
return `None`, 但这种更好的办法是 return `unimplemented!()`,来表明我们还没有实现这个函数. `unimplemented!()` 是一个宏 (`!` 表示一个宏) 让程序抛出 panic 
当我们用到它时 (\~主动控制其崩溃).

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

无条件崩溃是一个 [发散函数][diverging] 的例子.
发散函数永远不会return值给调用者, (译者：它可以代表任何类型的值, 因为它永远不会返回任何值). 此处, `unimplemented!()` 它被用在表示 `Option<T>` 类型的地方.
还要注意, 此处我们不需要写 `return`. 最后一个表达式在函数中, 隐式的表达返回值.这样我们表达一些简单的东西时, 会非常简洁. 你也可以像C-like语言那样, 显式的把 `return` 写出来.

```text
> cargo build

error[E0507]: cannot move out of borrowed content
  --> src/first.rs:28:15
   |
28 |         match self.head {
   |               ^^^^^^^^^
   |               |
   |               cannot move out of borrowed content
   |               help: consider borrowing here: `&self.head`
...
32 |             Link::More(node) => {
   |                        ---- data moved here
   |
note: move occurs because `node` has type `std::boxed::Box<first::Node>`, which does not implement the `Copy` trait
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^
```

Rust你快起点作用啊! 像往常一样, Rust对我们很生气. 十分感谢,
这次它也给了我们足够的信息! 在默认情况下, 模式匹配将尝试将其内容move到新的分支中, 但是我们不能这样做, 因为这是我们借来的self, 并不是 `value` 而是 `mutable reference`.

```text
help: consider borrowing here: `&self.head`
```

Rust 说我们应该添加一个引用在 `match` 来修复它. 🤷‍♀️ 让我们来试试:

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match &self.head {
        Link::Empty => {
            // TODO
        }
        Link::More(ref node) => {
            // TODO
        }
    };
    unimplemented!()
}
```

```text
> cargo build

warning: unused variable: `node`
  --> src/first.rs:32:24
   |
32 |             Link::More(node) => {
   |                        ^^^^ help: consider prefixing with an underscore: `_node`
   |
   = note: #[warn(unused_variables)] on by default

warning: field is never used: `elem`
  --> src/first.rs:13:5
   |
13 |     elem: i32,
   |     ^^^^^^^^^
   |
   = note: #[warn(dead_code)] on by default

warning: field is never used: `next`
  --> src/first.rs:14:5
   |
14 |     next: Link,
   |     ^^^^^^^^^^
```

好耶, 再编译一遍! 现在让我们来体验下逻辑. 我们想要做一个
Option, 所以我们来定义一个变量. 在 Empty 的情况下, 我们需要 return
None. 在其他情况下我们需要 return `Some(i32)`, 然后改变列表的head, 所以让我们来尝试下?

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match &self.head {
        Link::Empty => {
            result = None;
        }
        Link::More(ref node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
> cargo build
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
error[E0507]: cannot move out of borrowed content
  --> src/first.rs:35:29
   |
35 |                 self.head = node.next;
   |                             ^^^^^^^^^ cannot move out of borrowed content

```

*head*

*desk*

我们尝试move `node` 在我们只有共享引用(不可变引用)的情况下.

我们可以需要退一步, 重新想想我们需要做什么. 我们希望:

* 检测list是否为空.
* 如果为空, 就 return None
* 如果 *不* 为空
    * remove 列表的 head
    * remove 其中的 `elem`
    * replace 列表的head为其 `next`
    * return `Some(elem)`

问题的关键是我们想要 *remove* 一些东西, 这意味着我们需要获得list的head *为 value*. 我们当然不能对通过 `&self.head` 获得的共享引用这样做. 我们也 "只" 有一个 `self` 的可变引用, 所以移动其中填充物的唯一方式是 *replace* 它. 看来我们又要偷天换日了!

让我们来试一试:


```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    let result;
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => {
            result = None;
        }
        Link::More(node) => {
            result = Some(node.elem);
            self.head = node.next;
        }
    };
    result
}
```

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

卧槽

居然没报错!!!!!

我们在这要返回 这个 `result`, 但事实上我们不需要写return! 就像函数的返回值是最后一个表达式一样, 每一个代码块的返回结果, 都是其最后一个表达式. 通常我们可以使用分号制止这种行为,
这会使代码块的返回值为一个空元组, `()`. 这是未声明返回值的函数, 实际上的返回值, 例如 
`push` 的返回值.

>(译者: 表达式不包含分号, 包含分号的被称作语句. 当代码块或函数并没有声明返回值时(没有return语句, 最后一行也不是表达式), 这个函数实际上return的结果是一个单元类型`()`, 也就是一个空元组.)

所以, 我们可以这样写 `pop` :

```rust ,ignore
pub fn pop(&mut self) -> Option<i32> {
    match mem::replace(&mut self.head, Link::Empty) {
        Link::Empty => None,
        Link::More(node) => {
            self.head = node.next;
            Some(node.elem)
        }
    }
}

```

>(译者: 注意match结尾的分号没了, 说明整个match语句是个表达式, 其值作为整个函数的返回值. match语句的值是什么? 是match里面分支的值, 这是一个具有传递形式的等价)

这样更简洁且符合习惯一点. 注意 Link::Empty 分支完全失去了括号, 因为其中只有一个语句. 这是个很好的简写.

>(译者: 只有一条语句时, 花括号可以省略, 这是在很多语言中都常见的语法糖, 最常见的情况例如在匿名函数中)

```text
cargo build

   Finished dev [unoptimized + debuginfo] target(s) in 0.22s
```

很好, 它正常工作了!



[ownership]: first-ownership.html
[diverging]: https://doc.rust-lang.org/nightly/book/ch19-04-advanced-types.html#the-never-type-that-never-returns
