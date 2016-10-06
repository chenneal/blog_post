---
title: Reids源码分析1-惊鸿一瞥
date: 2016-03-12 18:52:29
categories:
- 数据库
- Redis
---

从这篇博文开始，我将正式开始分析`Redis`的源码，代码的版本是`Redis 3.0`。事先得申明一下，我写博文大多数并没有考虑多数人的要求，只是用来梳理自己的思路。所以如果你偶然看到这篇博文，却发现很难理解其内容是一件很正常的事情，当然我会尽量的保证文章的逻辑性以及完整性。再次强调一下，我是先将官方的一些文档看完再分析代码的，所以遇到一些像`sentinel模式`这样的词并不会陌生，所以你如果感到困惑，最好先去看一下文档。

我没有选择网上很多人采用的先阅读与主线无关的部分，比如`Redis`定义的许多数据结构以及特定算法接口代码，而是直接进入主线，等遇到需要的数据结构时再拿出来分析。

说到要理解`Redis`的执行流程，自然会想到`main()`函数，首先我们先看一下main函数的前面一小段代码：

{% codeblock lang:C %}
int main(int argc, char **argv) {
    struct timeval tv;

    /* We need to initialize our libraries, and the server configuration. */
#ifdef INIT_SETPROCTITLE_REPLACEMENT
    // 修改主进程的信息，一般都是指名字。 
    spt_init(argc, argv);
#endif
    // 本地化函数，有什么用？有些字符串函数可以根据当前的本地化信息经行特殊的处理。
    setlocale(LC_COLLATE,"");
    zmalloc_enable_thread_safeness();
    zmalloc_set_oom_handler(redisOutOfMemoryHandler);
    srand(time(NULL)^getpid());
    gettimeofday(&tv,NULL);
    dictSetHashFunctionSeed(tv.tv_sec^tv.tv_usec^getpid());
    server.sentinel_mode = checkForSentinelMode(argc,argv);
    initServerConfig();
    /* We need to init sentinel right now as parsing the configuration file
     * in sentinel mode will have the effect of populating the sentinel
     * data structures with master nodes to monitor. */
    if (server.sentinel_mode) {
        initSentinelConfig();
        initSentinel();
    }
{% endcodeblock %}

<!-- more -->

可喜可贺，由于良好的设计，`main()`函数并不是很长，下面我将按照代码的顺序去讲解这个主函数，十分容易理解的代码将直接跳过。

## spt_init()函数

这个函数我一开始在 `nigix` 的源码里也见到过，只是并不清楚它是干嘛的。后来查了相关资料，才知道这个函数为了修改进程的信息而特定弄的。比如你现在在LINUX环境下，且需要修改主进程的名词，由于主进程的名字就是 main 函数 `argv[0]` 的名称，所以你只需要修改argv指向的内存信息即可。但是这样会带来一个问题，就是因为除了argv数组的信息，每个进程的环境变量 environ 变量也存在那片内存空间里，而且和argv的内存是毗邻的，所以一旦你修改了 `argv[0]` 的内容却溢出了，那么后面的包括 environ 变量的信息也就 say goodbye 了，所以为了发生这种溢出，我们需要将这些信息移至别处再修改进程的名称，这也就是 spt_init 这个函数的作用。

什么？你问没事为何要修改进程的名称，我也不清楚，据说是为了`brand`。Redis将相关的函数接口放在了 setproctitle.c 这个文件里。

## 准备工作

在进入实际工作之前，Redis 在主函数里先做了一些的准备工作。大致步骤如下：

+ 本地化工作，Redis 里的 setlocale() 函数初始化之后，某些字符串函数会根据本地化的信息经行 `字符串` 的一些操作。
+ 开启线程安全模式，所谓线程安全模式，就是要保证线程访问不会出现 `打架` 的情况。
+ 设置内存溢出失败的回调函数，这里所谓的 oom 就是 `out of memory` 的意思，心塞，半天才看懂。顺便说一句，Redis会将内存溢出信息写到日志里。
+ 初始化随机数种子。
+ 获取当前的时间。
+ 设置哈希表的随机种子。
+ 检查当前服务器模式是否处于 sentinel_mode 模式，如果是，下面会根据这个参数作出别的动作。
+ 初始化服务器参数，这个 `initServerConfig` 做了很多初始化服务器配置参数的工作。其中有一些初始化步骤可能会令你费解，比如一些初始化命令操作、以及Redis字典结构的插入，可以先不用管，看下去再说。
+ 检查是否是处于 `sentinel_mode` 模式，如果是，初始化这种模式下的参数配置。实际上，只是改变了端口以及命令列表，此时将原先服务器的命令列表清空并替换成自己的一些命令。

## 读取参数

这段要说的东西不多，先看代码:

{% codeblock lang:C %}
    if (argc >= 2) {
        int j = 1; /* First option to parse in argv[] */
        sds options = sdsempty();
        char *configfile = NULL;

        /* Handle special options --help and --version */
        if (strcmp(argv[1], "-v") == 0 ||
            strcmp(argv[1], "--version") == 0) version();
        if (strcmp(argv[1], "--help") == 0 ||
            strcmp(argv[1], "-h") == 0) usage();
        if (strcmp(argv[1], "--test-memory") == 0) {
            if (argc == 3) {
                // 检测内存
                memtest(atoi(argv[2]),50);
                exit(0);
            } else {
                fprintf(stderr,"Please specify the amount of memory to test in megabytes.\n");
                fprintf(stderr,"Example: ./redis-server --test-memory 4096\n\n");
                exit(1);
            }
        }

        /* First argument is the config file name? */
        if (argv[j][0] != '-' || argv[j][1] != '-')
            configfile = argv[j++];
        /* All the other options are parsed and conceptually appended to the
         * configuration file. For instance --port 6380 will generate the
         * string "port 6380\n" to be parsed after the actual file name
         * is parsed, if any. */
        while(j != argc) {
            if (argv[j][0] == '-' && argv[j][1] == '-') {
                /* Option name */
                if (sdslen(options)) options = sdscat(options,"\n");
                options = sdscat(options,argv[j]+2);
                options = sdscat(options," ");
            } else {
                /* Option argument */
                options = sdscatrepr(options,argv[j],strlen(argv[j]));
                options = sdscat(options," ");
            }
            j++;
        }
        if (server.sentinel_mode && configfile && *configfile == '-') {
            redisLog(REDIS_WARNING,
                "Sentinel config from STDIN not allowed.");
            redisLog(REDIS_WARNING,
                "Sentinel needs config file on disk to save state.  Exiting...");
            exit(1);
        }
        if (configfile) server.configfile = getAbsolutePath(configfile);
        resetServerSaveParams();
        loadServerConfig(configfile,options);
        sdsfree(options);
    } else {
        redisLog(REDIS_WARNING, "Warning: no config file specified, using the default config. In order to specify a config file use %s /path/to/%s.conf", argv[0], server.sentinel_mode ? "sentinel" : "redis");
    }
{% endcodeblock %}

这段代码很容易理解，无非就是解析从控制台转进来的命令参数，其中比较有意思的就是Redis提供了一条内存检测的命令，这倒是在很多装机盘里面看过 :-)

## 初始化服务器

{% codeblock lang:C %}
// 将Redis作为守护进程使用
if (server.daemonize) daemonize();
    initServer();
    if (server.daemonize) createPidFile();
    redisSetProcTitle(argv[0]);
    redisAsciiArt();
    // 检查侦听事件数量的系统参数
    checkTcpBacklogSettings();

    if (!server.sentinel_mode) {
        /* Things not needed when running in Sentinel mode. */
        redisLog(REDIS_WARNING,"Server started, Redis version " REDIS_VERSION);
    #ifdef __linux__
        linuxMemoryWarnings();
    #endif
    // 从磁盘加载数据，这里的数据基本上都是用来做持久化的。
        loadDataFromDisk();
        if (server.cluster_enabled) {
            if (verifyClusterConfigWithData() == REDIS_ERR) {
                redisLog(REDIS_WARNING,
                    "You can't have keys in a DB different than DB 0 when in "
                    "Cluster mode. Exiting.");
                exit(1);
            }
        }
        if (server.ipfd_count > 0)
            redisLog(REDIS_NOTICE,"The server is now ready to accept connections on port %d", server.port);
        if (server.sofd > 0)
            redisLog(REDIS_NOTICE,"The server is now ready to accept connections at %s", server.unixsocket);
    } else {
    // sentinel模式的不同处理方式
        sentinelIsRunning();
    }

    /* Warning the user about suspicious maxmemory setting. */
    //内存至少1MB
    if (server.maxmemory > 0 && server.maxmemory < 1024*1024) {
        redisLog(REDIS_WARNING,"WARNING: You specified a maxmemory value that is less than 1MB (current value is %llu bytes). Are you sure this is what you really want?", server.maxmemory);
    }
{% endcodeblock %}

`daemonize()` 函数将 Redis 的主进程作为守护进程来使用，守护进程的内容参见 APUE 13章内容，简单的说就是使父进程退出二将 fork 出来的子进程放在后台运行的一种技术。

这部分代码的重点就是 `initServer()` 函数，这个函数做了很多服务器初始化的工作，而且有些工作比较复杂，如果能理解这部分代码，对后面理解 Redis 的工作原理极有帮助，然而篇幅有限，我可能会在后面才重点展开讲，这里只粗略的提一下它做了哪些工作:

+ 为许多隶属于服务器的结构分配了内存空间，大部分结构都使用了字典结构 `dict` 以及双向链表 `list` 。这里的内存分配其实是极小的，大多数数据的分配都是在以后动态分配的。其中 `dict` 字典结构有点复杂，后面会专门开辟一篇分析这个结构。
+ 初始化了一些全局结构，比如 `createSharedObjects()` 初始化一些全局对象， `replicationScriptCacheInit()` 初始化了脚本的缓存，很多地方我还不是特别清楚其用意，希望后面弄明白。
+ 为后续的网络连接做了初步工作，使用TCP连接，如果你熟悉网络编程的话，这里做的工作相当于 `bind` 和 `listen` 两个阶段，顺便提一下，除了 Redis 额外建立了一个本地连接，好像是为了作为本地 benchmark 测试使用的，除了 ipv4 协议，还增加了 ipv6 协议，至于 `accept` 阶段肯定是在后面了。
+ 初始化了两个事件结构，一个是 `aeFileEvent` `aeTimeEvent` ，由于Redis使用事件驱动模型，所以你要是对这个两个事件不熟悉的话，先看官方文档吧。
+ 获取了一些系统时间（UNIX时间戳），并将部分存到了一个时间缓存里，据说是为了速度，以后直接从缓存里拿，不需要重新获取。
+ 设置了很多回调函数。
+ 初始化了集群模式的一些结构，集群部分比较复杂，会开章单讲。

## 工作函数（重中之重）

那么重点来了，Redis 实际上是如何工作的呢？我们看到下面的代码就清楚了，它使用了一个循环不断地工作直到产生退出条件才退出。

{% codeblock lang:C %}
aeSetBeforeSleepProc(server.el,beforeSleep);
// 工作主函数
aeMain(server.el);
aeDeleteEventLoop(server.el);
return 0;
{% endcodeblock %}

主要看 `aeMian()`

{% codeblock lang:C %}
void aeMain(aeEventLoop *eventLoop) {
    eventLoop->stop = 0;
    while (!eventLoop->stop) {
        if (eventLoop->beforesleep != NULL)
            eventLoop->beforesleep(eventLoop);
        aeProcessEvents(eventLoop, AE_ALL_EVENTS);
    }
}
{% endcodeblock %}

`aeEventLoop` 是一个事件驱动的结构体，只要其 stop 成员不为真，这个主循环就会一直继续。`aeProcessEvents()` 函数处理每次循环的工作，每次在调用这个函数之前会事先调用 `beforesleep()` 函数，这是个回调函数，在初始化的时候已将它初始化到特定的函数。这个函数为处理一下上次循环留下来的残留事务，也就是所谓的 `善后的`。下面我们看 `aeProcessEvents()` 这个函数

{% codeblock lang:C %}
int aeProcessEvents(aeEventLoop *eventLoop, int flags)
{
    int processed = 0, numevents;

    /* Nothing to do? return ASAP */
    if (!(flags & AE_TIME_EVENTS) && !(flags & AE_FILE_EVENTS)) return 0;

    /* Note that we want call select() even if there are no
     * file events to process as long as we want to process time
     * events, in order to sleep until the next time event is ready
     * to fire. */
    if (eventLoop->maxfd != -1 ||
        ((flags & AE_TIME_EVENTS) && !(flags & AE_DONT_WAIT))) {
        int j;
        aeTimeEvent *shortest = NULL;
        struct timeval tv, *tvp;

        if (flags & AE_TIME_EVENTS && !(flags & AE_DONT_WAIT))
            shortest = aeSearchNearestTimer(eventLoop);
        if (shortest) {
            long now_sec, now_ms;

            /* Calculate the time missing for the nearest
             * timer to fire. */
            aeGetTime(&now_sec, &now_ms);
            tvp = &tv;
            tvp->tv_sec = shortest->when_sec - now_sec;
            if (shortest->when_ms < now_ms) {
                tvp->tv_usec = ((shortest->when_ms+1000) - now_ms)*1000;
                tvp->tv_sec --;
            } else {
                tvp->tv_usec = (shortest->when_ms - now_ms)*1000;
            }
            if (tvp->tv_sec < 0) tvp->tv_sec = 0;
            if (tvp->tv_usec < 0) tvp->tv_usec = 0;
        } else {
            /* If we have to check for events but need to return
             * ASAP because of AE_DONT_WAIT we need to set the timeout
             * to zero */
            if (flags & AE_DONT_WAIT) {
                tv.tv_sec = tv.tv_usec = 0;
                tvp = &tv;
            } else {
                /* Otherwise we can block */
                tvp = NULL; /* wait forever */
            }
        }

        // linux下建立epoll模型
        numevents = aeApiPoll(eventLoop, tvp);
        // 优先处理file event
        for (j = 0; j < numevents; j++) {
            aeFileEvent *fe = &eventLoop->events[eventLoop->fired[j].fd];
            int mask = eventLoop->fired[j].mask;
            int fd = eventLoop->fired[j].fd;
            int rfired = 0;

        /* note the fe->mask & mask & ... code: maybe an already processed
             * event removed an element that fired and we still didn't
             * processed, so we check if the event is still valid. */
            if (fe->mask & mask & AE_READABLE) {
                rfired = 1;
                fe->rfileProc(eventLoop,fd,fe->clientData,mask);
            }
            if (fe->mask & mask & AE_WRITABLE) {
                if (!rfired || fe->wfileProc != fe->rfileProc)
                    fe->wfileProc(eventLoop,fd,fe->clientData,mask);
            }
            processed++;
        }
    }
    /* Check time events */
    // 后续处理 time event
    if (flags & AE_TIME_EVENTS)
        processed += processTimeEvents(eventLoop);

    return processed; /* return the number of processed file/time events */
}
{% endcodeblock %}

这段代码可能你还觉得难以理解，没办法，毕竟知道的信息还太少。其实上述的代码猜也能猜到，就是一个不断侦听客户端的请求并不断处理请求的过程，而且对两个不同的事件会采取不同的方法和优先级。

好啦，第一篇就到这了。虽然基本上还处于迷迷糊糊的状态，起码稍稍把握了 Redis 的总体处理事务的流程。接下来可能会去分析 `dict` 这个复杂的字典结构，或者继续剖析 Redis 处理客户端命令的流程，谁知道呢，随性而为吧 :-) 

8~~