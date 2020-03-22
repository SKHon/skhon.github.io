---
title: 整理一下正则
date: 2020-03-21 00:00:00
categories: 前端
tags: [javascript]
comments: true
copyright: true
---

## 字面量字符和元字符
`字面量字符`： 写的啥就匹配啥。
```
dog/.test('old dog') // true
```
`点字符(.)`：匹配一个除回车（\r）、换行(\n) 、行分隔符（\u2028）和段分隔符（\u2029）以外的所有字符
<!--more-->
```
/c.t/.test('cst') // true
/c.t/.test('c\nt') // false
```
`位置字符(^ 和 $)`：^ 表示字符串的开始位置，$ 表示字符串的结束位置
```
/^test/.test('test123') // true
/test$/.test('new test') // true
/^test$/.test('test') // true
/^test$/.test('test test') // false
```

`选择符（|）`:表示“或关系”（OR），即cat|dog表示匹配cat或dog。
```
// 正则表达式指定必须匹配11或22。
/11|22/.test('911') // true

/a(1|2)b/.test('a1b') // true
```

## 转义符
需要反斜杠转义的，一共有12个字符：^、.、[、$、(、)、|、*、+、?、{和\\。需要特别注意的是，如果使用RegExp方法生成正则对象，转义需要使用两个斜杠，因为字符串内部会先转义一次。

## 特殊字符
\cX     表示Ctrl-[X]，其中的X是A-Z之中任一个英文字母，用来匹配控制字符。
[\b]    匹配退格键(U+0008)，不要与\b混淆。
\n      匹配换行键。
\r      匹配回车键。
\t      匹配制表符 tab（U+0009）。
\v      匹配垂直制表符（U+000B）。
\f      匹配换页符（U+000C）。
\0      匹配null字符（U+0000）。
\xhh    匹配一个以两位十六进制数（\x00-\xFF）表示的字符。
\uhhhh  匹配一个以四位十六进制数（\u0000-\uFFFF）表示的 Unicode 字符。

## 字符类 []
字符类（class）表示有一系列字符可供选择，只要匹配其中一个就可以了。所有可供选择的字符都放在方括号内，比如[xyz] 表示x、y、z之中任选一个匹配。

```
/[abc]/.test('hello world') // false 
/[abc]/.test('apple') // true
```

### 脱字符 ^
如果方括号内的第一个字符是[^]，则表示除了字符类之中的字符，其他字符都可以匹配。比如，[^xyz]表示除了x、y、z之外都可以匹配。
```
/[^abc]/.test('hello world') // true
/[^abc]/.test('bbc') // false
```

如果方括号内没有其他字符，即只有[^]，就表示匹配一切字符，其中包括换行符。相比之下，点号作为元字符（.）是不包括换行符的。
```
var s = 'Please yes\nmake my day!';

s.match(/yes.*day/) // null
s.match(/yes[^]*day/) // [ 'yes\nmake my day']
```

> 在[]中，^只有在第一位才标示非，否则就是字面量
> 在[]中，.就是字面量

### 连字符 -
某些情况下，对于连续序列的字符，连字符（-）用来提供简写形式，表示字符的连续范围。比如，[abc]可以写成[a-c]，[0123456789]可以写成[0-9]，同理[A-Z]表示26个大写字母。
```
/a-z/.test('b') // false
/[a-z]/.test('b') // true
```
## 预定义模式

\d 匹配0-9之间的任一数字，相当于[0-9]。
\D 匹配所有0-9以外的字符，相当于[^0-9]。
\w 匹配任意的字母、数字和下划线，相当于[A-Za-z0-9_]。
\W 除所有字母、数字和下划线以外的字符，相当于[^A-Za-z0-9_]。
\s 匹配空格（包括换行符、制表符、空格符等），相等于[ \t\r\n\v\f]。
\S 匹配非空格的字符，相当于[^ \t\r\n\v\f]。
\b 匹配词的边界。
\B 匹配非词边界，即在词的内部。

```
// \s 的例子
/\s\w*/.exec('hello world') // [" world"]

// \b 的例子
/\bworld/.test('hello world') // true
/\bworld/.test('hello-world') // true
/\bworld/.test('helloworld') // false

// \B 的例子
/\Bworld/.test('hello-world') // false
/\Bworld/.test('helloworld') // true
```

## 重复类 {}
模式的精确匹配次数，使用大括号（{}）表示。{n}表示恰好重复n次，{n,}表示至少重复n次，{n,m}表示重复不少于n次，不多于m次。


### 量词符
? 问号表示某个模式出现0次或1次，等同于{0, 1}。
\* 星号表示某个模式出现0次或多次，等同于{0,}。
\+ 加号表示某个模式出现1次或多次，等同于{1,}。

```
// t 出现0次或1次
/t?est/.test('test') // true
/t?est/.test('est') // true

// t 出现1次或多次
/t+est/.test('test') // true
/t+est/.test('ttest') // true
/t+est/.test('est') // false

// t 出现0次或多次
/t*est/.test('test') // true
/t*est/.test('ttest') // true
/t*est/.test('tttest') // true
/t*est/.test('est') // true
```

## 贪婪模式
上一小节的三个量词符，默认情况下都是最大可能匹配，即匹配直到下一个字符不满足匹配规则为止。这被称为贪婪模式。
```
var s = 'aaa';
s.match(/a+/) // ["aaa"]
```
上面代码中，模式是/a+/，表示匹配1个a或多个a，那么到底会匹配几个a呢？因为默认是贪婪模式，会一直匹配到字符a不出现为止，所以匹配结果是3个a。

如果想将贪婪模式改为非贪婪模式，可以在量词符后面加一个问号。
```
var s = 'aaa';
s.match(/a+?/) // ["a"]
```
> *?：表示某个模式出现0次或多次，匹配时采用非贪婪模式。
> +?：表示某个模式出现1次或多次，匹配时采用非贪婪模式。

## 修饰符
### g修饰符
```
var regex = /b/;
var str = 'abba';

regex.test(str); // true
regex.test(str); // true
regex.test(str); // true
```
上面代码中，正则模式不含g修饰符，每次都是从字符串头部开始匹配。所以，连续做了三次匹配，都返回true。

```
var regex = /b/g;
var str = 'abba';

regex.test(str); // true
regex.test(str); // true
regex.test(str); // false
```
上面代码中，正则模式含有g修饰符，每次都是从上一次匹配成功处，开始向后匹配。因为字符串abba只有两个b，所以前两次匹配结果为true，第三次匹配结果为false。

### i 修饰符
默认情况下，正则对象区分字母的大小写，加上i修饰符以后表示忽略大小写（ignorecase）。
```
/abc/.test('ABC') // false
/abc/i.test('ABC') // true
```

### m 修饰符
m修饰符表示多行模式（multiline），会修改^和$的行为。默认情况下（即不加m修饰符时），^和$匹配字符串的开始处和结尾处，加上m修饰符以后，^和$还会匹配行首和行尾，即^和$会识别换行符（\n）。
```
/world$/.test('hello world\n') // false
/world$/m.test('hello world\n') // true
```

## 组匹配 ()
括号有两个功能，分别是分组和引用
### 分组
量词控制之前元素的出现次数，而这个元素可能是一个字符，也可能是一个字符组，也可以是一个表达式

如果把一个表达式用括号包围起来，这个元素就是括号里的表达式，被称为子表达式

如果希望字符串'ab'重复出现2次，应该写为(ab){2}，而如果写为ab{2}，则{2}只限定b
```
console.log(/(ab){2}/.test('abab')); //true
console.log(/(ab){2}/.test('abb')); //false
console.log(/ab{2}/.test('abab')); //false
console.log(/ab{2}/.test('abb')); //true
```

### 捕获
括号不仅可以对元素进行分组，还会保存每个分组匹配的文本，等到匹配完成后，引用捕获的内容。因为捕获了文本，这种功能叫捕获分组
比如，要匹配诸如2016-06-23这样的日期字符串
```
/(\d{4})-(\d{2})-(\d{2})/

console.log(/(\d{4})-(\d{2})-(\d{2})/.test('2016-06-23'));//true
console.log(RegExp.$1);//'2016'
console.log(RegExp.$2);//'06'
console.log(RegExp.$3);//'23'
console.log(RegExp.$4);//''
```

### 反向引用
反向引用允许在正则表达式内部引用之前捕获分组匹配的文本，形式是\num，num表示所引用分组的编号
```
//重复字母
/([a-z])\1/
console.log(/([a-z])\1/.test('aa'));//true
console.log(/([a-z])\1/.test('ab'));//false
```

### 非捕获
除了捕获分组，正则表达式还提供了非捕获分组(non-capturing group)，以(?:)的形式表示，它只用于限定作用范围，而不捕获任何文本
比如，要匹配abcabc这个字符，一般地，可以写为(abc){2}，但由于并不需要捕获文本，只是限定了量词的作用范围，所以应该写为(?:abc){2}
```
console.log(/(abc){2}/.test('abcabc'));//true
console.log(/(?:abc){2}/.test('abcabc'));//true
```
由于非捕获分组不捕获文本，对应地，也就没有捕获组编号
```
console.log(/(abc){2}/.test('abcabc'));//true
console.log(RegExp.$1);//'abc'
console.log(/(?:abc){2}/.test('abcabc'));//true
console.log(RegExp.$1);//''
```
非捕获分组也不可以使用反向引用
```
/(?:123)\1/.test('123123');//false
/(123)\1/.test('123123');//true
```
捕获分组和非捕获分组可以在一个正则表达式中同时出现
```
console.log(/(\d)(\d)(?:\d)(\d)(\d)/.exec('12345'));//["12345", "1", "2", "4", "5"]
```
### 先行断言
x(?=y)称为先行断言（Positive look-ahead），x只有在y前面才匹配，y不会被计入返回结果。比如，要匹配后面跟着百分号的数字，可以写成/\d+(?=%)/。“先行断言”中，括号里的部分是不会返回的。
```
var m = 'abc'.match(/b(?=c)/);
m // ["b"]
```

### 先行否定断言
x(?!y)称为先行否定断言（Negative look-ahead），x只有不在y前面才匹配，y不会被计入返回结果。比如，要匹配后面跟的不是百分号的数字，就要写成/\d+(?!%)/。

```
/\d+(?!\.)/.exec('3.14')
// ["14"]
```

