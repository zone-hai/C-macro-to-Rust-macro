# C宏与Rust宏对应关系

## C宏分类

```c
#define identifier replacement-list(optional)
#define identifier( parameters ) replacement-list
#define identifier( parameters, ... ) replacement-list 
#define identifier( ... ) replacement-list
```



## Rust宏分类

声明式宏（Declarative macros）

过程宏（Procedural macros）

- 派生宏（Derive macro）
- 类属性式宏（Attribute-like macros）
- 类函数式宏（Function-like macros）



## 从功能上大致划分对应关系

**1.object-like macro**

**#define identifier replacement-list(optional)**

对应：声明式宏（Declarative macros）

```rust
macro_rules! $name {
    $pattern0 => ($expansion);		//对应c中define identifier 没有replacement-list的情况
    $pattern1 => ($expansion);		//对应c中identifier单独对应一个replacement-list的情况 
}
```



**2.function-like macro**   

**#define identifier( parameters ) replacement-list**

对应：类函数式宏（Function-like macros）

类函数式宏和声明式宏有时极为相似，声明式宏有时也可以对应该类C宏语法，但考虑了第三种情况（C语言中define后面带##的情况），故这里用类函数式宏（Function-like macros）来对应这类C语法。

```c
// demo mutliply(2 + 3, 4 + 5)
#define multiply(x, y) x * y // 错误，宏展开: 2 + 3 * 4 + 5，结果19
#define multiply(x, y) ((x) * (y)) // 正确，红展开: ((2 + 3) * (4 + 5))，结果45
```

对应：

```rust
macro_rules! multiply {
    ($x:expr, $y:expr) => {
        $x * $y
    };
}

fn main() {
    let a = multiply!(2 + 3, 4 + 5);
}
```



**3.function-like macro（带有##的）**

**#define identifier( parameters ) replacement-list  （带有##的）**

对应：类函数式宏（Function-like macros）

```c
#include <stdio.h>
 
//make function factory and use it
#define FUNCTION(name, a) int fun_##name(int x) { return (a)*x;}
 
FUNCTION(quadruple, 4)
FUNCTION(double, 2)
 
#undef FUNCTION
#define FUNCTION 34
#define OUTPUT(a) puts( #a )
 
int main(void)
{
    printf("quadruple(13): %d\n", fun_quadruple(13) );
    printf("double(21): %d\n", fun_double(21) );
    printf("%d\n", FUNCTION);
    OUTPUT(million);               //note the lack of quotes
}

/*
quadruple(13): 52
double(21): 42
34
million
*/
```

对应：

```rust
// 函数式宏
#[proc_macro]
pub fn make_hello(item: TokenStream) -> TokenStream {
    let name = item.to_string();
    let hell = "Hello ".to_string() + name.as_ref();
    let fn_name =
        "fn hello_".to_string() + name.as_ref() + "(){ println!(\"" + hell.as_ref() + "\"); }";
    fn_name.parse().unwrap()
}
```

`TokenStream` 相当编译过程中的语法树的流。

make_hello 使用`#[proc_macro]` ，定义自动生成一个传入参数函数。

```rust
extern crate custom_derive;
use custom_derive::make_hello;

make_hello!(world);
make_hello!(张三);

fn main() {
    // 使用make_hello生成
    hello_world();
    hello_张三();
}

/*
Hello world
Hello 张三
*/
```



**4.function-like macro**

**#define identifier( parameters, ... ) replacement-list** 

对应：类属性式宏（Attribute-like macros）

这里考虑到C中这类语法，至少identifier中会有一个参数，而类属性式宏（Attribute-like macros）有两个参数，我们考虑第一个参数用于处理这类C语法中的那第一个参数，而类属性式宏（Attribute-like macros）中的第二个参数处理 identifier( parameters, ... ) 中“..."可变参的情况，将这些参数用结构体包装放于第二个参数的item:TokenStream进行处理。

```c
#define G(X, ...) f(0, X __VA_OPT__(,) __VA_ARGS__)
G(a, b, c) // replaced by f(0, a, b, c)
G(a, )     // replaced by f(0, a)
G(a)       // replaced by f(0, a)
 
#define SDEF(sname, ...) S sname __VA_OPT__(= { __VA_ARGS__ })
SDEF(foo);       // replaced by S foo;
SDEF(bar, 1, 2); // replaced by S bar = { 1, 2 };
```

对应：

```rust
// 属性宏 （两个参数）
#[proc_macro_attribute]
pub fn log_attr(attr:TokenStream, item:TokenStream)->TokenStream{
    println!("Attr:{}", attr.to_string());
    println!("Item:{}", item.to_string());
    item
}
```

log_attr 使用`#[proc_macro_attribute]` ，编译期间会打印结构类型和参数，后面可用修改替换原属性定义。

```rust
extern crate custom_derive;
use custom_derive::log_attr;

#[log_attr(struct, "world")]
struct Hello{
    pub name: String,
}

#[log_attr(func, "test")]
fn invoked(){}

/*
Attr:struct, "world"                                  
Item:struct Hello { pub name : String, }
Attr:func, "test"
Item:fn invoked() { }
*/
```



**5.function-like macro**

**#define identifier( ... ) replacement-list**

对应：派生宏（Derive macro）

C中这种identifier( ... )参数个数不确定的情况用派生宏（Derive macro）进行处理，对其中input: TokenStream提取相关信息进行处理。

```c
#define F(...) f(0 __VA_OPT__(,) __VA_ARGS__)
F(a, b, c) // replaced by f(0, a, b, c)
F()        // replaced by f(0)
```

对应：

```rust
// 派生宏
#[proc_macro_derive(Hello)]
pub fn hello_derive(input: TokenStream)-> TokenStream {
    println!("{:?}", input);
    TokenStream::new()  
    // 如果直接返回input，编译会报重复定义，说明派生宏用于扩展定义
    // input   
}
```

\#[derive(Hello)]  使用`#[proc_macro_derive(Hello)]`·，会打印当前TokenStream 结点流，可以和 syn 与 quto 库结合，扩展定义。

```rust
extern crate custom_derive;
use custom_derive::Hello;

#[derive(Hello)]
struct World;

/*
TokenStream [Ident { ident: "struct", span: #0 bytes(286..292) }, Ident { ident: "World", span: #0 bytes(293..298) }, Punct { ch: ';', spacing: Alone, span: #0 bytes(298..299) }]
*/
```

