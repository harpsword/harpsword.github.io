+++

title = "Rust Conditional Compilation"
date = 2022-07-23

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++

# Conditional Compilation

条件编译，原文来自[reference link](https://doc.rust-lang.org/reference/conditional-compilation.html)，本文是原文的翻译总结。

主要包含的内容：
1. ConfigurationPredicate: 配置断言
	1. ConfigurationOption
	2. ConfigurationAll
	3. ConfigurationAny
	4. ConfigurationNot
2. `cfg`,
3. `cfg_attr`
4. `cfg!`宏


# ConfigurationPredicate

有四种类别

| name                | 特点                                                                    |
| ------------------- | ----------------------------------------------------------------------- |
| ConfigurationOption | Identifier 是否等于 具体的取值(feature="std"), 或者Name是否被设置(unix) |
| ConfigurationAll    | 有一个条件为false，则为false。无条件时取值为true                        |
| ConfigurationAny    | 有一个题哦啊见为true则为true。无条件时取值为false                       |
| ConfigurationNot    | 取反                                                                    |

需要详细说一下 ConfigurationOption。对于Name is set or unset，设置的name是在编译过程中确定的。
对于key-value类型，有一些可用的例子：

1. target_arch: CPU架构
	1. "x86"
	2. "x86_64"
	3. "mips"
	4. "arm"
	5. "aarch64"
2. target_feature: 针对的feature，不是固定的取值
3. target_os: 目标操作系统
	1. "windows"
	2. "macos"
	3. "ios"
	4. "linux"
4. target_family: 目标操作系统族
	1. "unix"
	2. "windows"
	3. "wasm"
5. unix is set if target_family = "unix" is set, windows is the same.可以认为unix和windows是target_family=的简称。
6. target_env: 具体的工作环境？（软件）
	1. "gnu"
	2. "msvc"
	3. "musl"
7. target_endian: 大小序
	1. little
	2. big
8. target_pointer_width: 指针宽度
9. target_vendor: 
	1. apple
	2. fortanix
	3. pc
	4. unknown
10. test: 表示是测试代码


# cfg

对应语法
```
CfgAttrAttribute:
	cfg (ConfigurationPredicate)
```

cfg是一种attribute，可以用在所有attribute能用的地方。用cfg attri来描述一些代码，当断言为true时，对应的代码会被包含到源代码中（删除cfg attri的内容）；为false，对应代码则会被删除。

```rust
#![allow(unused)]
fn main() {
// The function is only included in the build when compiling for macOS
#[cfg(target_os = "macos")]
fn macos_only() {
  // ...
}

// This function is only included when either foo or bar is defined
#[cfg(any(foo, bar))]
fn needs_foo_or_bar() {
  // ...
}

// This function is only included when compiling for a unixish OS with a 32-bit
// architecture
#[cfg(all(unix, target_pointer_width = "32"))]
fn on_32bit_unix() {
  // ...
}

// This function is only included when foo is not defined
#[cfg(not(foo))]
fn needs_not_foo() {
  // ...
}

// This function is only included when the panic strategy is set to unwind
#[cfg(panic = "unwind")]
fn when_unwinding() {
  // ...
}

}
```



# cfg_attr

syntax:

_CfgAttrAttribute_ :  
   `cfg_attr` `(` _ConfigurationPredicate_ `,` _CfgAttrs_? `)`

_CfgAttrs_ :  
   [_Attr_](https://doc.rust-lang.org/reference/attributes.html) (`,` [_Attr_](https://doc.rust-lang.org/reference/attributes.html))* `,`?

当断言为true时，后面的CfgAttrs就会被应用到对应的代码上。

Examples
```rust
// linux 系统 mod os 定义在 linux.rs
// windows 系统, mod os 定义在 windows.rs
#[cfg_attr(target_os = "linux", path = "linux.rs")] 
#[cfg_attr(windows, path = "windows.rs")] 
mod os;
```


# cfg 宏

`cfg!`可以用作宏，评估包含的断言(ConfigurationPredicate)，返回最终的结果true or false.Examples

```rust
let machine_kind = if cfg!(unix) {
  "unix"
} else if cfg!(windows) {
  "windows"
} else {
  "unknown"
};

println!("I'm running on a {} machine!", machine_kind);
```
