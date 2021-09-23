# 泄漏

基于所有权的资源管理是为了简化组合：你在创建对象时获得资源，在对象被销毁时释放资源。由于销毁是自动为你处理的，这意味着你不能忘记释放资源，而且会尽快地释放！当然这很完美，我们所有的问题都解决了…………么？

一切都很糟糕，我们有新的、奇特的问题需要去解决。

很多人相信 Rust 能防止资源泄漏。在实践中，这基本上是对的。如果你看到一个安全的 Rust 程序以不受控制的方式泄漏资源，你会感到惊讶。

然而从理论的角度来看，无论你怎么看，都绝对不是这样的。在最严格的意义上，“泄漏”是如此抽象，以至于无法预防。在程序开始时初始化一个集合，用大量带有析构器的对象填充它，然后进入一个从未引用过它的无限事件循环，这是非常容易的。这个集合将毫无用处地坐着，守着它宝贵的资源，直到程序终止（无论如何，这时所有这些资源都会被操作系统回收）。

我们可以考虑一种更有限的泄漏形式：未能丢弃一个无法到达的值。Rust 也没有防止这种情况。事实上，Rust *有一个函数可以做到这一点*。`mem::forget`。这个函数消耗它所传递的值，*然后不运行它的析构器*。

在过去，`mem::forget`被标记为不安全，作为对使用它的一种提示，因为不调用一个析构器通常不是一件好的事情（尽管对一些特殊的不安全代码很有用）。然而，这通常被认为是一种站不住脚的立场：在安全代码中，有很多方法可以不调用析构函数。最著名的例子是使用内部可变性创建一个引用计数指针的循环引用。

对于安全代码来说，假设析构器的泄漏不会发生是合理的，因为任何泄漏析构器的程序都可能是错误的。然而，*不安全的*代码不能依赖析构器的运行来保证安全。对于大多数类型来说，这并不重要：如果你泄露了析构函数，那么根据定义，该类型是不可访问的，所以这并不重要，对吗？例如，如果你泄露了一个`Box<u8>`，那么你会浪费一些内存，但这几乎不会违反内存安全。

然而，我们必须注意的是*代理*类型的解构器泄露。这些类型管理对一个独立对象的访问，但实际上并不拥有它。代理对象是相当罕见的，你需要关注的代理对象就更少了。我们将专注于标准库中三个有趣的例子：

* `vec::Drain`
* `Rc`
* `thread::scoped::JoinGuard`

## Drain

`drain`是一个 collections API，它将数据从容器中移出而不消耗容器。这使我们能够在对一个`Vec`的所有内容都获得所有权后重新使用其底层的内存分配。它产生了一个迭代器（Drain），并按值返回 Vec 的内容。

现在，考虑一下迭代中的 Drain：一些值已经被移出，而另一些还没有。这意味着 Vec 的一部分现在充满了逻辑上未初始化的数据! 我们可以在每次移出一个值的时候对 Vec 中的所有元素进行后移，但这将会产生非常灾难性的性能后果。

相反，我们希望 Drain 能在 Vec 被删除时修复它底层需要的内存分配（译者注：也就是 Vec 的内存分配）。它应该自己运行直到完成，并回移任何没有被移除的元素（drain 支持子范围），然后修复 Vec 的`len`。它甚至是 unwind 安全的。很简单!

现在考虑下面的情况：

<!-- ignore: simplified code -->
```rust,ignore
let mut vec = vec![Box::new(0); 4];

{
    // start draining, vec can no longer be accessed
    let mut drainer = vec.drain(..);

    // pull out two elements and immediately drop them
    drainer.next();
    drainer.next();

    // get rid of drainer, but don't call its destructor
    mem::forget(drainer);
}

// Oops, vec[0] was dropped, we're reading a pointer into free'd memory!
println!("{}", vec[0]);
```

这很明显不是好事。不幸的是，我们正处于两难境地：在每一步保持一致的状态有巨大的成本（并且会抵消 API 带来的任何好处）。如果不能保持一致的状态，我们就会在安全代码中出现未定义的行为（使 API 不健全）。

那么我们能做什么呢？好吧，我们可以选择一个微弱的一致性状态：当我们开始迭代时，将 Vec 的 len 设置为 0，并在必要时在析构器中修复它。这样一来，如果一切执行正常，我们就能以最小的开销获得所需的行为。但是如果有人*胆敢*在迭代过程中 forget 了我们，那打不了就是*泄露更多*（并且可能让 Vec 处于一个虽然意外的但其他方面保持一致的状态）。既然我们已经接受了 mem::forget 是安全的，那么这就必须绝对是安全的。我们把一个泄漏导致更多的泄漏称为*泄漏放大*。

## Rc

Rc 是一个有趣的例子，因为乍一看，它似乎根本就不是一个代理值。毕竟，它管理着它所指向的数据，丢掉一个值的所有 Rcs 就会丢掉这个值。泄露一个 Rc 似乎并不特别危险。它将使 refcount 永久增加，并阻止数据被释放或丢弃，但这似乎就像 Box，对吗？

并不是这样。

让我们考虑一下 Rc 的一个简化实现：

<!-- ignore: simplified code -->
```rust,ignore
struct Rc<T> {
    ptr: *mut RcBox<T>,
}

struct RcBox<T> {
    data: T,
    ref_count: usize,
}

impl<T> Rc<T> {
    fn new(data: T) -> Self {
        unsafe {
            // Wouldn't it be nice if heap::allocate worked like this?
            let ptr = heap::allocate::<RcBox<T>>();
            ptr::write(ptr, RcBox {
                data: data,
                ref_count: 1,
            });
            Rc { ptr: ptr }
        }
    }

    fn clone(&self) -> Self {
        unsafe {
            (*self.ptr).ref_count += 1;
        }
        Rc { ptr: self.ptr }
    }
}

impl<T> Drop for Rc<T> {
    fn drop(&mut self) {
        unsafe {
            (*self.ptr).ref_count -= 1;
            if (*self.ptr).ref_count == 0 {
                // drop the data and then free it
                ptr::read(self.ptr);
                heap::deallocate(self.ptr);
            }
        }
    }
}
```

这段代码包含了一个隐含的、微妙的假设：`ref_count`可以装入`usize`，因为内存中的 Rcs 不能超过`usize::MAX`。然而这本身就假设`ref_count`准确反映了内存中的 Rcs 数量，我们知道用`mem::forget`是错误的。使用`mem::forget`我们可以溢出`ref_count`，然后用大量的 Rcs 将其降至 0。然后我们就可以愉快地对内部数据进行 use-after-free 了。负负得正？

这个问题可以通过检查`ref_count`并做一些防御来解决。标准库的立场是直接 abort，因为你的程序肯定是摊上事儿了，摊上大事儿了。卧槽，这真是一个可笑的边界情况。

## thread::scoped::JoinGuard

> 译者注：实际上这个 API 很早就从标准库中删除了，具体原因可以参考 https://github.com/rust-lang/rust/issues/24292。
>
> 原文也有人提过 issue 询问是否可以删除，得到了答复说，这个例子仍然是非常重要的，所以保留了下来：https://github.com/rust-lang/nomicon/issues/57。

thread::scoped API 旨在允许引用其父线程栈上的数据的线程被创建出来，而不需要对这些数据进行任何同步。它确保父线程在任何共享数据失效之前 join 子线程。

<!-- ignore: simplified code -->
```rust,ignore
pub fn scoped<'a, F>(f: F) -> JoinGuard<'a>
    where F: FnOnce() + Send + 'a
```

这里`f`是一些闭包，供其他线程执行。这里我们定义`F: Send +'a`意思是它捕获了生命周期为`'a`的数据，而且它要么拥有该数据，要么该数据是`Sync`的（暗示`&data`是`Send`）。

因为 JoinGuard 有一个生命周期，它通过借用捕获了所有它需要的父线程中的数据。这意味着 JoinGuard 不能超过其他线程正在处理的数据的生命周期。当*JoinGuard*被丢弃时，它会 block 父线程，确保子线程中捕获的数据在父线程中 drop 之前失效。

使用方法看起来像这样：

<!-- ignore: simplified code -->
```rust,ignore
let mut data = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10];
{
    let guards = vec![];
    for x in &mut data {
        // Move the mutable reference into the closure, and execute
        // it on a different thread. The closure has a lifetime bound
        // by the lifetime of the mutable reference `x` we store in it.
        // The guard that is returned is in turn assigned the lifetime
        // of the closure, so it also mutably borrows `data` as `x` did.
        // This means we cannot access `data` until the guard goes away.
        let guard = thread::scoped(move || {
            *x *= 2;
        });
        // store the thread's guard for later
        guards.push(guard);
    }
    // All guards are dropped here, forcing the threads to join
    // (this thread blocks here until the others terminate).
    // Once the threads join, the borrow expires and the data becomes
    // accessible again in this thread.
}
// data is definitely mutated here.
```

原则上，这完全是可行的！Rust 的所有权系统完美地保证了这一点！……只是它必须依赖于一个保证被调用到的析构器才是安全的。

<!-- ignore: simplified code -->
```rust,ignore
let mut data = Box::new(0);
{
    let guard = thread::scoped(|| {
        // This is at best a data race. At worst, it's also a use-after-free.
        *data += 1;
    });
    // Because the guard is forgotten, expiring the loan without blocking this
    // thread.
    mem::forget(guard);
}
// So the Box is dropped here while the scoped thread may or may not be trying
// to access it.
```

在这里，一个会运行的析构器对 API 来说是非常基本的。因此它不得不被废弃，而采用完全不同的设计。