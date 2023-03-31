# Rust过程宏用法

过程宏，它更像函数，接受一些代码作为参数输入，然后对他们进行加工，生成新的代码，他不是在做声明式宏那样的模式匹配。三种过程式宏都是这种思路。

过程宏分为三种：

- 派生宏（Derive macro）：用于结构体（struct）、枚举（enum）、联合（union）类型，可为其实现函数或特征（Trait）。
- 属性宏（Attribute macro）：用在结构体、字段、函数等地方，为其指定属性等功能。如标准库中的#[inline]、#[derive(...)]等都是属性宏。
- 函数式宏（Function-like macro）：用法与普通的规则宏类似，但功能更加强大，可实现任意语法树层面的转换功能。



不能在原始的crate中直接写过程式宏，需要把过程式宏放到一个单独的crate中（以后可能会消除这种约定）。定义过程式宏的方法如下：

```rust
use proc_macro;

#[some_attribute]
pub fn some_name(input: TokenStream) -> TokenStream {
}
```

需要引入`proc_macro` 这个 crate，然后标签是用来声明它是哪种过程式宏的，接着就是一个函数定义，函数接受 `TokenStream`，返回 `TokenStream`。`TokenStream` 类型就定义在 `proc_macro` 包中，表示 token 序列。除了标准库中的这个包，还可以使用`proc_macro2` 包，使用 `proc_macro2::TokenStream::from()` 和 `proc_macro::TokenStream::from()` 可以很便捷地在两个包的类型间进行转换。使用 `proc_macro2` 的好处是可以在过程宏外部使用 `proc_macro2` 的类型，相反 `proc_macro` 中的类型只可以在过程宏的上下文中使用。且 `proc_macro2` 写出的宏更容易编写测试代码。



### Derive macro 宏

实现下面的代码，使用编译器生成名为 `HelloMacro` 的 `Trait`

```rust
use hello_macro::HelloMacro;
use hello_macro_derive::HelloMacro;

#[derive(HelloMacro)]
struct Pancakes;

fn main() {
    Pancakes::hello_macro();
}
```

该 `Trait` 的定义如下，目的是打印实现该宏的类型名

```rust
pub trait HelloMacro {
    fn hello_macro();
}
```

由于过程宏不能在原 crate 中实现，我们需要如下在 `hello_crate` 的目录下新建一个 `hello_macro_derive` crate

```bash
cargo new hello_macro_derive --lib
```

在新的 crate 内，我们需要修改 `Cargo.toml` 配置文件，

```toml
[lib]
proc-macro = true

[dependencies]
syn = "1.0"
quote = "1.0"
```

在 `src/lib.rs` 中可以着手实现该宏，其中 `syn` 是用来解析 rust 代码的，而quote则可以用已有的变量生成代码的 `TokenStream`，可以认为 `quote!` 宏内的就是我们想要生成的代码

```rust
extern crate proc_macro;

use proc_macro::TokenStream;
use quote::quote;
use syn;

#[proc_macro_derive(HelloMacro)]
pub fn hello_macro_derive(input: TokenStream) -> TokenStream {
    // Construct a representation of Rust code as a syntax tree
    // that we can manipulate
    let ast = syn::parse(input).unwrap();

    // Build the trait implementation
    impl_hello_macro(&ast)
}

fn impl_hello_macro(ast: &syn::DeriveInput) -> TokenStream {
    let name = &ast.ident;
    let gen = quote! {
        impl HelloMacro for #name {
            fn hello_macro() {
                println!("Hello, Macro! My name is {}!", stringify!(#name));
            }
        }
    };
    gen.into()
}
```

另外，**Custom Derive 宏可以携带Attributes，称为 Derive macro helper attributes**，具体编写方法可以参考 [Reference](https://doc.rust-lang.org/reference/procedural-macros.html#derive-macro-helper-attributes)（Rust 中共有[四类 Attributes](https://doc.rust-lang.org/reference/attributes.html)）。关于 Derive macro helper attributes 这里有一个坑就是**在使用 `cfg_attr` 时，需要把 Attributes 放在宏之前。**



### Attribute-Like 宏

attribute-like 宏和 custom derive 宏很相似，只是标签可以自定义，更加灵活，甚至可以使用在函数上。他的使用方法如下，比如假设有一个宏为 `route` 的宏

```rust
#[route(GET, "/")] 
fn index() { ... }
```

按下面的语法定义 `route` 宏

```rust
#[proc_maco_attribute]
pub fn route(attr: TokenStream, item: TokenStream) -> TokenStream { ... }
```

其中 `attr` 参数是上面的 `Get`，`"/"` ；`item` 参数是 `fn index(){}` 。





### Function-Like 宏

这种宏看上去和 `macro_rules!` 比较类似，但是在声明式宏只能用 `match` 去做模式匹配，但是在这里可以有更复杂的解析方式，所以可以写出来

```rust
let sql = sql!(SELECT * FROM posts WHERE id=1);
```

上面这个 `sql` 宏的定义方法如下

```rust
#[proc_macro]
pub fn sql(input: TokenStream) -> TokenStream { ... }
```





## 过程宏扩展需用的一些库

[proc_macro](https://doc.rust-lang.org/proc_macro/index.html)：默认 token 流库，只能在过程宏中使用，编译器要用它，将它作为过程宏的返回值，大多数情况我们不需要，只需要在宏返回结果的时候把 `proc_macro2::TokenSteam` 的流 `into()` 到 `proc_macro::TokenSteam` 就行了。

[proc_macro2](https://crates.io/crates/proc_macro2)：我们真正在使用的过程宏库，可以在过程宏外使用。

[syn](https://crates.io/crates/syn)：过程宏左护法，可以将 `TokenStream` 解析成语法树，注意两个 `proc_macro` 和 `proc_macro` 都支持，需要看文档搞清楚库函数到底是在解析哪个库中的 `TokenStream`。

[quote](https://crates.io/crates/quote)：过程宏右护法，将语法树解析成 `TokenStream`。只要一个 `quote!{}` 就够了！`quote!{}` 宏内都是字面量，即纯纯的代码，要替换进去的变量是用的 `#` 符号标注，为了和声明宏中使用的 `$` 相区分（也就意味着用 `quote` 写过程宏的时候，可以和声明宏结合 ）。模式匹配时用到的表示重复的符号和声明宏中一样，是使用 `*`。

[darling](https://crates.io/crates/darling) 好用的标签宏解析库。
