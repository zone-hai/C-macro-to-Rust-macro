## C_Macro_grammar

```
C_MACRO ::= Replacing_Text_Macro
		  | Conditional_Inclusion
```

### Replacing text macros

```
Replacing_Text_Macro ::= '#define' Macro_Body New_Line

// 换行符"\"可以看成预处理器读文件时进行的操作, 即遇到"\"就不认为语句在本行结束,
// 这样就不用在CFG中进行考虑.
Macro_Body ::= Var_Macro [Id_Replacement_List]
			 | Func_Macro Statements

// 前两个非连续空格用作分隔符. Q: 这里Func_Name和"("中不能有空格怎么表示?
Func_Macro ::= Func_Name '(' [Parameters] ')'

Parameters ::= Para
			 | Para ',' Parameters

// ##用来把两个同类型的token连接为一个token, 其中token不需要是宏变量.
// #用来字符化宏.
Statements ::= Statement
			 | Statement Statements

Statement ::= Token
			| Token Statement

Token ::= Str_Token
		| PP_Token

Str_Token ::= '#' Para
			| Whatever_in_double_quotation_marks
			| Str_Token "##" Str_Token

PP_Token ::= Para
		   | Other_words
		   | PP_Token '##' PP_Token
```

### Conditional inclusion

```
Conditional_Inclusion ::= If_Section
						| Ifdef_Section

If_Section ::= If_Group [Elif_Group] [Else_Group] Endif_Line

If_Group ::= '#if' Conditional_Exp New_Line [Statement_Group]

Elif_Group ::= '#elif' Conditional_Exp New_Line [Statement_Group]

Else_Group ::= '#else' New_Line [Statement_Group]

Endif_Line ::= '#endif' New_Line

Ifdef_Section ::= Ifdef_Line Endif_Line

Ifdef_Line ::= '#ifdef' Identifier New_Line
			 | '#ifndef' Identifier New_Line

Identifier ::= Var_Macro
			 | Func_Name

// Expressions BEGIN
Conditional_Exp ::= Logical_Or_Exp

Logical_Or_Exp ::= Logical_And_Exp
			   | Logical_Or_Exp '||' Logical_And_Exp

Logical_And_Exp ::= Bit_Or_Exp
				  | Logical_And_Exp '&&' Bit_Or_Exp

Bit_Or_Exp ::= Bit_Xor_Exp
			 | Bit_Or_Exp '|' Bit_Xor_Exp
			 
Bit_Xor_Exp ::= Bit_And_Exp
			  | Bit_Xor_Exp '^' Bit_And_Exp
			  
Bit_And_Exp ::= Equality_Exp
			  | Bit_And_Exp '&' Equality_Exp

Equality_Exp ::= Relational_Exp
			   | Equality_Exp '==' Cmp_Exp
			   | Equality_Exp '!=' Cmp_Exp

Cmp_Exp ::= Shift_Exp
				 | Cmp_Exp '>' Shift_Exp
				 | Cmp_Exp '>=' Shift_Exp
				 | Cmp_Exp '<' Shift_Exp
				 | Cmp_Exp '<=' Shift_Exp
				 
Shift_Exp ::= Add_Exp
			| Shift_Exp '<<' Add_Exp
			| Shift_Exp '>>' Add_Exp
			
Add_Exp ::= Multi_Exp
		  | Add_Exp '+' Mul_Exp
		  | Add_Exp '-' Mul_Exp

Mul_Exp ::= Basic_Exp
			| Mul_Exp '*' Basic_Exp
			| Mul_Exp '/' Basic_Exp

Basic_Exp ::= 'defined' Identifier
			| Integer_Const
			| Character_Const
			| Variables
// Expressions END
```

