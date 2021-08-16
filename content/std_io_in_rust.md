+++

title = "std::io"
date = 2021-08-15

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++


ref: [Rust 中的文件操作](https://juejin.cn/post/6926143816734867470)


## Seek 如何使用

Definition
```rust
pub enum SeekFrom {
    //Sets the offset to the provided number of bytes.
    Start(u64),
	// Sets the offset to the size of this object plus the specified number of bytes.
	// It is possible to seek beyond the end of an object, but it’s an error to seek before byte 0.
    End(i64),
    Current(i64),
}

impl File {
    /// Seek to an offset, in bytes, in a stream.
    ///
    /// A seek beyond the end of a stream is allowed, but behavior is defined
    /// by the implementation.
    ///
    /// If the seek operation completed successfully,
    /// this method returns the new position from the start of the stream.
    /// That position can be used later with [`SeekFrom::Start`].
    ///
    /// # Errors
    ///
    /// Seeking can fail, for example because it might involve flushing a buffer.
    ///
    /// Seeking to a negative offset is considered an error
	fn seek(&mut self, pos: SeekFrom) -> Result<u64>;
}
```

可以把一个文件视为一个字节流（或者说是一个字节数组），假设起始位置是文件的开头，那么结束位置就是文件的结尾。

比如一个文件foo.txt有10字节长（数据分别为1,2,3,4,5,6,7,8,9,0），在这个文件上进行一些操作，具体的代码和结果如下所示。第一个尝试是采用 SeekFrom::Start(0)作为pos，意图是重定位到文件流的开头（index=0）。

```rust
let mut buffer = [0; 10];
let file = File::open("foo.txt");
let pos = file.seek(SeekFrom::Start(0));
let n = f.read(&mut buffer)?;
println!("The bytes: {:?}", &buffer[..n]);

// output
// len: 10, The bytes: [49, 50, 51, 52, 53, 54, 55, 56, 57, 48]
```
下面采用SeekFrom::Start(5)，offset会被设置到index=5的位置，从这开始到结尾一共有5个字节。
```rust
let mut buffer = [0; 10];
let file = File::open("foo.txt");
let pos = file.seek(SeekFrom::Start(5));
let n = f.read(&mut buffer)?;
println!("The bytes: {:?}", &buffer[..n]);

// output
// len: 5, The bytes: [54, 55, 56, 57, 48]
```

Seek超过范围也是可以执行的，只是已经到了文件的末尾，读取不到任何数据。
```rust
let mut buffer = [0; 10];
let file = File::open("foo.txt");
let pos = file.seek(SeekFrom::Start(12));
let n = f.read(&mut buffer)?;
println!("The bytes: {:?}", &buffer[..n]);

// output
// len: 0, The bytes: []
```

SeekFrom::End和Start类似，只是End的基础位置是在文件字节流的最后一个字节之后，假设文件字节流长度为n，那么最后一个元素的index=n-1，则End的基础位置index=n，End中的元素也是直接加在offset上的。End还有一个用处，就是用来获取文件字节流的长度，具体例子如下所示。

```rust
let mut buffer = [0; 10];
let file = File::open("foo.txt");
let pos = file.seek(SeekFrom::End(0));
let n = f.read(&mut buffer)?;
println!("pos: {:?}", pos);

// output
// pos: 10
```


TODO
- 补一个图（一维数组的图）
