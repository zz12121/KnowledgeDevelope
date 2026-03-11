# String 类的常用方法都有那些？⁠⁠​

## 一句话说明（白话）

String 不可变，任何修改都会新建对象。

## 它解决什么问题 / 为什么重要

安全、可共享、线程安全。

## 核心原理（一步步讲清楚）

内部 final 字节/字符数组，拼接创建新对象。

##典型使用场景

常量、日志、配置。

## 简单例子 /伪代码

s+="x" 会创建新字符串。

## 常见坑与误区

频繁拼接用 StringBuilder。

##题库要点（原始材料）
String类提供了丰富的方法，以下是一些最常用的：

|方法类别|方法签名（示例）|作用|
|---|---|---|
|**获取信息**​|`int length()`|返回字符串长度|
||`boolean isEmpty()`/ `boolean isBlank()`|判断是否为空/空白符|
||`char charAt(int index)`|获取指定索引处的字符|
|**比较判断**​|`boolean equals(Object anObject)`|比较内容是否相等|
||`int compareTo(String anotherString)`|按字典顺序比较字符串|
||`boolean startsWith(String prefix)`|判断是否以指定前缀开头|
||`boolean contains(CharSequence s)`|判断是否包含指定字符序列|
|**操作子串**​|`String substring(int beginIndex, int endIndex)`|截取子串|
||`String concat(String str)`|连接字符串（等效于`+`）|
|**查找索引**​|`int indexOf(String str)`|返回指定子串第一次出现的索引|
||`int lastIndexOf(String str)`|返回指定子串最后一次出现的索引|
|**修改替换**​|`String replace(char oldChar, char newChar)`|替换字符|
||`String toLowerCase()`/ `toUpperCase()`|转换大小写|
||`String trim()`|去除首尾空白符|
|**切割转换**​|`String[] split(String regex)`|根据正则表达式切割字符串|
||`char[] toCharArray()`|转换为字符数组|
||`static String valueOf(基本类型/对象)`|将其他类型转换为String|

##关联知识
- 

## 延伸阅读（后续补充）
- 
