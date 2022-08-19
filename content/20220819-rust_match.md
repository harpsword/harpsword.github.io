+++

title = "rust match with guard"
date = 2022-08-19

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]

+++

# 问题描述

rust中的match语法支持添加 `if condition`，当变量匹配上时要求`if condition`中的`condition`满足，如下代码所示，第二个分支才会被匹配上。
```rust
fn main() {
    let pair = (2, -2);
    // TODO ^ Try different values for `pair`

    println!("Tell me about {:?}", pair);
    match pair {
        (x, y) if x == y => println!("These are twins"),
        // The ^ `if condition` part is a guard
        (x, y) if x + y == 0 => println!("Antimatter, kaboom!"),
        (x, _) if x % 2 == 1 => println!("The first one is odd"),
        _ => println!("No correlation..."),
    }
}
```

那么当match的匹配没有成功时，`if condition`是否会运行？

# 实验

实验代码如下所示，只有能匹配上的第二、第三分支guard()函数才会被执行；另外假如第二分支能匹配上，第三分支的guard函数并不会被执行。
```rust
fn guard() -> i32 {
	println!("guard");
	5
}

fn test_while_guard() {
	let a = 5;
	match a {
		1 if guard() > 3 => {

		},
		5 if guard() > 6 => {
			
		},
		5 if guard() > 3 => {
			
		},
		_ => {}
	}
}
```
总结一下，执行原则：
1. match Pattern匹配上之后，假如有guard存在，会检查guard的条件是否满足。
2. 遇到第一个满足的match arm后，之后的match arm都不会被执行。