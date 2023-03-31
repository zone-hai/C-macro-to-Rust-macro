**C-Macro**

- define

  ```c
  #define identifier
  ```

  把`identifier`加到token表中, 如果原来有对应的`replacement-list`就擦除替换列表.

- object-like macro

  ```c
  #define identifier replacement-list
  ```

  把`identifier`加到token表中, 继续扫描源文件遇`identifier`到时用`replacement-list`替换.

- function-like macro

  ```c
  #define identifier(parameters) replacement-list
  ```

  `identifier`和`(`中间不能有空格, 参数列表由与之匹配的`)`终止.(`identifier`后有无空格是预处理器区分macro是object-like还是function-like的依据)

  将源文件中的`identifier(paramenters)`用`replacement-list`替换, 替换内容由源文件中的实参指定, 替换位置由`replacement-list`中对应的形参位置指定.

- undef

  ```c
  #undef identifier
  ```

  从token表中擦除`identifier`. 对以上所有#define identifier均有效



**Syntax**

```c
// from en.cppreference.com/w/c/preprocessor/replace
#define identifier replacement-list(optional)
#define identifier( parameters ) replacement-list
#define identifier( parameters, ... ) replacement-list //  __VA_ARGS__可用...的内容替换后##连接前后来实现
#define identifier( ... ) replacement-list
#undef identifier
```



