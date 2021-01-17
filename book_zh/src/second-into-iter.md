# IntoIter 方法

Rust中通过 *Iterator* trait 使集合变得可迭代. 它比 `Drop` 要复杂点:

```rust ,ignore
pub trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}
```

你会发现一个新东西: `type Item`. 
这是声明 Iterator 的每一个实现, 都有一个 *相关类型* 叫作 Item. 
在这种情况下,它代表当你调用 `next` 时, 它可以吐出这种类型.

Iterator 可以生成 `Option<Self::Item>` 是因为接口整合了 `has_next` 和 `get_next` 的概念. 当存在下一个值时, 将产生 `Some(value)`, 反之则产生 `None`. 
这使得 API 更符合人体工程学, 且使用和实现起来更安全, 
同时避免 `has_next` 和 `get_next` 之间冗余的判断和逻辑. Nice!

很遗憾, Rust 没有像 `yield` 一样的语法 (至今), 所以我们不得不自己手动实现逻辑. 另外, 每个集合实际上都应该努力去实现三种 iterator:

* IntoIter - `T`
* IterMut - `&mut T`
* Iter - `&T`

我们实际上已经准备好了工具, 在我们 List 的接口中: 只需要一遍又一遍的调用 `pop`. 就像这样, 我们仅仅需要实现个 IntoIter 作为来包装 List 的新类型:


```rust ,ignore
// 元组结构体是结构体的替代品,
// 非常适合来包装其他琐碎的类型.
pub struct IntoIter<T>(List<T>);

impl<T> List<T> {
    pub fn into_iter(self) -> IntoIter<T> {
        IntoIter(self)
    }
}

impl<T> Iterator for IntoIter<T> {
    type Item = T;
    fn next(&mut self) -> Option<Self::Item> {
        // 以数字方式访问元组结构体的字段
        self.0.pop()
    }
}
```

让我们写一段测试:

```rust ,ignore
#[test]
fn into_iter() {
    let mut list = List::new();
    list.push(1); list.push(2); list.push(3);

    let mut iter = list.into_iter();
    assert_eq!(iter.next(), Some(3));
    assert_eq!(iter.next(), Some(2));
    assert_eq!(iter.next(), Some(1));
    assert_eq!(iter.next(), None);
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 4 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::peek ... ok

test result: ok. 4 passed; 0 failed; 0 ignored; 0 measured

```

很好!
