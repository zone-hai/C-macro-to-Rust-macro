# Rust声明宏与条件编译

在rust中，声明宏本质就是匹配规则 + 转译替换规则，也是代码模版按照匹配规则进行代码化替换。调用声明宏时，就是传入一串代码片段，在编译期由编译期根据传入代码片段来匹配宏自身定义的匹配规则，再经过转译替换规则，将宏调用代码替换为转译后的代码。



### 一、声明式宏的几类写法：

1. 常量宏，简单模式匹配替换
2. 语句宏，语句替换，返回表达式结果

3. 函数宏（Function-like macro），和过程中的类函数式宏极为相似。

这里对前两种用法做个总结，然后对Rust中条件编译宏cfg和cfg_if用法做简要总结。



### 二、声明宏一般形式：

```rust
macro_rules! $name {
    $pattern0 => ($expansion);
    $pattern1 => ($expansion);
    _ => ($expansion);
}
```

**注意：**

1. `$name`表示宏的名字，内部一般由1个或者多个模式匹配组成。匹配上规则之后就用`($expansion)`代替。 举个栗子。
2. 每个 `rule` 的格式：`($pattern) => {$expansion}`，其中括号和大括号不是特定的。可以使用 `[]`、`()`、`{}` 中的任意一种，在调用宏的时候也是，而过程宏中的类函数式宏后面只能是`()`。
3. **从最具体到最不具体的顺序编写宏规则很重要**，pattern越准确书写的pattern的位置越靠前，否则会有意想不到的错误。



### 三、常量宏形式：

```c
#define pi 3.14
#define p
int a = pi;
```

可以使用 `[]`、`()`、`{}` 中的任意一种。

```rust
macro_rules! pi {
    () => { 3.14 };
}

let a = pi();
let b = pi[];
let c = pi{};
```



### 四、语句宏形式：

```c
// demo mutliply(2 + 3, 4 + 5)
#define multiply(x, y) x * y // 错误，宏展开: 2 + 3 * 4 + 5，结果19
#define multiply(x, y) ((x) * (y)) // 正确，红展开: ((2 + 3) * (4 + 5))，结果45
```

**而Rust中不用考虑这((x) * (y))加括号的情况**，Rust中的Token trees介于tokens和 AST 之间，tokens是Token trees的叶子，而值得注意的是(...)、[...]和{...}不是叶子，而是Token trees的内部节点。比如：a + b + (c + d[0]) + e将有如下Token trees的结构：

```shell
«a» «+» «b» «+» «(   )» «+» «e»
          ╭────────┴──────────╮
           «c» «+» «d» «[   ]»
                        ╭─┴─╮
                         «0»
```

**这与表达式将产生的 AST没有关系**；根级别有七棵Token trees，而不是单个根节点。

所以之前的例子可以翻译为如下rust代码：

```rust
macro_rules! multiply {
    ($x:expr, $y:expr) => {				//匹配模式越精准的要放在前面，否则可能有意想不到的错误！
        $x * $y
    };
    ($x:expr) => {
        $x
    };
}

fn main() {
    let a = multiply!(2 + 3, 4 + 5);
    let b = multiply!(2);
}
```

如果是有多个expr，add_as(x,y,z) 或 add_as(x,y,z,m) 或 add_as(x,y,z,m,n) ......

```rust
macro_rules! add_as{
    ( $($a:expr),* )=>{
       	{
  			 // to handle the case without any arguments
   			0
  			 // block to be repeated
  			 $(+$a)*
    	}
    }
}

fn main(){
    println!("{}",add_as!(1,2,3,4)); // => println!("{}",{0+1+2+3+4})
}
```

匹配规则中包含meta变量用$标识来标示，其类型包括block、expr、ident、item、lifetime、literal、meta、pat、path、stmt、tt、ty、vis。具体用法可见[片段说明符](https://veykril.github.io/tlborm/decl-macros/minutiae/fragment-specifiers.html)。

- `item`：[*Item*](https://doc.rust-lang.org/reference/items.html)，如函数定义，常量声明 等
- `block`：[*BlockExpression*](https://doc.rust-lang.org/reference/expressions/block-expr.html)，如`{ ... }`
- `stmt`：[*Statement*](https://doc.rust-lang.org/reference/statements.html)，如 `let` 表达式（传入为 stmt 类型的参数时不需要末尾的分号，但需要分号的 item 语句除外）
- `pat`：[*Pattern*](https://doc.rust-lang.org/reference/patterns.html)，模式匹配中的模式，如 `Some(a)`
- `expr`：[*Expression*](https://doc.rust-lang.org/reference/expressions.html)，表达式，如 `Vec::new()`
- `ty`：[*Type*](https://doc.rust-lang.org/reference/types.html#type-expressions)，类型，如 `i32`
- `ident`：[IDENTIFIER_OR_KEYWORD](https://doc.rust-lang.org/reference/identifiers.html)，标识符或关键字，如 `i` 或 `self`
- `path`：[*TypePath*](https://doc.rust-lang.org/reference/paths.html#paths-in-types)，类型路径，如 `std::result::Result`
- `tt`：[*TokenTree*](https://doc.rust-lang.org/reference/macros.html#macro-invocation)，Token 树，被匹配的定界符 `(`、`[]` 或 `{}` 中的单个或多个 [token](https://doc.rust-lang.org/reference/tokens.html)
- `meta`：[*Attr*](https://doc.rust-lang.org/reference/attributes.html)，形如 `#[...]` 的属性中的内容
- `lifetime`：[LIFETIME_TOKEN](https://doc.rust-lang.org/reference/tokens.html#lifetimes-and-loop-labels)，生命周期 Token，如 `'static`
- `vis`：[*Visibility*](https://doc.rust-lang.org/reference/visibility-and-privacy.html)，可能为空的可见性限定符，如 `pub`
- `literal`：匹配 -? [*LiteralExpression*](https://doc.rust-lang.org/reference/expressions/literal-expr.html)



**匹配器可以包含重复项。这些允许匹配一系列标记。这些都有一般的形式`$ ( ... ) sep rep`。**

- `$`是字面上的美元标记。

- `( ... )`是被重复的 paren 分组匹配器。

- **`sep`是一个*可选的*分隔符。**它可能不是定界符或重复运算符之一。常见的例子是`,`和`;`。

- `rep`是*必需的*重复运算符。目前，这可以是：

  - `?`: 表示最多重复一次
  - `*`：表示零次或多次重复
  - `+`: 表示一次或多次重复

  由于`?`最多代表一次出现，因此不能与分隔符一起使用。

重复可以包含任何其他有效的匹配器，包括token trees、元变量和其他允许任意嵌套的重复。

**注意：这里`sep`可以是其他的形式，比如下面的; and和; or**

```rust
/// test宏定义了两组匹配和转译替换规则
    macro_rules! test_macro {
        ($left:expr; and $right:expr) => {
            println!("{:?} and {:?} is {:?}",
                     // $left变量的内容对应匹配上的语法片段的内容
                     stringify!($left),
                     // $right变量的内容对应匹配上的语法片段的内容
                     stringify!($right),
                     $left && $right)
        };
        ($left:expr; or $right:expr) => {
            println!("{:?} or {:?} is {:?}",
                     stringify!($left),
                     stringify!($right),
                     $left || $right)
        };
	}

/// 传入的字面上的代码片段，解析后生成的语法片段，
///  - 在解析过程中进行简易分词和解析后生成一个语法片段(包含解析出来的不同类型及其对应的值)
///  - 与声明宏中定义的匹配规则包含的字面量token和meta变量类型等，按照从左到右一对一的方式进行匹配(匹配都是进行深度匹配的，一旦当前规则匹配过程出现问题，则不会再进行后续的规则匹配)
///  - 一旦提供的语法片段和某个声明宏定义的规则匹配了，那么对应类型的值绑定到meta变量中，即用$标示来代替;
///    再匹配后，进入转译替换阶段，直接读取对应的转译替换规则的内容，将其meta变量的内容用上一阶段绑定过来的值替换，完成处理后输出即可;
/// 正好能匹配上第一个匹配规则；
/// - 第一个匹配规则为
///  一个表达式类型语法片段和; and 和另一个表达式类型语法片段
///  其中;和and需要字面上一对一匹配；
test_macro!(1i32 + 1 == 2i32; and 2i32 * 2 == 4i32);

/// 下面传入的字面上的代码片段，解析后生成的语法片段，
/// 正好能匹配上第二个匹配规则；
/// - 第二个匹配规则为：
/// 一个表达式类型语法片段和; or 和另一个表达式类型语法片段
/// 其中;和or需要字面上一对一匹配；
test_macro!(true; or false);
```



### 五、[Hygiene](https://veykril.github.io/tlborm/decl-macros/minutiae/hygiene.html#hygiene)

Hygiene是Rust宏中的一个重要的特性，写声明宏时注意以下问题。

```rust
macro_rules! using_x {
    ( ¥action:expr ) => {
        {
            let x = 1;			// let macro_x = 1;
            ¥action				// 这里展开后为x + 1 ，实际上这里是outer_x + 1 不同的x
        }
    }
}

fn main() {
    let two = using_x!(x + 1);
}

// 报错： using_x!(x + 1);
//      		 ^ not found in this scope

// using_X换成以下写法则可通过
macro_rules! using_x {
    ( ¥id:ident, ¥action:expr ) => {
        {
            let ¥id = 1;
            ¥action
        }
    }
}

fn main() {
    let two = using_x!(x, x + 1);
}
```



### 六、条件编译

条件编译可能通过两种不同的操作符实现：

- `cfg` 属性：在属性位置中使用 `#[cfg(...)]`

- `cfg!` 宏：在布尔表达式中使用 `cfg!(...)`

- `cfg_if!`宏：这个 crate 提供的宏`cfg_if`类似于`if/elif`C 预处理器宏，允许定义级联`#[cfg]`案例，发出最先匹配的实现。这使您可以方便地提供一长串`#[cfg]`代码块，而不必多次重写每个子句。

  

 `#[cfg(...)]`是属性宏，**这里考虑用cfg！宏和cfg_if！宏完成条件编译**。下面是前两种条件编译宏的用法：

```rust
// 这个函数仅当目标系统是 Linux 的时候才会编译
#[cfg(target_os = "linux")]
fn are_you_on_linux() {
    println!("You are running linux!")
}

// 而这个函数仅当目标系统 **不是** Linux 时才会编译
#[cfg(not(target_os = "linux"))]
fn are_you_on_linux() {
    println!("You are *not* running linux!")
}

fn main() {
    are_you_on_linux();
    
    println!("Are you sure?");
    if cfg!(target_os = "linux") {
        println!("Yes. It's definitely linux!");
    } else {
        println!("Yes. It's definitely *not* linux!");
    }
}

/* 输出
You are running linux!
Are you sure?
Yes. It's definitely linux!
*/
```



#### 6.1.**cfg!格式**：

```rust
macro_rules! cfg {
    ($($cfg:tt)*) => { ... };			
}
```

cfg！宏在编译时评估配置标志的布尔组合，赋予cfg！宏与属性#[cfg(...)]相同的功能。

内置的 `cfg`宏接受单个配置谓词，当谓词为真时计算为 `true` 字面量，当谓词为假时计算为 `false` 字面量。

```rust
#![allow(unused)]
fn main() {
    let machine_kind = if cfg!(unix) {
      "unix"
    } else if cfg!(windows) {
      "windows"
    } else {
      "unknown"
    };

    println!("I'm running on a {} machine!", machine_kind);
}
```



#### 6.2.**cfg_if！格式**：

```rust
cfg_if::cfg_if! {
    if #[cfg(unix)] {
        fn foo() { /* unix specific functionality */ }
    } else if #[cfg(target_pointer_width = "32")] {
        fn foo() { /* non-unix, 32-bit functionality */ }
    } else {
        fn foo() { /* fallback implementation */ }
    }
}

fn main() {
    foo();
}
```

将扩展为：

```rust
#[cfg(unix)]
fn foo() { /* unix specific functionality */ }
#[cfg(all(target_pointer_width = "32", not(unix)))]
fn foo() { /* non-unix, 32-bit functionality */ }
#[cfg(not(any(unix, target_pointer_width = "32")))]
fn foo() { /* fallback implementation */ }        
```

cfg_if更多信息见[cfg_if](https://docs.rs/cfg-if/latest/cfg_if/macro.cfg_if.html)。



#### 6.3设置配置选项

设置哪些配置选项是在 crate 编译期时就静态确定的。一些选项属于*编译器设置集(compiler-set)*，这部分选项是编译器根据相关编译数据设置的。其他选项属于*任意设置集(arbitrarily-set)*，这部分设置必须从代码之外传参给编译器来自主设置。无法在正在编译的 crate 的源代码中设置编译配置选项。**对于 `rustc`，任意配置集的配置选项要使用命令行参数 [`--cfg`](https://doc.rust-lang.org/rustc/command-line-arguments.html#--cfg-configure-the-compilation-environment) 来设置。**

**编译器设置集(compiler-set)**： [`编译器设置集`](https://rustwiki.org/zh-CN/reference/conditional-compilation.html)

条件编译更多用法详细见：[条件编译](https://rustwiki.org/zh-CN/reference/conditional-compilation.html)
