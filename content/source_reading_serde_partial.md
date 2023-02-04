+++

title = "Source Reading: serde-partial"
date = 2023-02-04

[taxonomies]
categories = ["Rust"]
tags = ["Rust"]
+++

# 目的
使用serde的时候很方便的只对部分字段进行serialize。可行思路：
1. 利用 serde(skip_serializing_if = "Option::is_none")的 field注解
2. 自定义实现Serializer接口
以上两种方法都需要一定的烦人代码。

# 例子
使用serde-partial库之后
```rust
#[derive(Serialize, SerializePartial)]
#[serde(rename_all = "camelCase")]
struct User {
    full_name: &'static str,
    age: u8,
    #[serde(rename = "contact")]
    email: &'static str,
}

const USER: User = User {
    full_name: "John Doe",
    age: 42,
    email: "john.doe@example.com",
};

// serialize only the `full_name` field
let filtered = USER.with_fields(|u| [u.full_name]);
let json = serde_json::to_value(&filtered).unwrap();
assert_eq!(
    json,
    serde_json::json!({
        "fullName": USER.full_name,
    })
);
```


# 思路
采用了自定义实现Serializer的思路，将重复代码采用过程宏的方式进行省略，如下所示：
```rust
#[derive(Serialize, SerializePartial)]
#[serde(rename_all = "camelCase")]
struct User {
    full_name: &'static str,
    age: u8,
    #[serde(rename = "contact")]
    email: &'static str,
}
```
其中的SerializePartial expand之后生成了如下代码，Partial结构实现了Serializer方法。
```rust
// Recursive expansion of SerializePartial! macro
// ===============================================

#[doc(hidden)]
#[allow(non_upper_case_globals, non_camel_case_types)]
const _: () = {
    #[derive(Debug, Clone, Copy)]
    struct UserFields {
        pub full_name: ::serde_partial::Field<'static, User>,
        pub age: ::serde_partial::Field<'static, User>,
        pub email: ::serde_partial::Field<'static, User>,
    }
    impl UserFields {
        pub const FIELDS: Self = Self {
            full_name: ::serde_partial::Field::new("fullName"),
            age: ::serde_partial::Field::new("age"),
            email: ::serde_partial::Field::new("contact"),
        };
    }
    impl ::core::iter::IntoIterator for UserFields {
        type Item = ::serde_partial::Field<'static, User>;
        type IntoIter = ::core::array::IntoIter<Self::Item, 3usize>;
        fn into_iter(self) -> Self::IntoIter {
            #[allow(deprecated)]
            ::core::array::IntoIter::new([self.full_name, self.age, self.email])
        }
    }
    #[derive(Debug, Default, Clone, Copy, PartialEq, Eq, Hash)]
    struct UserFilter {
        full_name: bool,
        age: bool,
        email: bool,
    }
    impl ::serde_partial::SerializeFilter<User> for UserFilter {
        fn skip(&self, field: ::serde_partial::Field<'_, User>) -> bool {
            match field.name() {
                "fullName" => !self.full_name,
                "age" => !self.age,
                "contact" => !self.email,
                _ => panic!("unknown field"),
            }
        }
        fn filtered_len(&self, _len: Option<usize>) -> Option<usize> {
            let mut len = 0;
            if self.full_name {
                len += 1;
            }
            if self.age {
                len += 1;
            }
            if self.email {
                len += 1;
            }
            Some(len)
        }
    }
    impl<'a> ::serde_partial::SerializePartial<'a> for User {
        type Fields = UserFields;
        type Filter = UserFilter;
        fn with_fields<F, I>(&'a self, select: F) -> ::serde_partial::Partial<'a, Self>
        where
            F: ::core::ops::FnOnce(Self::Fields) -> I,
            I: ::core::iter::IntoIterator<Item = ::serde_partial::Field<'a, Self>>,
        {
            let fields = Self::Fields::FIELDS;
            let mut filter = <Self::Filter as ::core::default::Default>::default();
            for filtered in select(fields) {
                match filtered.name() {
                    "fullName" => filter.full_name = true,
                    "age" => filter.age = true,
                    "contact" => filter.email = true,
                    _ => panic!("unknown field"),
                }
            }
            ::serde_partial::Partial {
                value: self,
                filter,
            }
        }
    }
};
```