# Macro

宏分用法分为：

1. **声明式宏( *declarative macros* )** `macro_rules!`
2. **过程宏( *procedural macros* )**

## Declarative macros

声明宏就是写一个宏的逻辑，给外部调用，比如`vec!`的逻辑如下：

```rust
#[macro_export]
macro_rules! vec {
    ( $( $x:expr ),* ) => {
        {
            let mut temp_vec = Vec::new();
            $(
                temp_vec.push($x);
            )*
            temp_vec
        }
    };
}
```

`#[macro_export]`是明确导出，`macro_rules!`是声明我在写一个宏，`vec`为宏的名称，调用时使用`vec!`以区分正常的函数调用。

## Procedural macros

过程宏则是一个函数generator，它会自动补充函数，其又分为三种宏

- Custom Derive
- Attribute-Like
- Function-Like



```rust
use proc_macro;

#[proc_macro_derive(HelloMacro)]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```





