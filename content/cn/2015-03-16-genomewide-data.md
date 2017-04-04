---
title: 理解基因组数据分析之数据读取与数据结构篇
date: 2015-03-16
slug: genomewide data
---

这一系列的文章源于edx上PH525这一系列[课程](https://www.edx.org/course/statistics-r-life-sciences-harvardx-ph525-1x)，从实用角度出发分为四部分：

- 原始数据读取与数据结构
- 高维数据的分组差异比较
- 基因组数据建模与可视化
- 结果注释与通路分析

其实就是按工作流程走的，首先设计实验，跑芯片，然后拿到机器读数。根据厂家不同，原始数据的处理方式也不同，我们采用基于R的bioconductor来进行分析。为此我们需要知道基因组数据的读取方法及在bioconductor中的数据结构。之后就是按照我们的科学问题进行差异比较，这里面会遇到假阳性等问题。之后就是构建模型排除一些影响，筛出我们真正关心的基因或序列片段。最后就是为这些基因片段进行注释与分析，回答科学问题。

## 原始数据读取

先说数据源，按技术分两种讲吧。

先看面向已知的microarrays，也就是微阵列基因芯片，可以用来测定基因表达。原理上你的微阵列上的探针序列是知道的，所以不太合适用来做未知基因的筛查或测序，当然这个看思路。微阵列基因芯片的应用场景还包括通过甲基化水平研究表观遗传表达、通过ChIP-on-chip研究转录因子或结合位点或着进行流行病学与基因组学的交叉去解决些科学问题。

另一种就是面向未知的，例如RNA测序或NGS，也就是二代测序，可以探索未知。就NGS而言，原则上一代二代三代测序都可以用来探索未知，例如测定物种基因组。另外可解决的科学问题包括研究同一物种不同人群的表达，同一个体不同器官的表达，同一器官不同生长阶段的表达，同一生长阶段不同环境因素下基因表达。这里面有些问题基因芯片也能做或配合一些技巧去做，但RNA测序或NGS得到的信息量更多一些。

其实在实际场景下有些人是不管技术的，但会区别厂商品牌。比较常见的就是affymetrix、illumina跟agilent这三家。原理不多说，基本都是依靠光信号。如果算上二代测序品牌可能还要多些，但商业化的大都能在bioconductor上搜索到相关数据的读取软件包。由于技术进步，高通量测序所得到的数据越来越多也越来越不直观，因此对这类数据进行数据分析就显得很重要了。

下面用Affy包为例，演示下读取数据。

~~~
# 安装bioconductor与affy包
# source("http://bioconductor.org/biocLite.R")
# biocLite()
# biocLite('affy')
library(affy)
# 读取样本采集信息
tab <- read.delim("sampleinfo.txt", check.names = FALSE, as.is = TRUE)
# 读取样本数据，探针层次
ab <- ReadAffy(phenoData = tab)
# 直接读取为基因层数据，适用于特定科学问题
ejust <- justRMA(filenames = tab[, 1], phenoData = tab) 
# 对样本进行背景校正与正则化，从探针层转化为基因层数据
e <- rma(ab)
~~~

原始的数据就是探针水平的光信号，也就是CEL文件，这个读取后要做背景矫正，矫正了背景要正则化，如果你有spike-in的探针也可以进行一把基于探针的背景矫正，都做完了就可以连同矫正后的误差等信息一并保存到一个ExpressionSet类型对象（Bioconductor里的基因芯片基础数据类型）中等待进一步处理。

在解释ExpressionSet类型对象的数据结构前，分析者有必要清楚这些矫正与正则化的原因。根据affymetrix的测序原理，每个基因大概有十几个探针，在芯片上我们可以通过加入已知浓度梯度的阳性或阴性探针来作为背景矫正的基础。这样我们以浓度作为x轴，响应作为y轴，得到的应该是十几条平行的直线。但真实的数据往往是在低浓度时探针变化不大而高浓度呈递增状态，也就是先平后升。如果我们用一个模型来描述就是

$$Y = a + bx + e$$

a表示低浓度截距，x表示浓度，b是浓度系数。这样低浓度时a比较大，浓度起作用有限，当浓度高时误差就不起作用了。这种情况下我们要描述单个探针的变化就比较麻烦了，得想办法从背景中提取出信号。厂商提供的解决方案是加入一些伪探针来模拟同样浓度下的噪声，用差减方式得到信号，或者对两者相关性建模模拟出信号，但这样同样会导致低浓度下信号方差。另一个方法就是对噪音建模，模拟出方差与信号的关系，这样造成了偏差，但实际效果不错。扣掉背景可以让我们知道一组探针光信号的真实变化。

除了背景矫正，另一个要做的就是正则化，出发点是我们通常认为多数基因在单一处理下应该是不变的，但实际数据确可能出现整体的偏移。这种情况我们可以用局部回归或分位数回归等正则化方法让数据回到零为中心的状态，配合加标的方法可以做到最大程度上减弱偏差与噪音的影响。

上面啰嗦半天其实你也看到了，代码里就用了`e <- rma(ab)`一句就搞定了，因为这条命令包含了背景扣除与正则化还有将探针数据转为基因层数据等一系列命令，算是个懒人包，可以用来处理微阵列数据。如果你打算处理RNA测序数据，可能面临的问题不太一样，需要的模型也要重新设计，听我一句劝，用别人现成的吧，除非你是学生物统计的。到这一步你已经有一组基因层的数据了。

## 数据结构

ExpressionSet是Bioconductor为了衔接后续的基因芯片表达数据分析包提供了一个通用的数据类型，也是eSet对象的一个类。不管哪一家的芯片都要转成这个类型才好进行下一步分析，这样做的原因往往是方便后续分析的开发人员，也便于提供统一接口。用R语言说就是可以直接为ExpressionSet对象写方法，而不用对每一家公司都提供数据格式支持，当然可能损失点效率，只是可能。

~~~
# 读取Biobase包与示例数据，该包提供一些对ExpressionSet类型对象的基础操作
library(Biobase)
data(sample.ExpressionSet)
sample.ExpressionSet
~~~

这个类型有如下要求：

- 至少要包括assayData（也就是第一部分转换到的）
- assayData的数据保存在m＊n的矩阵中，m个基因，n个样本
- phenoData会用来保存n＊p的矩阵，n个样本，p个样本特征例如实验分组，测定时间之类的
- featureData用来保存每个基因的信息，m＊t，m个基因，t个基因描述。这里如果你读取数据直接读了探针层的，这个部分可以描述基因；如果是基因层的，这里用来保存基因的注释
- experimentData用来保存该组数据的实验信息与出处等
- annotation用来保存一个供其他Bioconductor包调用的注释，可理解为特征码
- protocolData用来保存基因芯片上真对每一个测定数据所保存的方案或信息

搞清楚这个结构的目的其实在于数据共享与数据操作。当你进行了一组实验并发表文章后，你的数据可以直接用这个对象类型封装为一个Bioconductor包上传供其他研究人员使用。至于数据操作则是很多方法都是针对这个类型写的，例如提取assayData直接用`exprs(e)`就可以把阵列矩阵搞出来了，详细信息都可以在帮助页面查到。此外也有SnpSet的数据类型来对应相关的研究内容。

## 小结

读到这里希望理解下面几点：

- 基因芯片的数据结构
- 读取基因芯片时对原始数据的预处理原因
- 读取或构建基因芯片数据集