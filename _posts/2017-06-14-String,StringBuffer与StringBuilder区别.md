---
layout: post
title:  String,StringBuffer与StringBuilder区别
date:   2017-06-14 22:23:03 +0800
tags: 能工巧匠集
---

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String、StringBuffer、StringBuilder这三个类，学习Java的同学应该都有印象。我刚开始学习的时候也背过它们之间的区别。但实际使用中，往往还是一知半解。等到我开始接触到iOS开发时，开始熟练使用里面的NSString与NSMutableString后。才慢慢发现Java中这三者的关系。这两天回过头来，又重新复习一遍，豁然开朗。

## 可变字符串与不可变字符串

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;在iOS中NSString表示不可变字符串，NSMutableString表示可变字符串。在Java中String是不可变字符串（因为Java简洁的语法，导致很长一段时间我都误认为String就是可变字符串）。而实际上String只是一个不可变字符串，而StringBuffer和StringBuilder才是Java中的可变字符串。比如你可能会像这样写：

    String string = "abc" + "def";
    string = string + "xxx";

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;String型的变量string明明已经发生变化，为什么说是不可变呢? 其实这是一种欺骗，JVM是这样解析这段代码的：首先创建对象string，创建两个零时变量赋予一个abc，一个def，然后组合他们并赋值给了创建的string变量，这一过程中一个创建了3个字符串常量。第二行代码再创建一个新的对象string，也就是说我们之前对象string并没有变化（实际上整个过程中创建出的所有字符串都会存储在常量池中），所以我们说String类型是不可改变的对象了。

&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;而StringBuffer与StringBuilder就不一样了，他们是字符串变量，是可改变的对象，每当我们用它们对字符串做操作时，实际上是在一个对象上操作的，这样就不会像String一样创建一些而外的对象进行操作的。



## StringBuilder与StringBuffer

StringBuilder：非线程安全的，单线程下使用，速度相对较快。

StringBuffer：线程安全的，多线程操作使用保证数据安全，速度较慢。

随着不断的学习，慢慢发现语言之间的相通之处。