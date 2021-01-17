# Peek 方法

上次我们没有费力去实现 Peek 方法. 现在让我们来实现它. 
我们需要做的仅仅是返回一个List 头部元素的引用, 如果它存在的话. 
听起来很简单, 让我们来试一下:

```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.map(|node| {
        &node.elem
    })
}
```


```text
> cargo build

error[E0515]: cannot return reference to local data `node.elem`
  --> src/second.rs:37:13
   |
37 |             &node.elem
   |             ^^^^^^^^^^ returns a reference to data owned by the current function

error[E0507]: cannot move out of borrowed content
  --> src/second.rs:36:9
   |
36 |         self.head.map(|node| {
   |         ^^^^^^^^^ cannot move out of borrowed content


```

*唉*. 又怎么了, 编译器大哥?

Map 拿到的是 `self` 的值本身, 它将消耗掉 Option.
以前这样做是很不错的, 我们仅仅是想把值给 `take` 出去, 
但是这里, 我们想把它留在原处. 
处理这个问题 *正确* 方法是使用如下 Option 定义的 `as_ref`:

```rust ,ignore
impl<T> Option<T> {
    pub fn as_ref(&self) -> Option<&T>;
}
```

`Option<T>` 的 `as_ref` 可以返回一个 `Option<&T>`.

>(译者: 原文读着逻辑很奇怪: "我们可以怎么怎么做, 但是不. 这将意味着什么什么, 但是多亏了有什么什么的帮助." 我实在看不出它到底要不要做- -)

>It demotes the Option<T> to an Option to a reference to its internals. We could do this ourselves with an explicit match but *ugh no*. It does mean that we need to do an extra dereference to cut through the extra indirection, but thankfully the `.` operator handles that for us.


```rust ,ignore
pub fn peek(&self) -> Option<&T> {
    self.head.as_ref().map(|node| {
        &node.elem
    })
}
```

```text
cargo build

    Finished dev [unoptimized + debuginfo] target(s) in 0.32s
```

我们同样可以通过 `as_mut` 做一个 *可变* 版的:

```rust ,ignore
pub fn peek_mut(&mut self) -> Option<&mut T> {
    self.head.as_mut().map(|node| {
        &mut node.elem
    })
}
```

```text
lists::cargo build

```

很简单

别忘了加个测试:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
}
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

很棒, 但是我们还没有测试是否可以改变 `peek_mut` 的返回值. 
让我们试试用 `map` 在我们得到的 `Option<&mut T>` 放一个值:

```rust ,ignore
#[test]
fn peek() {
    let mut list = List::new();
    assert_eq!(list.peek(), None);
    assert_eq!(list.peek_mut(), None);
    list.push(1); list.push(2); list.push(3);

    assert_eq!(list.peek(), Some(&3));
    assert_eq!(list.peek_mut(), Some(&mut 3));
    list.peek_mut().map(|&mut value| {
        value = 42
    });

    assert_eq!(list.peek(), Some(&42));
    assert_eq!(list.pop(), Some(42));
}
```

```text
> cargo test

error[E0384]: cannot assign twice to immutable variable `value`
   --> src/second.rs:100:13
    |
99  |         list.peek_mut().map(|&mut value| {
    |                                   -----
    |                                   |
    |                                   first assignment to `value`
    |                                   help: make this binding mutable: `mut value`
100 |             value = 42
    |             ^^^^^^^^^^ cannot assign twice to immutable variable          ^~~~~
```

编译器抱怨 `value` 是不可变的, 但是我们明明写了 `&mut value`; 
怎么回事? 实际上像这样写闭包参数并, 没有指定 `value` 是一个可变引用. 
恰恰相反, 这创建了一个 `模式匹配`, 它将匹配传进闭包的参数; 
`|&mut value|` 的含义是 "参数是一个可变引用, 
但请把它的值复制进 `value`."  
如果我们仅仅只写 `|value|`,  `value` 的类型将是 `&mut i32` 
然后我们就可以修改head:

```rust ,ignore
    #[test]
    fn peek() {
        let mut list = List::new();
        assert_eq!(list.peek(), None);
        assert_eq!(list.peek_mut(), None);
        list.push(1); list.push(2); list.push(3);

        assert_eq!(list.peek(), Some(&3));
        assert_eq!(list.peek_mut(), Some(&mut 3));

        list.peek_mut().map(|value| {
            *value = 42
        });

        assert_eq!(list.peek(), Some(&42));
        assert_eq!(list.pop(), Some(42));
    }
```

```text
cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 3 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::peek ... ok

test result: ok. 3 passed; 0 failed; 0 ignored; 0 measured

```

好很多了!


>译者: 实际上这里提到一个概念, 闭包定义时写的参数属于模式匹配. 模式匹配通常还伴随一个词 "解构". 这个东西在match中经常被用到, 元组解包也是一种解构. 这是一个非常智能的功能.

> 举个简单的例子, 我们可以通过如下代码, 自动定义好a, b, y:
```
struct Foo { x: (u32, u32), y: u32 }
let foo = Foo { x: (1, 2), y: 3 };
let Foo { x: (a, b), y } = foo;
```

> 上面属于结构体的解构, 再例如上文闭包的例子:

```
list.peek_mut().map(|value| {
    *value = 42
});
```
> 如果你有用类似 Rust-Analyzer 之类的插件, 你实际上看到的可能是.
```
list.peek_mut().map(|value: &mut i32| {
    *value = 42
});
```
> 那如果改一下闭包的参数到之前错误的写法呢?
```
list.peek_mut().map(|&mut value: i32| {
    *value = 42
});
```
> 我们定义了 `value` 参数, 但它实际上是 `&mut i32` 类型. 我们定义了 `&mut value` 参数, 但实际上它是 `i32` 类型. 我们可以分步来看. 首先map之间的内容 `list.peek_mut()` 它代表一个 `Option<&mut i32>`类型的值. 

> map方法会将Option自身的内容: `&mut i32` 作为参数传到我们写的闭包中. 实际上闭包参的类型已经由map预定好了, 它一定会传一个`&mut i32`. 以前我们写函数, 都是定义好需求方的要求, 等着供给方来满足. 但在这里恰恰相反, 供给方很强硬, 明确表示了自己一定要提供一个 `&mut i32` 参数. 你作为需求方只能被迫接受. 如果你只写 `|value|`, 代表将传过来的参数全盘接受, value能代表参数的全部, 也就是 `&mut i32`. 而如果是 `|&mut value|` 呢? 它表示我知道你要传的参数具有 `&mut` 的特点, 而我要的是我已经表明了的内容之外的东西, 请把它们放到value中.

> 其实上面这一大段都是废话, 我们可以用个最简单的方法. 也就是写等式. 已知 &mut i32 = &mut value . 问 value = ?