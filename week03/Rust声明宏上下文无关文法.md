# Rust声明宏与条件编译上下文无关文法

**Rust声明宏：**

```html
MacroRulesDefinition :
   macro_rules ! IDENTIFIER MacroRulesDef

MacroRulesDef :
      ( MacroRules ) ;
   | [ MacroRules ] ;
   | { MacroRules }

MacroRules :
   MacroRule ( ; MacroRule )* ;?

MacroRule :
   MacroMatcher => MacroTranscriber

MacroMatcher :
      ( MacroMatch* )
   | [ MacroMatch* ]
   | { MacroMatch* }

MacroMatch :
      Token排除 $ 和 定界符
   | MacroMatcher
   | $ IDENTIFIER : MacroFragSpec
   | $ ( MacroMatch+ ) MacroRepSep? MacroRepOp

MacroFragSpec :
      block | expr | ident | item | lifetime | literal
   | meta | pat | path | stmt | tt | ty | vis

MacroRepSep :
   Token排除 定界符 和 重复操作符

MacroRepOp :
   * | + | ?

MacroTranscriber :
   DelimTokenTree
```



**条件编译：**

```html
ConfigurationPredicate :
      ConfigurationOption
   | ConfigurationAll
   | ConfigurationAny
   | ConfigurationNot

ConfigurationOption :
   IDENTIFIER (= (STRING_LITERAL | RAW_STRING_LITERAL))?

ConfigurationAll
   all ( ConfigurationPredicateList? )

ConfigurationAny
   any ( ConfigurationPredicateList? )

ConfigurationNot
   not ( ConfigurationPredicate )

ConfigurationPredicateList
   ConfigurationPredicate (, ConfigurationPredicate)* ,?
```

每种形式的编译条件都有一个计算结果为真或假的*配置谓词(configuration predicate)*。谓词是以下内容之一：

- 一个配置选项。如果设置了该选项，则为真，如果未设置则为假。
- `all()` 这样的配置谓词列表，列表内的配置谓词以逗号分隔。如果至少有一个谓词为假，则为假。如果没有谓词，则为真。
- `any()` 这样的配置谓词列表，列表内的配置谓词以逗号分隔。如果至少有一个谓词为真，则为真。如果没有谓词，则为假。
- 带一个配置谓词的 `not()` 模式 。如果此谓词为假，整个配置它为真；如果此谓词为真，整个配置为假。

*配置选项*可以是名称，也可以是键值对，它们可以设置，也可以不设置。名称以单个标识符形式写入，例如 `unix`。键值对被写为标识符后跟 `=`，然后再跟一个字符串。例如，`target_arch=“x86_64”` 就是一个配置选项。键在键值对配置选项列表中不是唯一的。例如，`feature = "std"` and `feature = "serde"` 可以同时设置。