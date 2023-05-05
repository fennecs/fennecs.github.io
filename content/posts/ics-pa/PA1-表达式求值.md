---
title: "PA1 表达式求值"
date: 2023-03-21T00:42:56+08:00
draft: true
slug: 83980562
---
# 为什么printf()的输出要换行?
因为`printf`是C标准库，自带了一个buffer，printf标准输出只有遇到换行符才会换行。

# 如何生成长表达式, 同时不会使buf溢出?
假设表达式的生成方式是`rand() % 1000`，也就是一个数字最多占3个字符。

既然随机生成的buf长度就是不可用的。我们要保证buf不溢出，就要做好长度限制。

字符串的长度是`strlen(buf)`, 数组的可用长度是`LENGTH - 1`（我就是没有减1导致buf数组越界丢了`\0`找了好久）。

jyy已经把生成表达式的框架给出来了，三种case都可以生成完整的表达式，三种case所需要的代价（cost）不同，我们再加上计算`remain`的表达式。

根据`remain`讨论case

```c
#define BUF_LENGTH 65536

static inline void gen_rand_expr() {

  int remain = LENGTH - 1 - strlen(buf); // not final

  switch (choose(3)) {
    case 0:
      gen_num();
      break;
    case 1:
      gen('(');
      gen_rand_expr();
      gen(')');
      break;
    default: // case 2
      gen_rand_expr();
      gen_rand_op();
      gen_rand_expr();
      break;
  }
}
```
其实这里`remain`的计算方式

这里用一个全局变量`count`记录未出栈的字符长度，这部分长度也是buf长度的一部分

于是`switch`代码变成

```c
#define NUM_LENGTH 10

  switch (choice) {
    case 0:           // cost at least NUM_LENGTH
      gen_num();
      break;
    case 1:           // cost at least NUM_LENGTH + 2
      gen('(');       // cost 1
      count++;
      gen_rand_expr();// cost NUM_LENGTH
      gen(')');       // cost 1
      count--;
      break;
    default/* 2 */:    // cost at least NUM_LENGTH * 2 + 1
      count += NUM_LENGTH + 1;
      gen_rand_expr(); // cost NUM_LENGTH
      gen_rand_op();   // cost 1
      count -= NUM_LENGTH + 1;
      gen_rand_expr(); // cost NUM_LENGTH
      break;
  }
```
然后`remain`的计算方式应该包含`count`

```c
int remain = LENGTH - 1 - strlen(buf) - count;
```
知道了剩余空间，分析3个case需要的字符数

case0: 至少需要`NUMBER_LENGTH`个字符
case1: 至少需要`NUMBER_LENGTH + 2`个字符
case2: 至少需要`NUMBER_LENGTH * 2 + 1`个字符

于是有
```c
  int choice = 0;

  if (remain >= NUM_LENGTH * 2 + 1) {
    choice = choose(3);
  } else if (remain >= NUM_LENGTH + 2) {
    choice = choose(2);
  } else if (remain >= NUM_LENGTH) {
    choice = choose(1);
  } else {
    fprintf(stderr, "cannot reach here!\n");
    exit(-1);
  }

  switch(choice) {
    ...
  }

```
# 为什么要使用无符号类型? (建议二周目思考)
首先内存地址就是无符号的，所以结果一定是要求无符号。

其次，补码的加减乘，在有符号和无符号都不影响结果，所以可以最后把结果转为无符号数。

但是除法由于符号位参与了计算，所以CPU指令实际是区分有符号`DIV`和无符号`IDIV`的，不能简单的把有符号计算结果转为无符号数。

因此，需要全程做无符号计算。

> 生成表达式测试用例应该每个数字后面加上`u`后缀

# 除0的确切行为
除数为0在linux下会coredump，macos会返回随机数(undefined行为)，个人倾向于在求值过程中报逻辑错误。