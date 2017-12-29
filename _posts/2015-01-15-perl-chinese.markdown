---
layout: post
category: program
title:  "perl正则，unicode匹配"
tags: [正则,shell,编程]
---

把一个文件中的含有中文的行过虑出来， 使用perl正则的命令

```
cat strings.xml | grep -P "[一-龥]+"
```

但在网上搜索，很多匹配中文的正则应该这样写

```
cat strings.xml | grep -P "[\u4e00-\u9fa5]+" 或者
cat strings.xml | grep -P "[\x4e00-\x9fa5]+"
```

可是一个提示“grep: PCRE does not support \L, \l, \N{name}, \U, or \u”，另一个也不解决问题。

原因其实是很简单，后面的两个正则用于编程语言， (如java, js)，这些语言本身就可以用unicode码来表示一个字符。但grep命令参数不能这样表示，所以达不到想到的效果。

扩展： 所有的中文字符在unicode编码中的范围是：\u4e00-\u9fa5， 怎样找到和对应成字符：一-龥， 网上也有很多在线转换工具，这里记录另外两种方式。

1.Windows系统查找字符， 控制面板->字体， 左边那一栏有一个“查找字符”，点击打开

  ![perl_cn_charater](../assets/2015-01-15_perl_cn_charater.jpg)

  ![perl_cn_charater1](../assets/2015-01-15_perl_cn_charater1.jpg)

2.使用微软拼音输入法，安装、切换、配置成unicode码输入，便可输入对应unicode码看到中文字符。

![perl_cn_input](../assets/2015-01-15_perl_cn_input.jpg)
