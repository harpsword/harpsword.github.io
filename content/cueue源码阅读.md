+++

title = "cueue包阅读-一种高性能、单生产者单消费者、环形缓存channel"
date = 2022-10-26

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++

# 包介绍

cueue包实现了一个高性能的、单生产者、单消费者、环形缓存的通道，支持无锁的原子（批）读写，适合跨线程的数据流动。

[github link](https://github.com/erenon/cueue)

# 存储设计
cueue中的存储如下图所示。一共存在两种不同的存储memory和file，对于control block，两边是单独采用mmap进行映射；对于buf，将两块连续的memory都映射到文件的同一块区域，目的是为了环型缓存下依旧可以获取一个连续的内存空间。
![](/image/cueue_memory1.png)
其中memory的两块绿色的buf和file中黄色的区域大小是相同的。


# 读写设计
ControlBlock是读写之间共享的结构体，ControlBlock的结构中只有两个原子变量，这个结构体会由Writer和Reader共同持有。
```rust
/// The shared metadata of a Cueue.
///
/// Cueue is empty if R == W
/// Cueue is full if W == R+capacity
/// Invariant: W >= R
/// Invariant: R + capacity >= W
#[derive(Default)]
struct ControlBlock {
    write_position: CacheLineAlignedAU64,
    read_position: CacheLineAlignedAU64,
}
```

## Writer
```rust
/// Writer of a Cueue.
///
/// See examples/ for usage.
pub struct Writer<T> {
    mem: std::sync::Arc<MemoryMapInitialized<T>>,
    cb: *mut ControlBlock,
    mask: u64,  // cap - 1, 当index超过了cap之后, index & mask 可以用来获取真实的index

    buffer: *mut T, // buf 内存的起点
    write_begin: *mut T, // 考虑 真实index后的内存写入位置
    write_capacity: usize, // 还剩多少空间
}
```
write_chunk的实现如下所示：
```rust
    /// Get a writable slice of maximum available size.
    ///
    /// The elements in the returned slice are either default initialized
    /// (never written yet) or are the result of previous writes.
    /// The writer is free to overwrite or reuse them.
    ///
    /// After write, `commit` must be called, to make the written elements
    /// available for reading.
    pub fn write_chunk(&mut self) -> &mut [T] {
	    // 获取读、写的位置
        let w = self.write_pos().load(Ordering::Relaxed); 
        let r = self.read_pos().load(Ordering::Acquire);

        debug_assert!(r <= w);
        debug_assert!(r + self.capacity() as u64 >= w);
		// 获取写入的真实index，& self.mask 相当于 % cap
        let wi = w & self.mask;
        // w.wrapping_sub(r)计算得到使用的内存空间大小
        self.write_capacity = (self.capacity() as u64 - (w.wrapping_sub(r))) as usize;

        unsafe {
            self.write_begin = self.buffer.offset(wi as isize);
            // 由于特殊的存储设计，即使 write_begin在 第一个buffer的结束位置
            // 也能拿到对应的存储
            std::slice::from_raw_parts_mut(self.write_begin, self.write_capacity)
        }
    }
```
如下图所示，当wi=ri的时候，返回的chunk即黑色的那一块内存，对于使用方来说这是一个可变的slice。由于mmap映射了buf(2)，对应的数据也会出现在buf的前面一段。
![](/image/cueue_memory2.png)

写入完成之后，可以调用Writer.commit来提交写入，这里会修改controlBlock中的`write_position`，这样Reader中才能知道有数据写入。
```rust
    /// Make `n` number of elements, written to the slice returned by `write_chunk`
    /// available for reading.
    ///
    /// `n` is checked: if too large, gets truncated to the maximum committable size.
    ///
    /// Returns the number of committed elements.
    pub fn commit(&mut self, n: usize) -> usize {
        let m = usize::min(self.write_capacity, n);
        unsafe {
            self.unchecked_commit(m);
        }
        m
    }

    unsafe fn unchecked_commit(&mut self, n: usize) {
        let w = self.write_pos().load(Ordering::Relaxed);
        self.write_begin = self.write_begin.add(n);
        self.write_capacity -= n;
        self.write_pos().store(w + n as u64, Ordering::Release);
    }
```

## Reader
```rust
/// Reader of a Cueue.
///
/// See examples/ for usage.
pub struct Reader<T> {
    mem: std::sync::Arc<MemoryMapInitialized<T>>,
    cb: *mut ControlBlock,
    mask: u64,

    buffer: *const T,
    read_begin: *const T,
    read_size: u64,
}
```
Reader中的逻辑和Writer类似，尤其是read_chunk的逻辑，只是返回的数据类型是一个不可变的slice。
```rust
    /// Return a slice of elements written and committed by the Writer.
    pub fn read_chunk(&mut self) -> &[T] {
        let w = self.write_pos().load(Ordering::Acquire);
        let r = self.read_pos().load(Ordering::Relaxed);

        debug_assert!(r <= w);
        debug_assert!(r + self.capacity() as u64 >= w);

        let ri = r & self.mask;

        self.read_size = w - r;

        unsafe {
            self.read_begin = self.buffer.offset(ri as isize);
            std::slice::from_raw_parts(self.read_begin, self.read_size as usize)
        }
    }
```
数据读取完之后也需要提交
```rust
    /// Mark the slice previously acquired by `read_chunk` as consumed,
    /// making it available for writing.
    pub fn commit(&mut self) {
        let r = self.read_pos().load(Ordering::Relaxed);
        let rs = self.read_size;
        self.read_pos().store(r + rs, Ordering::Release);
    }
```

# 一些细节

## 系统调用
这里就不详细说明了，主要起作用的系统调用是mmap。
1. ftruncate
2. mmap
3. munmap
4. sysconf
5. mkstemp(macos)
6. shm_open(macos)
7. memfd_create(linux)


## 内存顺序
考虑多线程情况下的Reader和Writer访问原子变量的情况。如下图所示
![](/image/cueue_sync.png)
可以看到Writer采用release的order来存储`write pos`，当Reader按照acquire ordering读取`write pos`，假如读到了`write pos`的本次变化，那么之后在内存中也能读到对应的数据。同理，当Writer按照acquire ordering读取`read pos`，假如读到了`read pos`，也能确认内存中的这块数据被读取了（或者说适用方手动调用了commit）。

Release-Acquire in C++ Reference:
>If an atomic store in thread A is tagged memory_order_release and an atomic load in thread B from the same variable is tagged memory_order_acquire, all memory writes (non-atomic and relaxed atomic) that _happened-before_ the atomic store from the point of view of thread A, become _visible side-effects_ in thread B. That is, once the atomic load is completed, thread B is guaranteed to see everything thread A wrote to memory. This promise only holds if B actually returns the value that A stored, or a value from later in the release sequence.


# 学到的内容

1. 利用mmap把两块连续的内存和同一个文件绑定，即使是ring buffer也能对外提供类似数组（slice）的体验，提供批处理的能力。
2. Release-Acquire的合理使用。Release可以认为是把写入发布出去，Acquire可以理解是尝试读取。

# Another Similar Crate: rtrb


设容量为C，存储的大小也为C。有两个指针，head和tail，取值在$[0, 2*C-1]$。存储的图如下所示，黄色的块是存储，其大小为C，pointer range的大小是2C。
![rtrb](/image/rtrb.png)

这两个指针会被映射到Memory上的位置，映射方式如下代码所示（是一种ring buffer的实现方式）。

```rust
    /// Wraps a position from the range `0 .. 2 * capacity` to `0 .. capacity`.
    fn collapse_position(&self, pos: usize) -> usize {
        debug_assert!(pos == 0 || pos < 2 * self.capacity);
        if pos < self.capacity {
            pos
        } else {
            pos - self.capacity
        }
    }
```

Writer和Reader会维护head和tail这两个指针，来实现安全的读写。

|        | atomic head | atomic tail       |
| ------ | ----------- | ---------- |
| Writer | only load   | only write |
| Reader | only write  | only load  |

Writer和Reader内部还会维护各自的head和tail指针，其中Writer需要时不时加载atomic head，Reader需要时不时加载atomic tail。

为什么只需要时不时加载？而不是一直读取atomic tail/head？
1. 对于Writer，不断push的时候，只需要根据本地的head和tail来判断是否buffer满了。因为即使Reader在更新atomic head，也只会先出现本地head+atomic tail算出的buffer使用容量先到cap。
2. 对于Reader逻辑类似。