# Drop 实现

和可变链表一样, 我们存在一个递归析构函数的问题.
诚然, 这对于不可变链表不是一个很糟的问题: 如我们在某处遇到其他链表的head, 我们不会直接drop它. 
然而我们还是知道如何处理. 这是我们之前解决的方法:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut cur_link = self.head.take();
        while let Some(mut boxed_node) = cur_link {
            cur_link = boxed_node.next.take();
        }
    }
}
```

问题在于循环体:

```rust ,ignore
cur_link = boxed_node.next.take();
```

这将改变Box中的Node, 但我们不能对Rc这样做; 
它只给我们共享访问权, 因为任何其他Rc都可能指向它.

但是如果我们知道这是这个链表所独有的node, 那么把这个node移出Rc就好了. 然后我们也可以知道什么时候停止: 当我们不能移出 Node.

Rc有一个方法可以做到这一点: `try_unwrap`:

```rust ,ignore
impl<T> Drop for List<T> {
    fn drop(&mut self) {
        let mut head = self.head.take();
        while let Some(node) = head {
            if let Ok(mut node) = Rc::try_unwrap(node) {
                head = node.next.take();
            } else {
                break;
            }
        }
    }
}
```

>(注: Rc::try_unwrap, 注释中提到 Returns the inner value, if the `Rc` has exactly one strong reference.)

```text
cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/too-many-lists/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.10s
     Running /Users/ABeingessner/dev/too-many-lists/lists/target/debug/deps/lists-86544f1d97438f1f

running 8 tests
test first::test::basics ... ok
test second::test::basics ... ok
test second::test::into_iter ... ok
test second::test::iter ... ok
test second::test::iter_mut ... ok
test second::test::peek ... ok
test third::test::basics ... ok
test third::test::iter ... ok

test result: ok. 8 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
```

很好.
