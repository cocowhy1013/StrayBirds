---
layout: post
title: Faster Mutation Analysis via Equivalence Modulo States (ISSTA2017)
category: 阅读
comments: true
---


## 引言

本文是关于软件工程会议[ISSTA2017](http://conf.researchr.org/home/issta-2017)录用的论文“[Faster Mutation Analysis via Equivalence Modulo States](http://xueshu.baidu.com/s?wd=paperuri%3A%2858fc680748b07dfa0259884a40309fe8%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Farxiv.org%2Fabs%2F1702.06689&ie=utf-8&sc_us=982844565878477910)”的个人阅读和分析，单纯用于个人阅读记录，观点和分析仅供参考，有些翻译少许生硬，可以参考英文解释。本文是关于mutation testing的一个效率优化的工作，需要了解mutation testing相关知识可移步我的另一篇文章“A Brief Introduction to Mutation Testing”。

Mutation testing在其使用过程中的scalability问题是非常普遍的，即使是一个代码规模较小的程序都可能会产生非常多的mutant，很容易便达到成千上万的mutant数目，全部考虑并进行测试执行无疑是非常耗时并且有时甚至做不到的，如何对这样的过程进行加速就是本文研究的问题。我在阅读本文时其实认为整篇工作与Just的"[Efficient mutation analysis by propagating and partitioning infected execution states](https://www.researchgate.net/profile/Rene_Just/publication/266659273_Efficient_mutation_analysis_by_propagating_and_partitioning_infected_execution_states/links/54d3cdd20cf2970e4e6057b1.pdf)"的idea非常相似，建议可以一起阅读，本文在文中也有提到此工作并做了相应的致敬，解释了与其不同的区分点。

## 已有的研究工作
在对于如何提高mutation testing的测试速度上，本文将已有工作分为了两大类：

### Lossy Acceleration
这一类指的是加速过程会影响测试的精度(precision)的工作，有两类典型的工作：
####  1. weak mutation
[Weak mutation](http://xueshu.baidu.com/s?wd=paperuri%3A%281bccc09c4d053ff47f4794cd8f30c5cb%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Fieeexplore.ieee.org%2Fdocument%2F1702959%2F&ie=utf-8&sc_us=7357925753506907425)工作认为mutant不需要完整执行，只需要对于mutate point进行执行，观察当前状态，不需要执行完全到最后，但是这样的方式观察到的中间mutate过后的程序状态与是否violate测试用例是不完全等价的。中间状态与golden version完全相同，自然最终结果一定一致（我们全部考虑的是确定性程序，那些不确定程序咱不讨论），但反之则不成立，即中间状态不同不能意味着最终一定会violate测试用例。因此，weak mutation单纯观察测试中间的程序状态可能会造成较多的mutation score虚高，造成结果的不精确。

#### 2. sameple
除了上面的weak mutation工作，sample也是一个非常直觉的策略，有几种常见的sample办法：一种是对于所有的产生的mutant进行随机的sample出一个子集进行测试，这里随机sample也是为了避免一些人为的bias，较为常用；还有是对于mutant进行一些预处理后对于每一类的mutant都进行sample，可以有意识的选出一些作者认为较多样或较有代表性的mutant，这一方式通常是认为随机选择过于粗暴，选出的子集未必具有代表性也可能受制于mutant种类之间的不平衡，影响最终的测试结果；还有一种是对于mutation operator的sample，选择一些相比较而言更加合适的operator子集，总而减少最终的产生的mutant数目。

### Lossless Acceleration
这一类则是在不损失精度的前提下，达到只需要部分运行mutant就可以得到全集的测试结果，主要有三份典型的工作：
#### 1. mutation schemata
[Mutation schemata](http://xueshu.baidu.com/s?wd=paperuri%3A%282809f8063dde3aad31d92a09a21e7b76%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Fdx.doi.org%2F10.1145%2F154183.154265&ie=utf-8&sc_us=12278191206072157693)：在编译时期进行优化加速，对于代码编译中可复用的部分可以只编译一次便可以进行复用，极大地减少mutant的编译时间；

#### 2. Split-stream execution
Split-stream execution ([1](https://www.computer.org/csdl/proceedings/icst/2016/1827/00/1827a320-abs.html)，[2](http://xueshu.baidu.com/s?wd=paperuri%3A%289e82a4714ebac38be5968f0671d228b2%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Fieeexplore.ieee.org%2Fdocument%2F7883390%2F&ie=utf-8&sc_us=16933628091378364782)，[3](http://xueshu.baidu.com/s?wd=paperuri%3A%283cc51623c21499729b7a460298899a6f%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Fonlinelibrary.wiley.com%2Fdoi%2F10.1002%2Fspe.4380210704%2Fpdf&ie=utf-8&sc_us=17805333039950884866))：对于每个mutation来说，实际mutate位置之前执行的代码其实都是和golden version完全一致的，可以只执行一次即可，每个mutant都可以减少实际的这部分执行时间；

#### 3. Equivalence Modulo States
Equivalence Modulo States ([1](http://homes.cs.washington.edu/~mernst/pubs/state-infection-issta2014.pdf)，[2](http://xueshu.baidu.com/s?wd=paperuri%3A%2858fc680748b07dfa0259884a40309fe8%29&filter=sc_long_sign&tn=SE_xueshusource_2kduw22v&sc_vurl=http%3A%2F%2Farxiv.org%2Fabs%2F1702.06689&ie=utf-8&sc_us=982844565878477910))：对于实际执行中，两个并不equivalent的mutant，在当前的输入下可能是表现一致的，例如a=b+1与a=b*2并不等价，但若当前状态是b=1则两者其实是一致的，这样的两个mutant其实只需要执行一个，就可以得到两个分别的结果。本文和Just的工作都是这一类的方法。

这三类方法优化的level不同，可以相互结合使用的，本文则是在SSE(Split-stream execution)的方法上结合进行研究的。

## 本文的技术与方法