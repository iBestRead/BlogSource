---
title: 使用Humanizer让.NET中的类型可读性更友好
tags: 
  - .NET
  - Humanizer

categories:
  - 随笔

date: 2020-02-06 10:49:00
---

[Humanizer](https://github.com/Humanizr/Humanizer)，是一个可以让.NET中字符串，枚举，日期，时间，数字等类型阅读起来更加友好的类库。 例如，按驼峰命名法转句子，单词单数转复数，*timespans*转换为较友好形式显示等。并且对多国语言都有支持。*Humanizer*的安装和使用都非常简单，下面会详细介绍。

<!-- more -->

## 安装

可以使用[*Nuget*方式安装*Humanizer*](https://www.nuget.org/packages/Humanizer)到项目中：

.NET CLI

```bash
dotnet add package Humanizer --version 2.7.9
```

Package Reference

```xml
<PackageReference Include="Humanizer" Version="2.7.9" />
```

> 注意：如果你只需要支持英文，安装`Humanizer.Core`包即可。如需其它语言（中文），就需要`Humanizer`包。

## 使用

### 字符串更友好

*Humanize*可以把计算机文字转换成易于阅读的句子。比如，转换类名，方法名和属性等。

``` csharp
"PascalCaseInputStringIsTurnedIntoSentence".Humanize() => "Pascal case input string is turned into sentence"

"Underscored_input_string_is_turned_into_sentence".Humanize() => "Underscored input string is turned into sentence"

"Underscored_input_String_is_turned_INTO_sentence".Humanize() => "Underscored input String is turned INTO sentence"
```

还可以使用不同参数：

```csharp
"CanReturnTitleCase".Humanize(LetterCasing.Title) => "Can Return Title Case"

"Can_return_title_Case".Humanize(LetterCasing.Title) => "Can Return Title Case"

"CanReturnLowerCase".Humanize(LetterCasing.LowerCase) => "can return lower case"

"CanHumanizeIntoUpperCase".Humanize(LetterCasing.AllCaps) => "CAN HUMANIZE INTO UPPER CASE"
```

### 转换字符串

使用`Transform`方法，可以替代上面的`LetterCasing`用法：

```csharp
"Sentence casing".Transform(To.LowerCase) => "sentence casing"
"Sentence casing".Transform(To.SentenceCase) => "Sentence casing"
"Sentence casing".Transform(To.TitleCase) => "Sentence Casing"
"Sentence casing".Transform(To.UpperCase) => "SENTENCE CASING"
```

### 截断字符串

使用`Truncate`方法方便对字符串进行截断操作：

```csharp
"Long text to truncate".Truncate(10) => "Long text…"
```

截断后默认使用`…`作为后缀（注意这个后缀只占位1），如果需要自定义后缀，使用下面的方法：

```csharp
"Long text to truncate".Truncate(10, "---") => "Long te---"
```

默认的截断策略*Truncator.FixedLength*是将输入字符串截断为指定的长度，包括后缀的长度。 另外还有两种截断器策略：

- 固定指定数量的字符。
- 固定指定数量的单词。 

```csharp
"Long text to truncate".Truncate(10, Truncator.FixedLength) => "Long text…"
"Long text to truncate".Truncate(10, "---", Truncator.FixedLength) => "Long te---"
 
"Long text to truncate".Truncate(6, Truncator.FixedNumberOfCharacters) => "Long t…"
"Long text to truncate".Truncate(6, "---", Truncator.FixedNumberOfCharacters) => "Lon---"

"Long text to truncate".Truncate(2, Truncator.FixedNumberOfWords) => "Long text…"
"Long text to truncate".Truncate(2, "---", Truncator.FixedNumberOfWords) => "Long text---"
```

### DateTime更友好

可以对*DateTime*或*DateTimeOffset*的进行友好性转换，返回倒退或前进多少时间：

```csharp
DateTime.UtcNow.AddHours(-30).Humanize() => "yesterday"
DateTime.UtcNow.AddHours(-2).Humanize() => "2 hours ago"

DateTime.UtcNow.AddHours(30).Humanize() => "tomorrow"
DateTime.UtcNow.AddHours(2).Humanize() => "2 hours from now"

DateTimeOffset.UtcNow.AddHours(1).Humanize() => "an hour from now"
  
// 中文本地化
DateTime.UtcNow.AddDays(-1).Humanize() => "昨天"
DateTime.UtcNow.AddDays(-2).Humanize() => "2 天前"
DateTime.UtcNow.AddDays(-3).Humanize() => "3 天前"
DateTime.UtcNow.AddDays(-11).Humanize() => "11 天前"
```

### TimeSpan更友好

同样也可以对*TimeSpan*进行转换：

```csharp
TimeSpan.FromMilliseconds(1).Humanize() => "1 millisecond"
TimeSpan.FromMilliseconds(2).Humanize() => "2 milliseconds"
TimeSpan.FromDays(1).Humanize() => "1 day"
TimeSpan.FromDays(16).Humanize() => "2 weeks"
```

此方法默认使用的是一个单位精度（不同单位算一节），你可以指定返回多个精度：

```csharp
TimeSpan.FromDays(1).Humanize(precision:2) => "1 day" // no difference when there is only one unit in the provided TimeSpan
TimeSpan.FromDays(16).Humanize(2) => "2 weeks, 2 days"

// the same TimeSpan value with different precision returns different results
TimeSpan.FromMilliseconds(1299630020).Humanize() => "2 weeks"
TimeSpan.FromMilliseconds(1299630020).Humanize(3) => "2 weeks, 1 day, 1 hour"
TimeSpan.FromMilliseconds(1299630020).Humanize(4) => "2 weeks, 1 day, 1 hour, 30 seconds"
TimeSpan.FromMilliseconds(1299630020).Humanize(5) => "2 weeks, 1 day, 1 hour, 30 seconds, 20 milliseconds"
  
// 中文本地化
TimeSpan.FromDays(1).Humanize() => "1 天"
TimeSpan.FromDays(2).Humanize() => "2 天"

TimeSpan.FromMilliseconds(1299630020).Humanize(1, collectionSeparator: null) => "2 周"
TimeSpan.FromMilliseconds(1299630020).Humanize(2, collectionSeparator: null) => "2 周 & 1 天"
TimeSpan.FromMilliseconds(1299630020).Humanize(3, collectionSeparator: null) => "2 周, 1 天 & 1 小时"
TimeSpan.FromMilliseconds(1299630020).Humanize(4, collectionSeparator: null) => "2 周, 1 天, 1 小时 & 30 秒"
```

### 单词复数转换

使用`Pluralize`方法可以获取到单词的复数形式：

```csharp
"Man".Pluralize() => "Men"
"string".Pluralize() => "strings"
```

当不确定单词是否为单数时，可以指定`inputIsKnownToBeSingular`参数：

```csharp
"Men".Pluralize(inputIsKnownToBeSingular: false) => "Men"
"Man".Pluralize(inputIsKnownToBeSingular: false) => "Men"
"string".Pluralize(inputIsKnownToBeSingular: false) => "strings"
```

### 单词单数转换

使用`Singularize`方法可以获取到单词的单数形式，不确定是否为复数时，可以指定`inputIsKnownToBePlural`：

```csharp
"Men".Singularize() => "Man"
"strings".Singularize() => "string"
  
"Men".Singularize(inputIsKnownToBePlural: false) => "Man"
"Man".Singularize(inputIsKnownToBePlural: false) => "Man"
"strings".Singularize(inputIsKnownToBePlural: false) => "string"
```

### 计数文字转换

很多时候，需要在单数或复数单词前面添加一个数字。 例如 “ 2 requests ”，“  3 men ”。 `ToQuantity`可以实现此功能：

```csharp
"case".ToQuantity(0) => "0 cases"
"case".ToQuantity(1) => "1 case"
"case".ToQuantity(5) => "5 cases"
"man".ToQuantity(0) => "0 men"
"man".ToQuantity(1) => "1 man"
"man".ToQuantity(2) => "2 men"
```

`ToQuantity`会根据输入的数量，对单词的单复数形式进行自动转换：

```csharp
"men".ToQuantity(2) => "2 men"
"process".ToQuantity(2) => "2 processes"
"process".ToQuantity(1) => "1 process"
"processes".ToQuantity(2) => "2 processes"
"processes".ToQuantity(1) => "1 process"
```

另外还有一个重载方法，根据不同语言文化，返回文字：

```csharp
"dollar".ToQuantity(2, "C0", new CultureInfo("en-US")) => "$2 dollars"
"dollar".ToQuantity(2, "C2", new CultureInfo("en-US")) => "$2.00 dollars"
"cases".ToQuantity(12000, "N0") => "12,000 cases"

// 中文本地化 时有问题会在元后面加个s
"元".ToQuantity(2, "C0", new CultureInfo("zh-Hans")) => "￥2 元s"
"元".ToQuantity(2, "C2", new CultureInfo("zh-Hans")) => "￥2.00 元s"
```

### 有序文本转换

使用`Ordinalize`方法，可以把数字转成类似` 1st, 2nd, 3rd, 4th `的文本：

```csharp
1.Ordinalize(new CultureInfo("en")) => "1st"
5.Ordinalize(new CultureInfo("en")) => "5th"

// 中文本地化
1.Ordinalize(new CultureInfo("zh-Hans")) => "1"
5.Ordinalize(new CultureInfo("zh-Hans")) => "5"
```

### 大驼峰转换

使用`Pascalize`可以把字符串转为大驼峰形式，并会移除下划线：

```csharp
"some_title for something".Pascalize() => "SomeTitleForSomething"
```

### 小驼峰转换

首字母会小写

```csharp
"some_title for something".Camelize() => "someTitleForSomething"
```

### 下划线分隔文本转换

```csharp
"SomeTitle".Underscore() => "some_title"
```

### 中划线（破折号）文本转换

`Dasherize`和`Hyphenate`可以把以下划线分隔的文本，转换成以中划线分隔的文本：

```csharp
"some_title".Dasherize() => "some-title"
"some_title".Hyphenate() => "some-title"
```

### 中划线全小写文本转换

`Kebaberize`方法可以把以驼峰命名的文本，转换成以中划线分隔的全小写文本：

```csharp
"SomeText".Kebaberize() => "some-text"
```

### 数字转换

```csharp
1.ToWords() => "one"
10.ToWords() => "ten"
11.ToWords() => "eleven"
122.ToWords() => "one hundred and twenty-two"
3501.ToWords() => "three thousand five hundred and one"
  
// 中文本地化
1.ToWords(GrammaticalGender.Masculine) => "一"
10.ToWords(GrammaticalGender.Feminine) => "十"
11.ToWords(GrammaticalGender.Neuter) => "十一"
122.ToWords(GrammaticalGender.Neuter) => "一百二十二"
3501.ToWords(GrammaticalGender.Neuter) => "三千五百零一"
(-3).ToWords(GrammaticalGender.Neuter) => "负 三"

```

### 罗马数字转换

```csharp
1.ToRoman() => "I"
2.ToRoman() => "II"
3.ToRoman() => "III"
4.ToRoman() => "IV"
5.ToRoman() => "V"
6.ToRoman() => "VI"
7.ToRoman() => "VII"
8.ToRoman() => "VIII"
9.ToRoman() => "IX"
10.ToRoman() => "X"
  
"I".FromRoman() => 1
"II".FromRoman() => 2
"III".FromRoman() => 3
"IV".FromRoman() => 4
"V".FromRoman() => 5
```

### 公制数字转换

```csharp
1d.ToMetric() => "1"
1230d.ToMetric() => "1.23k"
0.1d.ToMetric() => "100m"

"1".FromMetric() => 1
"1.23k".FromMetric() => 1230
"100m".FromMetric() => 0.1
```

### 字节大小转换

```csharp
var fileSize = (10).Kilobytes();

fileSize.Bits      => 81920
fileSize.Bytes     => 10240
fileSize.Kilobytes => 10
fileSize.Megabytes => 0.009765625
fileSize.Gigabytes => 9.53674316e-6
fileSize.Terabytes => 9.31322575e-9

var maxFileSize = (10).Kilobytes();

maxFileSize.LargestWholeNumberSymbol;  // "KB"
maxFileSize.LargestWholeNumberValue;   // 10

7.Bits().ToString();           // 7 b
8.Bits().ToString();           // 1 B
(.5).Kilobytes().Humanize();   // 512 B
(1000).Kilobytes().ToString(); // 1000 KB
(1024).Kilobytes().Humanize(); // 1 MB
(.5).Gigabytes().Humanize();   // 512 MB
(1024).Gigabytes().ToString(); // 1 TB
```

还可以选择用`b`，`B`，`KB`，`MB`，`GB`，`TB`格式化显示。另外使用`#.##`作为默认格式，四舍五入数字到小数点后两位：

```csharp
var b = (10.505).Kilobytes();

// Default number format is #.##
b.ToString("KB");         // 10.52 KB
b.Humanize("MB");         // .01 MB
b.Humanize("b");          // 86057 b

// Default symbol is the largest metric prefix value >= 1
b.ToString("#.#");        // 10.5 KB

// All valid values of double.ToString(string format) are acceptable
b.ToString("0.0000");     // 10.5050 KB
b.Humanize("000.00");     // 010.51 KB

// You can include number format and symbols
b.ToString("#.#### MB");  // .0103 MB
b.Humanize("0.00 GB");    // 0 GB
b.Humanize("#.## B");     // 10757.12 B
```

## 总结

使用*Humanize*，可以让你的应用程序UI显示更加人性化的同时，提高你的开发效率。本文中只介绍了*Humanize*中的一部分功能，更多功能请访问[Humanizer](https://github.com/Humanizr/Humanizer)。