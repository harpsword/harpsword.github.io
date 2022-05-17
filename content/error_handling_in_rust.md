+++

title = "Error Handling in Rust"
date = 2021-08-15

[taxonomies]
categories = ["Rust"]
tags = ["Rust", "Error Handling"]

+++

Error Handling 主要有两种思路，一种是Try-Catch，另一种是通过Return Value，而Rust主要采用后一种。

主要的数据结构有Option、Result，具体的定义如下所示，Option主要用于处理存在None的情况，Result主要用于处理返回错误的情况。


```rust
enum Option<T> {
    None,
    Some(T),
}
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

## Option
Option的定位是处理存在Null的情况，在下述例子中，在字符串中查找某个字符，这个字符可能会不存在，这时候就会返回None。
接下来是调用函数find的部分，假如使用match来处理返回结果的话，rust的编译器会强制要求处理所有的情况，包括None和Some两种情况，这样不容易出错，比如遗漏了找不到字符的情况（None）。
```rust
// Searches `haystack` for the Unicode character `needle`. If one is found, the
// byte offset of the character is returned. Otherwise, `None` is returned.
fn find(haystack: &str, needle: char) -> Option<usize> {
    for (offset, c) in haystack.char_indices() {
        if c == needle {
            return Some(offset);
        }
    }
    None
}
```

接下来会列举一些Option常用的方法，这些方法是一些常见操作的定义，方便调用者使用。
```rust

impl<T> for Option<T> {
  // map
  // 当取值为some时，应用对应的函数
  pub fn map<U, F: FnOnce(T) -> U>(self, f: F) -> Option<U> {
      match self {
          Some(x) => Some(f(x)),
          None => None,
      }
  }

  // unwrap_or 
  // 返回Some中的值或者返回默认值default
  pub fn unwrap_or(self, default: T) -> T {
      match self {
          Some(x) => x,
          None => default,
      }
  }

  // unwarp_or_else 返回Some中的值，或者返回函数F返回的值
  pub fn unwrap_or_else<F: FnOnce() -> T>(self, f: F) -> T {
      match self {
          Some(x) => x,
          None => f(),
      }
  }

  // and_then 函数
  // if option is `None`, return None
  // else 执行函数f于包含的值x，并返回结果
  // 和 map不同之处在于调用的函数签名不同
  pub fn and_then<U, F: FnOnce(T) -> Option<U>>(self, f: F) -> Option<U> {
      match self {
          Some(x) => f(x),
          None => None,
      }
  }

  // take 函数
  /// Takes the value out of the option, leaving a [`None`] in its place.
  pub fn take(&mut self) -> Option<T> ;

  // as系列函数
  /// Converts from `&Option<T>` to `Option<&T>`.
  pub const fn as_ref(&self) -> Option<&T>;

  /// Converts from `&mut Option<T>` to `Option<&mut T>`.
  pub fn as_mut(&mut self) -> Option<&mut T>;

  /// Converts from `Option<T>` (or `&Option<T>`) to `Option<&T::Target>`.
  pub fn as_deref(&self) -> Option<&T::Target> {
      self.as_ref().map(|t| t.deref())
  }

  /// Converts from `Option<T>` (or `&mut Option<T>`) to `Option<&mut T::Target>`.
  pub fn as_deref_mut(&mut self) -> Option<&mut T::Target> {
      self.as_mut().map(|t| t.deref_mut())
  }
}

```

## Result
Result和Option一样都定义在标准库中，可以认为Option是Result的一个特例，具体的定义如下所示。Result可以用来表示可能返回错误的返回结果。

```rust
enum Result<T, E> {
    Ok(T),
    Err(E),
}
type Option<T> = Result<T, ()>;
```

简单罗列一下Result的一些方法，这些方法定义中的T和E是Result的泛型参数，具体如上述Result的定义。
```rust
impl<T, E: ::std::fmt::Debug> Result<T, E> {
    fn unwrap(self) -> T {
        match self {
            Result::Ok(val) => val,
            Result::Err(err) =>
              panic!("called `Result::unwrap()` on an `Err` value: {:?}", err),
        }
    }
}

```
?的使用如下所示
```rust
// ? 当Result结果是Err时会返回err　
let content = std::fs::read_to_string("test.txt")?;

// 等价于 标黄的部分
fn main() -> Result<(), Box<dyn std::error::Error>> {
    let result = std::fs::read_to_string("test.txt");
    let content = match result {
        Ok(content) => { content },
        Err(error) => { return Err(error.into()); }
    };
    println!("file content: {}", content);
    Ok(())
}
```

### 自定义Error
可以根据可能遇到的错误来定义自己的错误，一般采用enum类型，里面是不同的错误类型。

```rust
use std::io;
use std::num;

// We derive `Debug` because all types should probably derive `Debug`.
// This gives us a reasonable human readable description of `CliError` values.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}
```

## trait
错误处理中常用的两个trait：
```rust
use std::fmt::{Debug, Display};

trait Error: Debug + Display {
  /// A short description of the error.
  fn description(&self) -> &str;

  /// The lower level cause of this error, if any.
  fn cause(&self) -> Option<&Error> { None }
}
```


### Error trait
拿前面实现的CliError来展示一下如何实现，需要先实现Display方法，
```rust
use std::error;
use std::fmt;

impl fmt::Display for CliError {
    fn fmt(&self, f: &mut fmt::Formatter) -> fmt::Result {
        match *self {
            // Both underlying errors already impl `Display`, so we defer to
            // their implementations.
            CliError::Io(ref err) => write!(f, "IO error: {}", err),
            CliError::Parse(ref err) => write!(f, "Parse error: {}", err),
        }
    }
}

impl error::Error for CliError {
    fn description(&self) -> &str {
        // Both underlying errors already impl `Error`, so we defer to their
        // implementations.
        match *self {
            CliError::Io(ref err) => err.description(),
            // Normally we can just write `err.description()`, but the error
            // type has a concrete method called `description`, which conflicts
            // with the trait method. For now, we must explicitly call
            // `description` through the `Error` trait.
            CliError::Parse(ref err) => error::Error::description(err),
        }
    }

    fn cause(&self) -> Option<&error::Error> {
        match *self {
            // N.B. Both of these implicitly cast `err` from their concrete
            // types (either `&io::Error` or `&num::ParseIntError`)
            // to a trait object `&Error`. This works because both error types
            // implement `Error`.
            CliError::Io(ref err) => Some(err),
            CliError::Parse(ref err) => Some(err),
        }
    }
}
```

### From trait
定义如下，举了一个例子，Error实现了from方法，那么可以通过调用Error::from()来把传入的参数转换为Error类型。
```rust
trait From<T> {
    fn from(T) -> Self;
}
impl<T> From<T> for Error {
	fn from(T) -> Self;
}
```

那么From trait和Error handling如何搭上关系的呢？下面先看一个宏？的定义，在遇到错误的时候，会调用convert::From方法来对错误进行转换，编译器会根据返回的Error和当前Error来推断使用的From实现。
```rust
macro_rules! r#try {
    ($expr:expr $(,)?) => {
        match $expr {
            $crate::result::Result::Ok(val) => val,
            $crate::result::Result::Err(err) => {
                return $crate::result::Result::Err($crate::convert::From::from(err));
            }
        }
    };
}
```

## 一个完整的例子
```rust
use std::fs::File;
use std::io::{self, Read};
use std::num;
use std::path::Path;

// We derive `Debug` because all types should probably derive `Debug`.
// This gives us a reasonable human readable description of `CliError` values.
#[derive(Debug)]
enum CliError {
    Io(io::Error),
    Parse(num::ParseIntError),
}

// From trait的实现
impl From<io::Error> for CliError {
    fn from(err: io::Error) -> CliError {
        CliError::Io(err)
    }
}

impl From<num::ParseIntError> for CliError {
    fn from(err: num::ParseIntError) -> CliError {
        CliError::Parse(err)
    }
}

// 读取文件后转换为i32， 再乘以2
fn file_double<P: AsRef<Path>>(file_path: P) -> Result<i32, CliError> {
    let mut file = File::open(file_path)?;
    let mut contents = String::new();
    file.read_to_string(&mut contents)?;
    let n: i32 = contents.trim().parse()?;
    Ok(2 * n)
}
```

## 一些经验
1. 当实现的代码只是一个简单的例子，使用unwrap很合适，实现完整的错误处理需要过多的时间。
2. 完成一个项目的时候，觉得还是要做一下错误处理，可以使用Box<Error> (or Box<Error + Send + Sync>) ，或者使用anyhow包中的anyhow::Error。
3. 当有一定精力的时候，可以实现一下自己的错误，甚至实现对应的From trait。
4. 熟练掌握Option和Result的一些方法可以提高自己的生产力。