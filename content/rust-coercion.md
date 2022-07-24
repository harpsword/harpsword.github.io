+++

title = "Rust 转换"
date = 2022-07-24

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++

# Rust 中的转换

涉及的内容有以下四类trait
1. Deref/DerefMut
2. AsRef/AsMut
3. Borrow/BorrowMut
4. From

还有自动转换与主动转换。

# 转换Trait

## Deref and DerefMut
trait定义如下所示，`?Sized`基本表示了所有的类型。deref函数只要求共享引用，生成的结果也是一个共享引用，并不会消耗所有权。
```rust
pub trait Deref {
    type Target: ?Sized;

    fn deref(&self) -> &Self::Target;
}
```

DerefMut的定义如下所示，实现DerefMut要求先实现Deref trait，针对的Target和Deref trait中的一样；只是多了一个方法deref_mut，入参和出参都是独占引用。
```rust
pub trait DerefMut: Deref {
    fn deref_mut(&mut self) -> &mut Self::Target;
}
```
从triat定义的位置来看，这两个trait都在`std::ops`下，ops包的定位是可重载的操作符，这两个trait在语法中有一定用处，具体内容会在自动转换一节来介绍。

这是ops包的doc引用。
> Overloadable operators.
>
> Implementing these traits allows you to overload certain operators.

## AsRef and AsMut
AsRef 定义如下所示，as_ref函数的目的是把`&self`转换为`&T`，**定位是一个廉价的共享引用-共享引用转换函数**。这个trait的定义和前面的Deref不同，采用了泛型T来表示转换的目标类型；Deref采用关联类型来表示目标类型，对于一个类型，只能定义一个关联类型，而泛型可以定义多个。
```rust
// std::convert::AsRef
pub trait AsRef<T> 
where    
	T: ?Sized, 
{
    fn as_ref(&self) -> &T;
}
```

> ⚠️ inner type是reference或者mutable reference都行。对于这种情况，AsRef会进行自动解引用。

AsMut的trait定义，和AsRef很接近，**定位是一个廉价的独占引用-独占引用转换函数。**
```rust
pub trait AsMut<T> 
where    
	T: ?Sized, 
{
    fn as_mut(&mut self) -> &mut T;
}
```

AsRef和AsMut都在包`std::convert`下，这个包的定位是一个转换trait。
> Traits for conversions between types.

## Borrow and BorrowMut
trait定义如下所示，这个trait用于实现借用数据。在某些情况下，Owner和Borrower的数据是不同的。比如String类型，对于Owner，这是一个可变的字符串，肯定是字符串之外的数据字段来实现可变；但是对于Borrower，只需要借用不可变字符串数据即可(&str)。
另外Borrow要求，**借用后的类型和原类型满足`Eq`,`Ord`,`Hash` trait是等价的。**

Borrow trait针对的是共享借用。BorrowMut trait针对的是独占借用。
```rust
// std::borrow::Borrow
pub trait Borrow<Borrowed> 
where    
	Borrowed: ?Sized, 
{
    fn borrow(&self) -> &Borrowed;
}
```

Borrow trait在`std::borrow`包中，该包的定位：
> A module for working with borrowed data.

## From and Into
trait定义如下所示，可以看到这两个trait都会消耗入参的所有权，实现value owner-> value owner的转换。From和Into互为相反方法。
```rust
pub trait From<T> {
    fn from(T) -> Self;
}
pub trait Into<T> {
    fn into(self) -> T;
}
```
对类型B实现`From<A>`之后，标准库中会提供一个默认实现`Into<B>` for A。所以一般只需要实现From trait即可。
在trait bound中一般采用`Into<T>`来限定类型。


# 自动强制转换

## 引用降级强转
引用降级强转是一种非常常见的强转操作，它可以将`&mut T`强转为`&T`。显然，这种强转总是安全的，因为共享引用会受到更多的限制。


下面是一个例子，在`let z`那一行y传进去后变成了`&T`，另外还有 print_num方法。
```rust
struct RefHolder<'a> {
    x: &'a i64,
}

impl<'a> RefHolder<'a> {
    fn new(x: &'a i64) -> RefHolder<'a> {
        RefHolder { x }
    }
}

fn print_num(y: &i64) {
    println!("y: {}", y);
}

fn main() {
	// Create `x`
	let mut x = 10;

	// Make sure `y` is `&mut i64`.
	let y = &mut x;

	// Package the downgraded reference into a struct.
	let z = RefHolder::new(y);
	
	// Print `y` downgrading it to an `&i64`.
	print_num(y);

	// Use the `z` reference again.
	println!("z.x: {}", z.x);

	*y = 3;
}
```

## Deref Coercion
会利用Deref和DerefMut两个trait的实现来对引用进行转换。会在以下场景自动进行：
1. 参数传递
2. 方法调用


## Unsized Coercion
如下所示，i32的数组会自动转换为 i32的slice，从一个sized转换为一个unsized类型。

```rust
[i32; 3] -> [i32]
```


# 主动转换
From /To

## From 

Definition
```rust
pub trait From<T> {
    fn from(T) -> Self;
}
```
假如为类型U实现了 `From<T>`，则可以调用U::from(value:T)来得到类型U的值；同时，可以调用`T.into<U>()`来将T转换为U类型，类型T的`Into<U>` trait会被标准库来实现。


同时也是利用From来进行错误处理，如下代码所述，CliError为我们自己定义的错误，为这个错误实现了From trait，其中泛型参数分别为`io::Error`和`num::ParseIntError`两个；这个也意味着 `io::Error`和`num::ParseIntError`类型分别实现了`Into<CliError>` trait。在open_and_parse_file函数中，处理Result<T,E>时，利用?宏来调用对应错误的into方法，编译器推断出可以转换为`CliError`，这样就实现了简便的错误处理。

```rust
use std::fs;
use std::io;
use std::num;

enum CliError {
    IoError(io::Error),
    ParseError(num::ParseIntError),
}

impl From<io::Error> for CliError {
    fn from(error: io::Error) -> Self {
        CliError::IoError(error)
    }
}

impl From<num::ParseIntError> for CliError {
    fn from(error: num::ParseIntError) -> Self {
        CliError::ParseError(error)
    }
}

fn open_and_parse_file(file_name: &str) -> Result<i32, CliError> {
    let mut contents = fs::read_to_string(&file_name)?;
    let num: i32 = contents.trim().parse()?;
    Ok(num)
}
```


关于强制转换，可以参考：https://rustmagazine.github.io/rust_magazine_2021/chapter_7/coercion_in_rust.html


