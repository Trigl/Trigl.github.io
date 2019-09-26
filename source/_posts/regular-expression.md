---
layout:     post
title:      "Java正则表达式基础"
date:       2016-03-21
author:     "Ink Bai"
catalog:    true
header-style: "text"
tags:
    - Java 基础
---

> 正则表达式是跨语言的，并没有 Java 语言还是 C 语言的区分，这里除了讲一下正则表达式的基础，另外再讲一下正则表达式在 Java 中的使用，比如与之相关的某些类。

## 定义
正则表达式（regular expression）用于指定字符串的模式，它由一些特殊的字符语法组成，可以匹配某种特定模式的字符串。简单地说，正则表达式就是特定的语法写的一个特殊字符串，然后看我们平时的普通字符串，比如电话号码、身份证号、邮箱是否符合这个特定字符串的规则，符合就选中，不符合就淘汰。
## 语法
|语法|解释|
|-|-|
|**字符**|单个字符|
|c<br><br>\unnn，\xnn，\0n，\0nn，\0nnn<br>\t，\n，\r，\f，\a，\e<br><br>\cc|字符本身<br><br>具有给定十六进制或十进制值的码元<br><br>控制符：制表符、换行符、回车符、换页符、警告符、逃逸符<br><br>与字符c相关的控制类|
|**字符类**|字符类是一个括在括号中的可选择的字符集。例如，[Jj]代表J或者j，[A-Za-z]代表所有大小写字母，[0-9]代表所有数字。这里“-”表示以恶范围（所有Unicode值落在两个边界范围之内的字符）
|[AB...]<br><br>[^...]<br><br>[...&&...]|任何由A、B...表示的字符，如正则：[Bb[0-9]]，匹配：B（或b或数字）<br><br>字符集的补集，如正则：[^0-9]，匹配：非数字<br><br>两个字符集的交集，如正则：[a-g&&d-n]，匹配：字母d或e或f或g|
|**预定义字符类**|预定义字符类在Java中反单斜杠"\"要写成反双斜杠"\\"|
|.<br><br>\d<br><br>\D<br><br>\s<br><br>\S<br><br>\w<br><br>\W<br><br>\p{name}<br><br>\P{name}|除了行终止符之外的所有字符（在DOTALL标志被设置时，则表示所有字符）<br><br>一个数字[0-9]<br><br>一个非数字[^0-9]<br><br>一个空白字符[\t\n\r\f\x0B]<br><br>一个非空白字符<br><br>一个词语字符[a-zA-Z0-9_]<br><br>一个非词语字符<br><br>一个命名字符类（字符类的名字见下面）<br><br>一个命名字符类的补集|
|**预定义字符类的名字**|上面预定义字符类\p{name}和\P{name}大括号里面的name|
|Lower<br><br>Upper<br><br>Alpha<br><br>Digit<br><br>Alnum<br><br>XDigit<br><br>Print或Graph<br><br>Punct<br><br>ASCII<br><br>Cntrl<br><br>Blank<br><br>Space<br><br>javaLowerCase<br><br>javaUpperCase<br><br>javaWhitespace<br><br>javaMirrored<br><br>InBlock<br><br>IsScript<br><br>Category或InCategory<br><br>IsProperty|ASCII的小写字母[a-z]<br><br>ASCII的大写字母[A-Z]<br><br>ASCII的字母[a-zA-Z]<br><br>ASCII的数组[0-9]<br><br>ASCII的字母或数字[A-Za-z0-9]<br><br>十六进制数字[0-9A-Fa-f]<br><br>可打印的ASCII字符[\x21-\x7E]<br><br>ASCII的非字母和数字字符[\p{Print}&&\P{Alnum}]<br><br>所有ASCII字符[\x00-\x7F]<br><br>ASCII的控制字符[\x00-\x1F]<br><br>空格字符或制表符[\t]<br><br>Space[\t\n\r\f\0x0B]<br><br>小写字母，正如Character.isLowerCase()确定的字符<br><br>大写字母，正如Character.isUpperCase()确定的字符<br><br>空白字符，正如Character.isWhitespqce()确定的字符<br><br>镜像字符，正如Character.isMirrored()确定的字符<br><br>Block是Unicode字符块的名字，不过要剔除名字中的空格，例如Arrows或LatinlSupplement<br>Script是Unicade脚本的名称，不过要剔除名字中的空格，例如Common<br><br>Category是Unicode字符类别的名字，例如，L（字母）和Sc（货币符号）<br><br>ProPerty是以下值之一：Alphabetic、Ideographic、Letter、Lowercase、Uppercase、Titlecase、Punctuation、Control、White_Space、Digit、Hex_Digit、Noncharater_Code_Point、Assigned|
|**边界匹配符**||
|^ $|输入的开头和结尾（或者在多行模式下行的开头和结尾）|
|\b|一个词语的边界|
|\B|一个非词语的边界|
|\A|输入的开头|
|\z|输入的结尾|
|Z|除了行终止符之外的输入结尾|
|\G|前一个匹配的结尾|
|**量词**||
|X?|可选的X|
|X*|X重复0或多次|
|X+|X重复1或多次|
|X{n} X{n,} X{n,m}|X重复n次，至少n次，在n到m次之间|
|**量词后缀**||
|?|将默认（贪婪）匹配转为勉强匹配|
|+|将默认（贪婪）匹配转为占有匹配|
|**集合操作**||
|XY|任何X中的字符串，后面跟随任何Y中的字符串|
|X&#124;Y|任何X或Y中的字符串|
|**群组**||
|(X)|捕获将X作为群组匹配的字符串|
|\n|第n个群组的匹配|
|**转义**||
|\c|字符c（必须是不在字母表中的字符）|
|\Q . . . \E|逐字地引用...|
|(?...)|特殊结构——请查看Pattern类的API注释|

## Java中处理正则表达式

`java.util.regex.Pattern`：模式类，用正则表达式可以构建一个Pattern对象，用来匹配将要输入的字符串。
`java.util.regex.Matcher`：匹配类，调用模式类的matcher(str)，向其中输入要匹配的字符串，就可以获得一个Matcher对象。

#### 基本正则匹配

```java
	Pattern pattern = Pattern.compile(regexString); // 调用Pattern的compile静态方法，输入正则表达式，获得Pattern对象
	Matcher matcher = pattern.matcher(input); // 调用Pattern的mathcer方法，输入要匹配的字符串，获得Matcher对象
	if(matcher.matches()) ...... // 调用Matcher的matches方法来验证字符串是否匹配所给的正则表达式
```

**注意**：匹配器的输入可以是任何实现了CharSequence接口的类的对象，例如String、StringBuilder和CharBuffer。

在Pattern类编译这个正则模式时，可以设置一个或多个标志，例如：

```java
		Pattern pattern = Pattern.compile(patternString,
				Pattern.CASE_INSENSITIVE + Pattern.UNICODE_CASE);
```

下面是所支持的六个标志：

|标识|描述|
|---|---|
|CASE_INSENSITIVE|匹配字符时忽略字母的大小写，默认情况下，这个标志只考虑US ASCII字符|
|UNICODE_CASE|当与CASE_INSENSITIVE组合时，用Unicode字母的大小写来匹配|
|MULTILINE|^和$匹配行的开头和结尾，而不是整个输入的开头和结尾|
|UNIX_LINES|在多行模式中匹配^和$时，只有'\n'被识别成行终止符|
|DOTALL|当使用这个标志时， . 符号匹配所有符号，包括行终止符|
|CANON_EQ|考虑Unicode字符规范的等价性|

#### Matcher类的find方法
通常情况下，我们并不会用正则来匹配全部输入，而只是想找出输入中一个或多个匹配的子字符串，这时可以用Matcher类的find来查找匹配内容，如果返回true，再使用start和end方法来查找匹配的内容。

```java
	while(matcher.find()) {
		int start = matcher.start();
		int end = matcher.end();
		String match = input.substring(start, end);
		System.out.println(match);
	}
```

#### 替换字符串
使用Matcher类的replaceAll方法可以将正则表达式出现的所有地方都用替换字符来替换，replaceFirst方法将只替换模式的第一次出现，如下：

```java
	Pattern pattern = Pattern.compile("[0-9]+");
	Matcher matcher = pattern.matcher(input);
	String output1 = matcher.replaceAll("#");
	String output2 = matcher.replaceFirst("$");
```

#### 分割字符串
Pattern类有一个split方法，类似于String的split方法，可以使用正则表达式来匹配边界进行分割。例如，下面的指令可以将输入分割成标记，其中分割符是由可选的空白字符包围的标点符号。

```java
	Pattern pattern = Pattern.compile("\\s*\\p{Punct}\\s*");
	String[] tokens = pattern.split(input);
```

#### 群组
如果正则表达式包含群组，那么Matcher对象可以获得具体的某个群组和揭示群组的边界。

**获得某个群组**
可以调用Matcher对线的以下方法抽取匹配的字符串中的群组：

> String group(int groupIndex)

群组0是整个输入，而用于第一个实际群组的群组索引是1。调用groupCount方法可以获得全部群组的数量。对于嵌套群组来说，是按照前括号进行排序的。

**得到群组的边界**
Matcher对象以下的方法将产生指定群组的开始索引和结束索引+1，注意加了1所以是结束索引之后的那个索引

> int start(int groupIndex)
> int end(int endIndex)

例如，假设有下面的正则表达式：

```java
((1?[0-9]):([0-5][0-9]))[ap]m
```

和下面的输出：

```java
11:59am
```

那么，11:59am所对应索引是0-6，所包含的群组和边界如下：

|群组索引|开始|结束|字符串|
|:-:|:-:|:-:|:-:|
|0|0|7|11:59am|
|1|0|5|11:59|
|2|0|2|11|
|3|3|5|59|


**实例：用括号打印出群组边界**

```java
package com.trigl.corejava.regex;

import java.util.Scanner;
import java.util.regex.Matcher;
import java.util.regex.Pattern;
import java.util.regex.PatternSyntaxException;
/**
 * 首先提示输入一个模式，然后提示输入用于匹配的字符串，随后将打印出输入是否
 * 与模式相匹配。如果输入匹配模式，并且模式包含群组，那么这个程序将用括号打
 * 印出群组边界
 * 如：
 * (([Cc][Rr][Ee][Aa][Tt][Ee])|([Rr][Ee][Pp][Ll][Aa][Cc][Ee])|(([Cc][Rr][Ee][Aa][Tt][Ee])\s([Oo][Rr])\s([Rr][Ee][Pp][Ll][Aa][Cc][Ee])))\s([Pp][Rr][Oo][Cc][Ee][Dd][Uu][Rr][Ee])
 * 匹配3种类型开头的字符串：create procedure/replace procedure/create or replace procedure
 * @author 白鑫
 * @date 2016年3月16日 上午10:16:27
 */
public class RegexTest {
	public static void main(String[] args) throws PatternSyntaxException {
		// 构造一个新的扫描器，产生来自指定输入流的值，现在是从键盘输入
		Scanner in = new Scanner(System.in);
		System.out.println("Enter Pattern: ");
		// 输入正则表达式并产生模式类对象
		String patternString = in.nextLine();
		Pattern pattern = Pattern.compile(patternString);

		while(true) {
			System.out.println("Enter string to match: ");
			// 输入字符串并尝试匹配类对象
			String input = in.nextLine();
			if(input == null || input.equals("")) return;
			Matcher matcher = pattern.matcher(input);
			// 验证是否匹配，若匹配是否有群组，有的话在群组边界打印括号
			if(matcher.matches()) {
				System.out.println("Match");
				int g = matcher.groupCount();
				if(g > 0) {
					for (int i = 0; i < input.length(); i++) {
						// 打印所有的空群组
						for(int j = 1; j <= g; j++)
							if(i == matcher.start(j) && i == matcher.end(j))
								System.out.print("()");
						// 打印非空群组的 ( 前半括号
						for(int j = 1; j <=g; j++)
							if(i == matcher.start(j) && i != matcher.end(j))
								System.out.print('(');
						System.out.print(input.charAt(i));
						// 打印非空群组的 ) 后半括号
						for(int j = 1; j <=g; j++)
							if(i+1 != matcher.start(j) && i+1 == matcher.end(j))
								System.out.print(')');
					}
					System.out.println();
				}
			} else {
				System.out.println("No match");
			}
		}
	}
}
```

**输出结果**

```java
Enter Pattern:
((1?[0-9]):([0-5][0-9]))[ap]m
Enter string to match:
11:11pm
Match
((11):(11))pm
......
```
