# 测试

我们已经写好了 `push` 和 `pop`, 现在我们可以来测试我们的堆了! Rust 和 cargo 的 Test 功能是一流的, 用着非常简单. 我们要做的仅仅是写一个函数, 然后对它进行Test相关标注
`#[test]`.

一般来说, 我们会尝试将测试放在代码的旁边. 但我们也经常为Test创造一个新的作用域, 来避免与真正的代码发生冲突. 就像我们用 `mod` 来指定
`first.rs` 应该包含在 `lib.rs` 中一样, 我们同样可以使用 `mod` 来创建一个新的 *内联文件* :

>(译者: 这段读着非常拗口, 实际上就是说Test可以写在专门的mod中, 以及mod的两种用法, 一是可以指定一个文件包含进来, 例如在lib.rs中写一句`pub mod first;` 就可以把first.rs文件作为mod包含进来。第二种是在当前文件直接开辟一个作用域, 将mod的内容写在其中)


```rust ,ignore
// in first.rs

mod test {
    #[test]
    fn basics() {
        // TODO
    }
}
```

然后我们调用 `cargo test` 指令.

```text
> cargo test
   Compiling lists v0.1.0 (/Users/ABeingessner/dev/temp/lists)
    Finished dev [unoptimized + debuginfo] target(s) in 1.00s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

哇 我们的空测试通过了! 我们向其中加一些内容. 我们将使用 `assert_eq!` 宏. 它并不是什么奇特的魔法. 它仅仅是对我们传入的两个参数判等, 如果不相等就抛出panic. 

```rust ,ignore
mod test {
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
        assert_eq!(list.pop(), Some(3));
        assert_eq!(list.pop(), Some(2));

        // Push some more just to make sure nothing's corrupted
        list.push(4);
        list.push(5);

        // Check normal removal
        assert_eq!(list.pop(), Some(5));
        assert_eq!(list.pop(), Some(4));

        // Check exhaustion
        assert_eq!(list.pop(), Some(1));
        assert_eq!(list.pop(), None);
    }
}
```

```text
> cargo test

error[E0433]: failed to resolve: use of undeclared type or module `List`
  --> src/first.rs:43:24
   |
43 |         let mut list = List::new();
   |                        ^^^^ use of undeclared type or module `List`


```

哦! 因为我们创造了一个新的 module, 我们应该明确的将List放进去才能使用它.

```rust ,ignore
mod test {
    use super::List;
    // everything else the same
}
```

```text
> cargo test

warning: unused import: `super::List`
  --> src/first.rs:45:9
   |
45 |     use super::List;
   |         ^^^^^^^^^^^
   |
   = note: #[warn(unused_imports)] on by default

    Finished dev [unoptimized + debuginfo] target(s) in 0.43s
     Running /Users/ABeingessner/dev/lists/target/debug/deps/lists-86544f1d97438f1f

running 1 test
test first::test::basics ... ok

test result: ok. 1 passed; 0 failed; 0 ignored; 0 measured; 0 filtered out
; 0 filtered out
```

好耶!

这个waring是怎么回事...? 我们明细已经使用了List!

...但是, 仅仅是在测试的时候! 为了安抚编译器 (并且为了对我们的用户友好), 我们应该表明整个 `test` 模块应该仅仅在我们运行测试的时候编译.


```rust ,ignore
#[cfg(test)]
mod test {
    use super::List;
    // everything else the same
}
```

这就是测试的全部内容!

>(译者: `#[cfg(test)]` 是一个属性, 你可以看作类似java的注解, 它会告诉编译器, 这个mod仅做测试用途, 在正式编译的时候会忽视)