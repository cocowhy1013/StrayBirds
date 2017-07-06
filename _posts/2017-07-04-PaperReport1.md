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
对于多个mutant的执行加速，不仅只可以省略mutated statement执行之前的重复执行，对于与golden version不equivalent的mutant，也有可能在当前状态下表现一致，本文的工作则是，利用主线程进行从头开始执行程度，每遇到某一行mutated statement时（至少有一个mutant在该行发生改变），则尝试分析mutant该行对当前状态的影响，若与golden version没有影响则不从主线程分出并行的线程专门处理该mutant，若与golden version相比较有不同的影响，但是多个mutant影响相同，则只需要在这多个mutant中选择执行一个即可。

文中用一个很明显的图示表示了不同加速办法之间的区分：
![1.png](http://oskh0ynw2.bkt.clouddn.com/1.png "Dynamic Mutation Analysis with Equivalence Analysis")

其中，圈中不同数字表示不同状态，箭头粗细表示需要合并加速的迫切程度。

传统mutation testing 如图(a)-(d)表示，对于给定一个测试用例，每一个mutant单独执行一次，主线程执行一次，这样做无疑是非常费时的，SSE工作，对于程序开始至每一个mutant实际改动的mutated statement这一部分代码的执行进行了复用，由于程序还未执行插入fault的代码，mutant与golden version执行一定是保持一致的，如图(e)表示，M1在loop发生之前代码发生改变，则会从主线程分出子线程执行M1，M2和M3均在loop内部代码发生mutate，则会在状态2进入loop之后分出各自线程执行各自的mutant。

但是，SSE方法对于每一个mutant都会最后新开线程进行执行，只能够优化加速实际的mutated statement执行之前的部分，这一优化还有很大的提升空间，例如当产生的所有的mutant数量庞大时，即使利用SSE的优化，执行代价也是非常高的，尤其对于mutated statement代码在程序入口附近是，优化的比例就非常小了，但是实际上对于不同mutant在mutated statement之后的执行也是能够优化的。一种是对于与原始的equivalent mutant的执行是可以省略的，一种是当前状态下，mutated statement执行过后状态与golden version一致的情况可以省略，一种是当前状态下，多个mutant的同一行发生mutated的statement执行后的状态相互一致，也是没必要区分开来分别执行的。这是三个优化的场景，第一份通常是mutation testing中关于equivalent mutant如何检测并去除，这里不做赘述，主要关注后两点。

本文AccMut方法的是希望能够“further reduce the redundancies in execution by exploiting the equivalence modulo the current state”，主要思想是通过主线程进行golden version执行，每遇到存在mutant发生的mutated statement时，尝试对这些在该处进行mutated的mutant进行分析执行，注意这里是trial execution，主要分析对于当前状态会发生何种改变，若存在多个mutant，对其发生的改变方式进行聚类，对当前状态做相同的改动操作则聚在一类，一个类只需要执行一次。当然，若对当前状态的改动方式与golden version一致，则将其与golden version看作一类，无须分出特别的线程进行后续执行。若聚类个数为n，则我们则会split出n-1个线程进行每个mutant的操作，主线程可继续进行golden version聚类的操作。

这里，本文的难点是如何对每个mutant的mutated statement前后的状态操作进行分析，如何得到前后状态从而帮助进行聚类，这是比较困难的，尤其对于mutated statement可能带来较多的side effect时是难以分析的。这里，作者通过记录abstract change来代替记录concrete change，即若有一行代码为a=fac(x);作者只记录函数名称，函数参数等信息，而不会具体的去分析该函数对于程序当前状态产生多大的影响。这里是基于一个确定性中程序认为成立的一条：“在同一个程序状态下，abstract change相同则一定会得到相同的执行后的程序状态”，因此我们将abstract change相同的mutant聚在一起，这一条的反之并不成立，在同一个程序状态下，abstract change不同， 也未必一定得不到相同的程序状态，因此这天然说明我们的方法较为保守，也有可能有一部分由于abstract change不同但是程序状态相同的mutant没有聚在一个类，从而多fork了线程执行这样的程序。但是我们方法能够聚在一起的一定是程序状态一致的，本文也以一个定理Theorem 3.1进行了保证。

实际的implementation中，主要关于三个函数，一个fork，一个try和一个apply。fork函数主要是为了进行线程的新建，以及需要一些后续的调度，try和apply则是在不真正改变程序状态的基础上，分析对于mutated statement其执行后的状态进行判断，若mutated statement操作较为简单，可以直接对于给定的内存地址进行操作，得到其改动的内存信息的前后值，从而判断多个mutant的执行后的程序状态是否一致，对于类似procedure call则利用之前提到的abstract change来进行帮助判断。根据try和apply得到的信息进行聚类之后，可以split合适的线程进行mutant各个聚类的执行。

主要思想就是这样，下面是一些实验结果：
![4.png](http://oskh0ynw2.bkt.clouddn.com/4.png)

![2.png](http://oskh0ynw2.bkt.clouddn.com/2.png)

上图中列出了对于这样的11个subject生成的数量从493~173683个mutant，每一个subject分别在AccMut(本文方法)，SSE(Split-stream Execution)和MS(mutant schemata)上面的执行速度，可以发现AccMut在执行速度上达到了SSE的2.56倍，MS的8.95倍，这是非常可观的，反映到实际的执行时间，可以发现原先利用MS需要跑10小时多，SSE需要3小时的mutant，AccMut只需要1小时10分钟就可以执行结束，得到不损失精度的结果。

![3.png](http://oskh0ynw2.bkt.clouddn.com/3.png)

而反映到实际的执行mutant上来说，对于每一个test，AccMut只需要执行SSE的35.6%，MS的3.2%个mutant，也正是由于AccMut大幅减少了分析后真正需要执行的mutant的个数，从而大大减少了整个mutant集合全部执行完的时间。

## 一些想法与总结

本文在实验数据上是非常amazing的，不管是速度的提升，还是mutant省去的数目，都非常不错。在评估实验上为了避免多线程程序的执行的不稳定，只允许单线程进行处理，最后其实衡量的串行衡量了执行每一个mutant的时间。与Just方法的对比上，在整个判断equivalent module state上较为简单，也利用abstract change能够handle复杂的函数调用，但是单就idea来说，确实是两份比较类似的工作。