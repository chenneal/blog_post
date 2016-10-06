---
title: Redis源码阅读(0)
subtitle: nimei
date: 2016-03-11 21:18:38
categories:
- 数据库
- Redis
---

虽然这是我写的第一篇关于 **Redis** 的文章，但其实我已经入坑有几天了，之前一直想完整地阅读一个开源的、工业级别的代码项目，奈何自己底子不够，一直搁置下来。趁着自己刚刚完成一篇论文的实验，想趁热打铁完成这件事。这之后我就开始物色对象，很快就决定阅读 Redis 的代码，原因有很多，一是我对 C 语言和 Unix 环境下的网络编程相对熟悉；二是Redis在工业界的使用率相当的高；三是由于 Redis 的代码简短精悍，功能却十分强大，执行效率也相当的高。这篇 `入坑文` 我先说说我对阅读开源项目前的一些看法。
<!-- more -->

## 疑惑

首先是关于如何从哪里开始下手？这个问题困扰了我好久。一开始我是秉着初生牛犊不怕虎的精神直接拿来源码就放到 `Source Insight` 里看，结果如你所料，大概看了一两个文件之后就云里雾里，果然是行不通 o(*////▽////*)q 后来有人推荐了我一些材料，主要是博客和相关的书籍，后来发现博客讲的相对透彻一些，决定边看博客边阅读代码，只能说确实比之前好了一些，然而读了一部分代码还是感觉有些模糊，看得懂语法，但却看不懂作者的意图。

## 思考

后来连番被虐之后我开始痛定思痛，我突然意识到一个问题，你去读人家的代码，你知道它是用来干什么的吗，你知道它多少特性呢？o((⊙﹏⊙))o.工业级别的代码，不可能说只有代码的语法这么简单，还有它指明的各种语义以及各个模块之间的协调。比如下面的结构体，就算你知道他是一个 `struct` 却不知道它具体的用途及目的，也没有什么卵用：

{% codeblock lang:c %}
/* State of an event based program */
typedef struct aeEventLoop {
    int maxfd;   /* highest file descriptor currently registered */
    int setsize; /* max number of file descriptors tracked */
    long long timeEventNextId;
    time_t lastTime;     /* Used to detect system clock skew */
    aeFileEvent *events; /* Registered events */
    aeFiredEvent *fired; /* Fired events */
    aeTimeEvent *timeEventHead;
    int stop;
    void *apidata; /* This is used for polling API specific data */
    aeBeforeSleepProc *beforesleep;
} aeEventLoop;
{% endcodeblock %}

## 方法

那么之后要做的事情就很明显了，当然是去搞清楚 Redis 的工作机制原理以及各个模块的功能。提到这个，首先想到的自然是去看 Redis 官方的文档。将 Redis 的文档大致的捋了一遍后，确实有了不少的提升，看代码也像是开了 `上帝视角` ,即使有的代码暂时看不明白，也能大致猜出它的用途，从而可以先跳过它更加从宏观的角度去浏览源码。

## 教训及总结
{% blockquote %}
+ 看一个工业级别的代码项目，最好不要一拿到源码就立马着手阅读。
+ 优秀的博客和书籍是捷径，但是绝不是万能的。
+ 阅读官方的文档可以从宏观上掌握项目的结构和功能，极大的提升源码阅读的效率。
{% endblockquote %}
后续将正式开始源码分析的实际工作，敬请期待。