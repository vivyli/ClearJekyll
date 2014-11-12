---
layout: post
title: "自动语言/编码检测算法"
description: ""
category: algorithm
tags: [algorithm]
---
{% include JB/setup %}
Mozilla的自动编码检测代码参考了论文[《A composite approach to language/encoding detection》] (http://www-archive.mozilla.org/projects/intl/UniversalCharsetDetection.html)，今天正好把这篇文章读了一下。文中总结了自动语言/编码检测的三种算法，做了比较，并提出了一个结合三种方法的算法。

<!-- more -->

####Coding scheme method####

这是一个比较直观的方法，一般被用于multi-byte encodings，因为在multi-byte encoding中，并不是所有的code point被用到，因此可以逐个字节检测，当发现一个不合法的byte时就可以判断尝试的编码类型错误。

在这个方法中，为每一种编码类型建立一个状态机，当输入一个字节时根据状态机转移状态，检测算法只需要关心这个状态机的三种状态：START、ME、ERROR，其中START表示开始状态；ME表示对与输入序列属于猜测的编码类型，并且不可能是其他编码的序列，因此可以可以直接给出猜测的结果；而ERROR则是输入字节序列是非法的，可以直接排除这个编码。

这个方法 对于一些multi-byte的编码类型来说是比较有效的，如UTF-8、ISO2022-xx、HZ，但是对于如EUC-JP、EUC-CN、EUC-KR来说效果并不好，对与single-byte的编码效果很差， 能够处理任意的输入文本。


####Character Distribution####

Coding schema method是一个规则方法，而另外两种则是概率的方法：对字符的分别做一个统计，一般来说任何语言中，一些字符会比其他的字符出现的概率更好，如对于用简体中文（6763个字符）来说，512个字符的出现累积概率约80%；繁体中文约75%；日文约92%；韩文约98%。

在这个方法中，对每个输入序列字符进行判断，将其分为两个“frequently used”和“not frequently used”，如果输入字符属于频繁出现的512个字符中，定义为“frequently used”，否则则为“not frequently used”。根据这个统计计算distribution ratio：

Distribution Ratio = (“frequently used”数量) / （“not frequently used”数量）
定义：

confidence level = （输入字符序列的distribution ratio） / （ideal distribution ratio）
其中ideal distribution ratio为根据训练语料库得到的数据，如简体中文中，512个字符出现的累积概率是0.79135，因此ideal distribution ratio = (0.79135) / (1 – 0.79135) =3.79

使用这个方法计算每个输入序列的confidence level，得到最高的编码猜测结果。

这个方法对于multi-byte编码效果较好，但只能处理特定的文本，因为需要文本的统计结果。


####2-Char Sequence Distribution####

Character Distribution方法是基于单个字符的，而这个方法则是基于连续两个字符的，并且结合了character distribution方法，定义了confidence level为：

confidence level = (TotalSequence - NegativeSequenceCount) / TotalSequence * FrequentCharCount / TotalCharacter
其中TotalSequence：目标文本中的相邻字符序列数量

NegativeSequenceCount：目标文本中的2-Char序列中，属于低概率出现序列数量

FrequentCharCount：“frequently used”字符的数量

TotalCharacter：字符总数

这种方法对于single-byte的效果较好，而对于multi-byte的效果较差，当样本数量较小时效果好，只对特定的文本类型有效。

因为这三种算法都只对特定的编码方式的检测效率较高，因此作者根据输入字符的类型分别用不同的方法进行检测，得到检测的结果。
