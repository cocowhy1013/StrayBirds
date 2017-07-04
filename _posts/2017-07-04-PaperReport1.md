---
layout: post
title: ISSTA2017(Faster Mutation Analysis via Equivalence Modulo States)
category: 阅读
comments: true
---


## 引言

本文是关于软件工程会议[ISSTA2017](http://conf.researchr.org/home/issta-2017)录用的论文“[Faster Mutation Analysis via Equivalence Modulo States](http://xueshu.baidu.com/s?wd=paperuri%3A%2858fc680748b07dfa0259884a40309fe8%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Farxiv.org%2Fabs%2F1702.06689&ie=utf-8&sc_us=982844565878477910)”的个人阅读和分析，单纯用于个人阅读记录，观点和分析仅供参考，有些翻译少许生硬，可以参考英文解释。本文是关于mutation testing的一个效率优化的工作，借此机会这里简要来补充一下相关知识。

## 背景介绍

mutation testing是一种通过对程序(可看做golden version)中人为插入一些错误（seeded faults）来构造新的程序变体（mutant），用以模拟实际生活中程序中可能存在的bug。通常可以用其来对于测试用例的质量进行衡量，是一个常用的metrics，一般来说认为一个测试用例能够kill掉越多的mutant，则认为该测试用例的检错能力更强。对于mutation testing不了解的可以移步Yue Jia和Mark Harman教授的survey工作“[An Analysis and Survey of the Development of Mutation Testing](http://ieeexplore.ieee.org/document/5487526/)”,这一篇survey对于mutation testing的发展和已有的研究工作都做了一个很好的梳理。

在此研究领域中，主要有几个不同维度的工作：

### 1. 变异操作
变异操作(mutation operator)是指在原始程序中通过一些变换，例如改变语法操作等来对原始程序做一些修改，已有两大类的变异操作traditional operator和class operator，也可称为method-level和class-level operators这里在下面列出常用工具[MuJava](http://delivery.acm.org/10.1145/1140000/1134425/p827-ma.pdf?ip=114.212.81.253&id=1134425&acc=ACTIVE%20SERVICE&key=BF85BBA5741FDC6E%2E180A41DAF8736F97%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35&CFID=956104855&CFTOKEN=90259377&__acm__=1499163920_8a2c80a29e178a0f4fe3841c2fd7e648)中支持的两大类operator的列表：

![traditionOp](2017-07-04/traditionOp.png "method-level operators supported in MuJava")
![classOp](2017-07-04/classOp.png "class-level operators supported in MuJava")
以最简单的AOR来举例，原始程序中包含一行简单的a=b+c语句，AOR的变异操作会对其中的算术运算符进行替换，将原始“a=b+c”替换成“a=b-c”,这是一个简单便于理解的例子，实际中其实method-level的变化大多也较为简单。针对不同领域，有研究工作设计了不同的operator，以及针对不同operator的横向比较和研究。

### 2. 变异操作次数
这里主要推荐Yue Jia教授的介绍higher order mutation testing的工作[Higher Order Mutation Testing](http://xueshu.baidu.com/s?wd=paperuri%3A%28256e59987845f8910d43dbee99522737%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Fwww.sciencedirect.com%2Fscience%2Farticle%2Fpii%2FS0950584909000688&ie=utf-8&sc_us=15796804892534341782)，传统mutation testing为了方便分析错误，只允许每次插入一个fault，即只允许程序中一处发生mutate，higher order mutation testing提出允许程序中多次进行mutate并对其被kill难度、影响与检错能力进行了探究，并对多个错误点之间可能存在的coupling进行了分析，是一个分析地比较详细且全面的工作。当然，大多数应用mutation testing的工作还是基于first order上面来做的。

### 3. 生成mutant区分
下面介绍几种常常讨论到的mutant类型区分，当然这些概念并不是严格独立的。
#### Equivalent mutant
这一概念指的是尽管程序代码上有部分语法改动，但是实际改动之后与golden version的效果是一样的，也就是这样的mutant其实与golden version程序是完全一致的。例如，golden version与mutantA只有一行代码不同，其余部分完全一致（这里用first order来说明），golden version中该行代码为a=a+1，mutantA中将其改成a=a++，很容易就能看出，尽管两行代码语法不同，但实际都是对a进行加一操作，因此该mutantA与golden version在程序功能上完全一致，无论是什么测试用例都不可能将该mutant来kill掉（kill掉指的是某测试用例能够在mutant和golden version表现不同）。

#### Duplicated mutant
这一概念指的是mutant之间相互一致的情况，例如，某程序P利用已有的mutation testing工具可以生成100个mutant，从mutant0~mutant99，但是其实mutant1=mutant2=...mutant10，此时，尽管有100个mutant但实际种类只有某几种，有可能导致某一类的mutant所占数目较多的时候，对于实验结果有误导，并且使得实验结果不可控。这一类情况也是可能的，比如golden version代码是a=a+1，mutantA中代码改为a=a-1，mutantB中代码改为a=a--，这两个mutant的程序行为与golden version确实不同，但是mutantA和mutantB的程序行为就是一样的，若选择mutantA为有效mutant，则mutantB即为duplicate的。

#### Stubborn mutant
源程序基础上生成的mutant通常数量很多，不同的测试用例通常可以kill掉不同的mutant列表，但是有一部分mutant已有测试用例都没有办法kill，这一部分mutant可能是equivalent mutant，即无论是什么测试用例都没有办kill，当然也有可能是stubborn mutant即有希望被kill只是已有的测试用例不足以kill，需要设计更好的测试用例来覆盖有效地kill这些stubborn mutant(顽固变体)。已有工作通常专注于检测并排除equivalent mutant，以达到得到实际可以被kill的那部分测试用例。Mike Papadakis有一份这方面很好的工作[Trivial Compiler Equivalence](http://delivery.acm.org/10.1145/2820000/2818867/p936-papadakis.pdf?ip=114.212.81.253&id=2818867&acc=ACTIVE%20SERVICE&key=BF85BBA5741FDC6E%2E180A41DAF8736F97%2E4D4702B0C3E38B35%2E4D4702B0C3E38B35&CFID=956104855&CFTOKEN=90259377&__acm__=1499165914_ab10f5f1acdfb62eab3d314dc68f76e8)，idea很简单“编译后相互一致的mutant一定是equivalent的”，效果很好。

#### Trivial mutant

#### Subsumed mutant

### 4. mutant编译与运行优化


### 5. 其他方面的探究