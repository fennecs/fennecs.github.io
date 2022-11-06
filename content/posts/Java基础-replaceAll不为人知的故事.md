---
title: '[正则表达式]replaceAll不为人知的故事'
author: 土川
tags:
  - 正则
categories:
  - Java基础
slug: 2850894714
date: 2018-05-09 15:37:00
---
> String.replaceAll和String.replace的区别是，String.replaceAll使用的匹配模式是正则表达式。但是第二参数`replacement`不是简单的字符串，里面有点特殊的东西。

<!--more-->
# `\`的问题
已知`String str = "a\\bc"`,请问利用`String.replaceAll`把`str`输出为`a\\bc`到控制台怎么做？  
答案是：`System.out.println(str.replaceAll("\\\\", "\\\\\\\\"))`  
我们知道一个反斜杠用java正则来表示需要这么写`"\\\\"`，所以我一开始这么写`str.replaceAll("\\\\", "\\\\"`,结果输出的却是：

	a\bc
少了一杠，于是我不明白，`replacement`用`"\\\\"`表示两个斜杠有错？

# `$`与`\`
在`String.replaceAll`的注释是这样的，
> Note that backslashes ({@code \}) and dollar signs ({@code $}) in the replacement string may cause the results to be different than if it were being treated as a literal replacement string; 

接下来他让我`详情请看Matcher类注释`，而在`java.util.regex.Matcher#replaceAll`里是这么注释的
> Dollar signs may be treated as references to captured subsequences as described above, and backslashes are used to escape literal characters in the replacement string.

`$`在`replacement`里可以用来表达`pattern`里的子序列，比如[你真的会用java replaceAll函数吗？](http://www.cnblogs.com/iyangyuan/p/4809582.html)里举例的

	System.out.println("abac".replaceAll("a(\\w)", "$1$1")); //bbcc
	System.out.println("abac".replaceFirst("a(\\w)", "$1$1")); //bbac

所以在`replacement`里，我们要表达`$`怎么办？用`\`来转义，于是代码里就这么写`"\\$"`  
接着在`replacement`里，我们要表达特殊的`\`怎么办，需要用`\`来转义`\`，于是代码里就这么写`\\\\`。

而在`replacement`里，其他的转义该怎么写还是怎么写，比如`"System.out.println("abcd".replaceAll("a", "\t"));"`,"a"就被替换成制表符了

> 按我的理解是存在两种转义，和正则处理一样，当你写`"\\"`的时候，Java第一次转义后在内存里就是`\`，`replacement`检测到反斜杠，会将后面的字符表示为纯文本，所以写`System.out.println(s.replaceAll("a", "\\"));`是会报异常的，因为`"\\"`存在第二次转义，而第二次转义的时候后面没跟字符串，所以报异常。idea在写正则的时候，`"\\"`是会提示错误的，但是`String.replaceAll`里`replacement`写`"\\"`只有在运行时才检测得到。