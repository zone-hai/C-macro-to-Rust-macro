# Rust宏

### 一、宏的分类

​		Rust 中的宏相较C/C++更为强大。C/C++ 中的宏在预处理阶段可以展开为文本，Rust 的宏则是对语法的扩展，是在构建语法树时，才展开的宏。

​		Rust 中宏可以分为很多类，包括通过 macro_rules 定义的**声明式宏**和三种**过程式宏**。

- 声明式宏（Declarative macros）使得你能够写出类似 match 表达式的东西，来操作你所提供的 Rust 代码。它使用你提供的代码来生成用于替换宏调用的代码。
- 过程宏（Procedural macros）允许你操作给定 Rust 代码的抽象语法树（abstract syntax tree, AST）。过程宏是从一个（或者两个）`TokenStream`到另一个`TokenStream`的函数，用输出的结果来替换宏调用。

有三种类型的过程宏：

- 派生宏（Derive macros）：适用于结构、枚举和联合，并使用`#[derive(MyMacro)]`声明进行注释。它们还可以声明辅助属性，这些属性可以附加到项目的成员（例如枚举变体或结构字段）。
- 类属性式宏（Attribute-like macros）：类属性式宏能够让你创建一个自定义的属性，该属性将其自身关联一个项（item），并允许对该项进行操作。它也可以接收参数。类似于派生宏，但可以附加到更多项，例如特征定义和函数。
- 类函数式宏（Function-like macros）：类函数宏类似于声明式宏，因为它们是用宏调用运算符调用的`!`，看起来像函数调用。它们对放在括号内的代码进行操作。



### 二、声明宏的用法

在Rust中，应用最广泛的一种宏就是声明式宏，类似于模式匹配的写法，将传入的 Rust 代码与预先指定的模式进行比较，在不同模式下生成不同的代码。

使用`macro_rules!`来定义一个声明式宏。

最基础的例子是很常见的`vec!`：

```rust
let v: Vec<u32> = vec![1, 2, 3];
```

简化版的定义是（实际的版本有其他分支，而且该分支下要预先分配内存防止在push时候再动态分划）

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

`#[macro_export]`标签是用来声明：只要 use 了这个crate，就可以使用该宏。同时包含被 export 出的宏的模块，在声明时必须放在前面，否则靠前的模块里找不到这些宏。

按照官方文档的说法，`macro_rules!`目前有一些设计上的问题，日后将推出新的机制来取代他。但是他依然是一个很有效的语法扩展方法。

注意点：如果想要创建临时变量，那么必须要像上面这个例子这样，放在某个块级作用域内，以便自动清理掉，否则会认为是不安全的行为。



### 三、声明宏的限制

声明式宏有一些限制，有些是与 Rust 宏本身有关，有些则是声明式宏所特有的：

- 缺少对宏的自动完成和展开的支持
- 声明式宏调式困难
- 修改能力有限
- 更大的二进制
- 更长的编译时间（这一条对于声明式宏和过程宏都存在）



### 四、过程宏的使用

1. cargo new custom   新建一个名为custom的工程。
2. cd custom && cargo new custom-derive  在custom内新建一个名为custom-derive 用于编写过程宏。

custom  Cargo.toml

```toml
[package]
name = "custom"
version = "0.1.0"
[dependencies]
custom-derive={path="custom-derive"}
```



custom-derive  Cargo.toml

```toml
[package]
name="custom-derive"
version="0.1.0"

[lib]
proc-macro = true   # 使用过程宏

[dependencies]
# quote = "1.0.9"                 # 目前没用到，先注释了
# proc-macro2 = "1.0.27" 
# syn = {version="1.0.72", features=["full"]}
```



项目结构：

```
.
├── Cargo.toml
├── custom-derive
│   ├── Cargo.toml
│   └── src
│       └── lib.rs
└── src
│   └── main.rs

```



3. lib.rs

```rust
use proc_macro::TokenStream;

extern crate proc_macro;

// 函数式宏
#[proc_macro]
pub fn make_hello(item: TokenStream) -> TokenStream {
    let name = item.to_string();
    let hell = "Hello ".to_string() + name.as_ref();
    let fn_name =
        "fn hello_".to_string() + name.as_ref() + "(){ println!(\"" + hell.as_ref() + "\"); }";
    fn_name.parse().unwrap()
}

// 属性宏 （两个参数）
#[proc_macro_attribute]
pub fn log_attr(attr:TokenStream, item:TokenStream)->TokenStream{
    println!("Attr:{}", attr.to_string());
    println!("Item:{}", item.to_string());
    item
}


// 派生宏
#[proc_macro_derive(Hello)]
pub fn hello_derive(input: TokenStream)-> TokenStream {
    println!("{:?}", input);
    TokenStream::new()  
    // 如果直接返回input，编译会报重复定义，说明派生宏用于扩展定义
    // input   
}
```

`TokenStream` 相当编译过程中的语法树的流。

4. main.rs

```rust
extern crate custom_derive;
use custom_derive::log_attr;
use custom_derive::make_hello;
use custom_derive::Hello;

make_hello!(world);
make_hello!(张三);

#[log_attr(struct, "world")]
struct Hello{
    pub name: String,
}

#[log_attr(func, "test")]
fn invoked(){}


#[derive(Hello)]
struct World;

fn main() {
    // 使用make_hello生成
    hello_world();
    hello_张三();
}
```

make_hello 使用`#[proc_macro]` ，定义自动生成一个传入参数函数。

```shell
Hello world
Hello 张三
```



log_attr 使用`#[proc_macro_attribute]` ，编译期间会打印结构类型和参数，后面可用修改替换原属性定义。

```shell
Attr:struct, "world"                                  
Item:struct Hello { pub name : String, }
Attr:func, "test"
Item:fn invoked() { }
```



\#[derive(Hello)]  使用`#[proc_macro_derive(Hello)]`·，会打印当前TokenStream 结点流，可以和 syn 与 quto 库结合，扩展定义。

```shell
TokenStream [Ident { ident: "struct", span: #0 bytes(286..292) }, Ident { ident: "World", span: #0 bytes(293..298) }, Punct { ch: ';', spacing: Alone, span: #0 bytes(298..299) }]
```

