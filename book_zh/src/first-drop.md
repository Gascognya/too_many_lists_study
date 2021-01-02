# Drop

我们可以做一个 stack, push 进去, pop 出来, 并且我们已经测试过了, 一切正常!

我们需要对清理我们的list担心么? 严格来说, 不需要, 完全不需要!
如同 C++, Rust 使用析构函数(destructors)来自动清理我们不再需要的资源. 一个类型如果实现了名为Drop的 *trait*.
Trait 是 Rust 对接口的叫法. Drop trait 代表下列接口:

```rust ,ignore
pub trait Drop {
    fn drop(&mut self);
}
```

基本上, "当你离开作用域时, 我将给你一点时间处理你的事务"(这句话是对资源说的).

实际上你不需要Drop, 如果你包含了实现Drop的类型, 你所需要做的就是调用 *它们* 的析构函数. 在 List 的例子中, 它所需要做的仅仅是 drop 它的 head, 而head又 *可能* 需要 drop 一个 `Box<Node>`. 所有这些都自动为我们解决了... 除了一个小问题.

自动处理将会变得很糟糕.

让我们来考虑一个简单的List:


```text
list -> A -> B -> C
```

当 `list` drop 时, 它将尝试 drop A, 然后又将尝试 drop B,
再然后是 drop C. 你可能感到有些紧张. 这是一段递归代码, 递归代码可能导致栈溢出!

你也可能在想 "这显然是尾递归, 任何像样的语言都应该确保这样的代码不会让栈崩溃". 事实上, 你错了! 来看看为什么, 让我们试着写出编译器要做什么, 通过像编译器那样为我们的 List 手动执行 Drop :


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        // NOTE: 你不能在真正的Rust代码中显式地调用“drop”;
        // 我们假装自己是编译器!
        self.head.drop(); // 尾递归-很好!
    }
}

impl Drop for Link {
    fn drop(&mut self) {
        match *self {
            Link::Empty => {} // 什么都不做!
            Link::More(ref mut boxed_node) => {
                boxed_node.drop(); // 尾递归-很好!
            }
        }
    }
}

impl Drop for Box<Node> {
    fn drop(&mut self) {
        self.ptr.drop(); // 哦, 不是尾递归
        deallocate(self.ptr);
    }
}

impl Drop for Node {
    fn drop(&mut self) {
        self.next.drop();
    }
}
```

我们 *不能* drop Box 里的内容在解除分配(deallocating)之后, 所以没有办法以尾递归的方式删除!相反，我们将不得不手动编写一个 `List` 的迭代drop，将节点从它们的盒子中吊出来。


```rust ,ignore
impl Drop for List {
    fn drop(&mut self) {
        let mut cur_link = mem::replace(&mut self.head, Link::Empty);
        // `while let` == "while持续到无法匹配为止"
        while let Link::More(mut boxed_node) = cur_link {
            cur_link = mem::replace(&mut boxed_node.next, Link::Empty);
            // boxed_node超出作用域并在这里被删除;
            // 但是他的节点 `next` 字段将被设置为Link::Empty
            // 所以没有无界递归
        }
    }
}
```

```text
> cargo test

     Running target/debug/lists-5c71138492ad4b4a

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured

```

很好!

----------------------


## 提前优化的奖励部分!

我们实现的 drop 与 `while let Some(_) = self.pop() { }`  *十分* 相似, 这个当然更简单. 它们之间有什么不同, 一旦我们开始泛化列表来存储整数以外的东西会导致什么性能问题呢?

<details>
  <summary>点击获得答案</summary>
(太累了所以略过答案, 机翻可能有失原意, 留下原文)

Pop returns `Option<i32>`, while our implementation only manipulates Links (`Box<Node>`). So our implementation only moves around pointers to nodes, while the pop-based one will move around the values we stored in nodes. This could be very expensive if we generalize our list and someone uses it to store instances of VeryBigThingWithADropImpl (VBTWADI). Box is able to run the drop implementation of its contents in-place, so it doesn't suffer from this issue. Since VBTWADI is *exactly* the kind of thing that actually makes using a linked-list desirable over an array, behaving poorly on this case would be a bit of a disappointment.

If you wish to have the best of both implementations, you could add a new method,
`fn pop_node(&mut self) -> Link`, from-which `pop` and `drop` can both be cleanly derived.

</details>
