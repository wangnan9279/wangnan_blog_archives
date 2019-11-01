---
title: Lucene学习笔记
link_title: lucene-study-note
date: 2018-08-18 16:54:36
tags: [Lucene]
categories: Lucene
thumbnailImage: https://i.loli.net/2019/09/24/5h8JU1iRIONgMAs.png
thumbnailImagePosition: left
---
<span/>
<!-- more -->
![](https://i.loli.net/2019/09/24/5h8JU1iRIONgMAs.png)
<!-- toc -->

> 概要：
1. 全文检索的原理和基本概念（铺垫）
2. Lucene简介，索引文档和检索文档的过程（主要）
3. Lucene 相似度评分算法（拓展）


# 全文检索原理（铺垫）

## 数据分类
生活中的数据总体分为三种：
- 结构化数据，固定格式和长度，如数据库数据，元数据等
- 非结构化数据，无固定格式和长度，如邮件，word文档，商品描述信息，非结构化数据也称为为全文数据
- 半结构化数据，如XML，HTML等，当然根据需要按结构化数据来处理，也可抽取出纯文本按非结构化数据来处理

## 搜索分类
按照数据的分类，搜索也分为两种：
- 对结构化数据的搜索，如对数据库的搜索，用SQL语句
- 对非结构化数据的搜索，如用Google和百度搜索

## 对非结构化数据（全文数据）搜索方法
- 顺序扫描法(Serial Scanning)

从头到尾的扫描，比如windows的搜索文件，Linux下的grep命令，这种方法比较原始，但对于小数据量的文件，这种方法还是最直接，最方便的。但是对于大量的文件，这种方法就很慢了。

- 全文检索（Full-text Search）

有人可能会说，对非结构化数据顺序扫描很慢，对结构化数据的搜索却相对较快（由于结构化数据有一定的结构可以采取一定的搜索算法加快速度），那么把我们的非结构化数据想办法弄得有一定结构不就行了吗？

这就是全文检索的基本思路：也即将非结构化数据中的一部分信息提取出来，重新组织，使其变得有一定结构，然后对此有一定结构的数据进行搜索，从而达到搜索相对较快的目的。

这部分从非结构化数据中提取出的然后重新组织的信息，我们称之**索引**

直观的例子就是字典，可以按照偏旁部首和读音去找到对应的页数范围

**这种先建立索引，再对索引进行搜索的过程就叫全文检索(Full-text Search)**

## 全文检索的过程
![image](https://upload-images.jianshu.io/upload_images/79431-9b8e7e82eecdd251.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/604/format/webp)

提出三个问题：
- 索引里面究竟存些什么？(Index)
- 如何创建索引？(Indexing)
- 如何对索引进行搜索？(Search)


###  索引里面究竟存些什么？(Index)
非结构化数据中所存储的信息：文件->字符串

而我们想搜索的是：字符串->文件

如果索引总能够保存从字符串到文件的映射，则会大大提高搜索速度。保存这种信息的索引称为**反向索引**或者**倒排索引**

反向索引报错信息一般如下：
假设有100个文档，id为1-100，我们得到如下的结构

![image](https://upload-images.jianshu.io/upload_images/79431-a913c229a41c03ab.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/598/format/webp)

左边报错的一系列字符串，称为**词典**
右边的文档链表称为**倒排表**

比如说，我们要寻找包含字符"lucene"而且包含"solr"的文档，只需要下面几步：
1. 取出包含字符串“lucene”的文档链表。
2. 取出包含字符串“solr”的文档链表。
3. 通过合并链表，找出既包含“lucene”又包含“solr”的文件。

![image](https://upload-images.jianshu.io/upload_images/79431-22a392e2974a611f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/606/format/webp)

索引反映出了全文搜索优势：一次索引，多次使用。

### 如何创建索引？(Indexing)
通过一个例子讲解

**第一步，准备一些要索引的文档（Document）**

> 文件一：Students should be allowed to go out with their friends, but not allowed to drink beer.

> 文件二：My friend Jerry went to school to see his students but found them drunk which is not allowed.


**第二步，将原文档传给分词组件（Tokenizer）**

分词组件做的几件事：
1. 将文档分成一个一个单独的单词
2. 去除标点符号
3. 去除停词(Stop word)：语言中最普通的一些单词，比如：the”,“a”，“this” 由于没有特别的意义，因而大多数情况下不能成为搜索的关键词

经过分词(Tokenizer)后得到的结果称为词元(Token)：
“Students”，“allowed”，“go”，“their”，“friends”，“allowed”，“drink”，“beer”，“My”，“friend”，“Jerry”，“went”，“school”，“see”，“his”，“students”，“found”，“them”，“drunk”，“allowed”。

**第三步，将词元（Token）传给语言处理组件（Linguistic Processor）**

对于英语，语言处理组件做的几件事：
1. 变为小写(Lowercase)。
2. 将单词缩减为词根形式，如“cars”到“car”等。这种操作称为：stemming。
3. 将单词转变为词根形式，如“drove”到“drive”等。这种操作称为：lemmatization。

语言处理组件(linguistic processor)的结果称为词(Term)

在我们的例子中，经过语言处理，得到的词(Term)如下：

“student”，“allow”，“go”，“their”，“friend”，“allow”，“drink”，“beer”，“my”，“friend”，“jerry”，“go”，“school”，“see”，“his”，“student”，“find”，“them”，“drink”，“allow”。

也正是因为有语言处理的步骤，搜索“drive”，“driving”，“drove”，“driven”也能够被搜到,因为在我们的索引中，“driving”，“drove”，“driven”都会经过语言处理而变成“drive”，

**第四步，将得到的词(Term)传给索引组件(Indexer)**
1. 利用得到的词(Term)创建一个字典。

![image](https://upload-images.jianshu.io/upload_images/79431-2b26bc124ab2228c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/442/format/webp)

2.对字典按字母顺序进行排序

![image](https://upload-images.jianshu.io/upload_images/79431-bc727881a13a8e98.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/442/format/webp)

3. 合并相同的词(Term)成为文档倒排(Posting List)链表。

![image](https://upload-images.jianshu.io/upload_images/79431-2a9072f16117d966.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/599/format/webp)

在此表中，有几个定义：
- Document Frequency 即文档频次，表示总共有多少文件包含此词(Term)。
- Frequency 即词频率，表示此文件中包含了几个此词(Term)

### 如何对索引进行搜索？(Search)
**第一步：用户输入查询语句**

**第二步：搜索应用程序对用户输入查询语句进行词法分析，语法分析，及语言处理**
步骤和索引过程中的语言处理几乎相同

**第三步：搜索索引，得到符合语法树的文档**

**第四步：根据得到的文档和查询语句的相关性，对结果进行排序。**


## 全文检索原理总结

![image](https://upload-images.jianshu.io/upload_images/79431-7370a4974548c96f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/601/format/webp)

刚才说的信息检索技术(Information retrieval)中的基本理论，Lucene就是对这种基本理论的一种基本的的实践。


# Lucene简介（主要内容）

## 概述

官网介绍：
> Apache LuceneTM is a high-performance, full-featured text search engine library written entirely in Java. It is a technology suitable for nearly any application that requires full-text search, especially cross-platform.

Lucene是一个高效的，可扩展的，基于Java的全文检索库

要注意的是它不是一个完整的搜索应用程序，而是为你的应用程序提供索引和搜索功能，

由于是它不是一个完整的搜索应用程序，所以有一些基于Lucene的开源搜索引擎产生，比如Elasticsearch和solr


## 索引结构
![image](https://upload-images.jianshu.io/upload_images/79431-c5930ee39168608f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1000/format/webp)

- index
同一文件夹中的所有的文件构成一个Lucene 索引
- Segment
一个索引可以包含多个段，段与段之间是独立的，添加新文档可以生成新的段，不同的段可以合并
- Document
- 文档是我们建索引的基本单位，不同的文档是保存在不同的段中的，一个段可以包含多 个文档
- Field
一篇文档包含不同类型的信息，可以分开索引，比如标题，时间，正文，作者等
- Term
词是索引的最小单位，是经过词法分析和语言处理后的字符串


lucene 的索引结构中，即保存了正向信息，也保存了反向信息
正向：索引(Index) –> 段(segment) –> 文档
(Document) –> 域(Field) –> 词(Term)
反向：
词(Term) –> 文档(Document



组件：
![image](http://onxkn9cbz.bkt.clouddn.com/lucene09.png)

- 被索引的文档用Document对象表示。
- IndexWriter通过函数addDocument将文档添加到索引中，实现创建索引的过程
- Lucene的索引是应用反向索引。
- 当用户有请求时，Query代表用户的查询语句。
- IndexSearcher通过函数search搜索Lucene Index。
- IndexSearcher计算term weight和score并且将结果返回给用户。
- 返回给用户的文档集合用TopDocsCollector表示。


此图简单介绍全文检索的流程对应的Lucene实现的包结构：

再详细到对Lucene API 的调用实现索引和搜索过程



## 索引过程

### 核心类：IndexWriter

IndexWriter 是 Lucene 用来创建索引的一个核心的类，他的作用是把 Document 对象加到索引中。

lucene6.5中构造方法：

```
public IndexWriter(Directory d, IndexWriterConfig conf) 
```
- Directory 

这个类代表了 Lucene 的索引的存储的位置，这是一个抽象类，它目前有两个实现，第一个是 FSDirectory，它表示一个存储在文件系统中的索引的位置。第二个是 RAMDirectory，它表示一个存储在内存当中的索引的位置。

![image](http://onxkn9cbz.bkt.clouddn.com/lucene15.png)

- IndexWriterConfig 是一个配置类，属性如下

![image](http://onxkn9cbz.bkt.clouddn.com/lucene16.png)

IndexWriterConfig主要包含了下面的信息：

- OpenMode 打开模式（CREATE-清空重建，APPEND-总是追加，CREATE_OR_APPEND-创建或追加）

- Analyzer 
分析器  在一个文档被索引之前，首先需要对文档内容进行分词处理

- Similarity 影响打分的标准化因子(normalization
factor)部分，对文档的打分分两个部分，一部分是索引阶段计算的，与查询语句无关，一部分是搜索阶段计算的，与查询语句相关

- Codec 编解码器 用来写入段的

- MergePolicy 段合并策略


### 示例代码

```
public class Index {

    public static void main(String[] args) throws IOException {

        //写索引类
        IndexWriter writer;

        //索引目录
        String indexDir = "C:\\lucene\\index";

        //储存方式
        Directory directory = FSDirectory.open(Paths.get(indexDir));

        //写索引类配置
        IndexWriterConfig config = new IndexWriterConfig(new StandardAnalyzer());
        config.setOpenMode(IndexWriterConfig.OpenMode.CREATE_OR_APPEND);
        writer = new IndexWriter(directory, config);

        //生成Document对象，Document对象就是对文档各个属性的封装
        String dataDir = "C:\\lucene\\data\\123.txt";//文件，里边要有.txt文件
        File file = new File(dataDir);
        Document doc = new Document();
        doc.add(new Field("filename", file.getName(), TextField.TYPE_STORED));
        doc.add(new Field("contents", new FileReader(file), TextField.TYPE_NOT_STORED));

        //写入索引，将生成的Document（目录）对象写入到索引中
        writer.addDocument(doc);

        //关闭
        writer.close();
    }
}
```

## 搜索过程
### 核心类 IndexSearcher

IndexSearcher 是用来在建立好的索引上进行搜索的。它只能以只读的方式打开一个索引，所以可以有多个 IndexSearcher 的实例在一个索引上进行操作。

步骤：
- IndexReader将磁盘上的索引信息读入到内存，INDEX_DIR就是索引文件存放的位置。
- 创建IndexSearcher准备进行搜索。
- 创建Analyer用来对查询语句进行词法分析和语言处理。
- 创建QueryParser用来对查询语句进行语法分析。
- QueryParser调用parser进行语法分析，形成查询语法树，放到Query中。
- IndexSearcher调用search对查询语法树Query进行搜索，得到结果TopDocs


示例代码：

```
public class Searcher {
    //这个方法是搜索索引的方法，传入索引路径和查询表达式
    public static void search(String indexDir,String query) throws IOException, ParseException {
        //打开索引目录
        Directory dir= FSDirectory.open(Paths.get(indexDir));
        // 创建搜索的Query
        IndexSearcher searcher=new IndexSearcher(DirectoryReader.open(dir));
        // 使用标准的分词器
        Analyzer analyzer = new StandardAnalyzer();
        // 在content中搜索，创建parser确定要搜索的内容，其中，第2个参数为搜索的域
        QueryParser parser=new QueryParser("contents",analyzer);
        // 创建Query表示搜索域为content中，包含搜索内容为query1的文档
        Query query1=parser.parse(query);
        long start=System.currentTimeMillis();
        // 开始搜索
        TopDocs hits=searcher.search(query1,11);
        long end=System.currentTimeMillis();
        System.out.println(hits.totalHits);
        System.out.println(end-start);//计算搜搜时间等
        //获取搜索的地址等
        for(ScoreDoc scoreDoc:hits.scoreDocs){
            Document doc=searcher.doc(scoreDoc.doc);
            System.out.println(doc.get("fullpath"));//地址，完整的
        }
    }

    public static void main(String[] args) throws IOException, ParseException {
        String indexDir="C:\\lucene\\index";//索引，index时建立的
        String query="food";//搜索的word
        search(indexDir, query);
    }
}
```






## 程序包和索引与搜索过程的对应关系

![image](http://onxkn9cbz.bkt.clouddn.com/lucene11.png)

- Lucene的analysis模块主要负责词法分析及语言处理而形成Term。
- Lucene的index模块主要负责索引的创建，里面有IndexWriter。
- Lucene的store模块主要负责索引的读写。
- Lucene的QueryParser主要负责语法分析。
- Lucene的search模块主要负责对索引的搜索。
- Lucene的similarity模块主要负责对相关性打分的实现。

# 第三部分：Lucene 相似度评分算法（similarity）

- TF/IDF (词频/逆文档频率)算法，ES5.0之前，TF/IDF是默认的评分算法,TF/IDF源于**向量空间模型(Vector Space Model)**

- BM25算法，ES5.0及之后（2017-05-04发布的5.4版本），BM25是默认的评分算法,BM25源于**概率相关模型(probabilistic relevance model)**

虽然听着好像差别巨大，但，两者都使用 词频, 逆文档频率 这些概念，且公式也很相似。**其区别在于如何处理出现频繁的词**

## TF/IDF(词频/逆文档频率)算法

### 铺垫：向量空间模型-Vector Space Model
问题：给定文档d 和 查询q，相似度如何计算呢？

定义V(q) 和 V(d) 是 d和q 分词完成后，利用每个词出现次数所形成的向量

相似度就等于文档d 和 查询q 的 加权查询向量V(q)和V(d)的 **余弦相似度**



## 计算文档得分需要考虑以下因子

- 文档权重（document boost）：**索引期**赋予某个文档的权重值

- 字段权重（field boost）:**查询期**赋予某个字段的权重值

- 词频(trem frequency)：一个基于词项的因子，用来表示一个词项在某个文档中出现多少次，**词频越高，文档得分越高** *比如：查询关键词是A，文档1和文档1都匹配上了，但是文档1中出现了2次A,文档2中出现了1次A,那么在这个项目中，文档1分数更高*

- 逆文档频率(inverse document frequency):一个基于词项的因子,用来告诉评分公式该词项有多么**罕见**，逆文档频率越低，词项越罕见，评分公式利用该因子为包含罕见词项的文档加权
  *比如：查询关键词是A和B,如果文档1命中了A,文档2命中了B,但是在整个文档范围内，A出现的次数比B少，那么在这个项目中，文档1分数更高*

- 协调因子（coord）：基于文档中词项命中个数的协调因子，**一个文档中命中了查询中的词项越多，得分越高**
  *比如：查询关键词被分词为A和B,如果文档1命中了A和B,文档2命中了A,那么在这个项目上，文档1的分数更高*

- 长度范数(length norm):每个字段的基于词项个数的归一化因子，一个字段包含的词项越多，改因子的权重越低，**这意味着lucene评分公式更"喜欢"包含更少词项的字段*
  *比如：查询关键词是A,文档1和2都匹配上了A,但是文档1内容长度比文档1短，那么在这个项目中，文档1分数更高*

- 查询范数（query norm）：一个基于查询的归一化因子，它等于**查询中词项的权重平方和**，**查询范数使得不同查询的得分能相互比较，尽管这种比较通常是困难且不可行的**

## TF/IDF评分公式
**Lucene理论评分公式**
注意，你并不需要深入理解这个公式的来龙去脉，了解它的工作原理非常重要

![image](http://onxkn9cbz.bkt.clouddn.com/lucene12.png)

上面的公式理论形式糅合了布尔检索模型和向量空间检索模型，我们可以不讨论这个理论评分公式，直接跳到lucene实际评分公式
**Lucene实际评分公式**
现在让我来看看Lucene实际评分公式：

![image](http://onxkn9cbz.bkt.clouddn.com/lucene13.png)


解释：这是一个关于**查询q**和**文档d**的函数，有两个因子coord和queryNorm并不直接依赖查询词项，而是与查询词项的一个**求和公式**相乘，**求和公式**中的每个加数由以下因子连乘所得：词频 逆文档频率 词项权重 长度范数

由这个公式我们可以导出一些规则：
- 越多罕见的词项被匹配上，文档分数越高
- 文档字段越短，文档分数越高
- 权重越高（无论是索引期还查询期赋予的权重值），文档得分越高

## BM25算法
BM 是 Best Matching (最佳匹配) 的缩写，它被认为是 当今最先进的排序函数

BM25源于 概率相关模型(probabilistic relevance mode), BM25和TF/IDF实际的打分函数有非常多的相似之处

BM25 同样使用词频、逆向文档频率以及字段长归一化，但是每个因子的定义都有细微区别。与其详细解释 BM25 公式，不如将关注点放在 BM25 所能带来的实际好处上


接下来对比下BM25与TFIDF的区别：

对于词频，BM25 有一个上限，文档里出现 5 到 10 次的词会比那些只出现一两次的对相关度有着显著影响。，文档中出现 20 次的词几乎与那些出现上千次的词有着相同的影响。



## 搜索流程映射到TF/IDF算法公式
![image](http://onxkn9cbz.bkt.clouddn.com/lucene14.png)












