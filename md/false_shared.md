# 伪共享避免

编写并行应用程序时最重要的考虑因素之一是不同线程或进程如何共享数据。在某些情况下，共享是不可避免的。但是，我们的数据布局和架构可能会引入无意的共享，称为伪共享。本文通过一些简单的基准测试和分析提供了对伪共享的基本介绍。

## 什么是伪共享

在计算机科学中，伪共享是一种降低性能的使用模式，它可能出现在具有分布式、连续缓存的系统中，缓存大小为缓存机制管理的最小资源块。当系统使用者尝试定期访问未被另一方更改的数据，但该数据与正在更改的数据共享缓存块时，缓存协议可能会强制第一个参与者重新加载整个缓存块，尽管缺少逻辑上的必要性。 缓存系统不知道此块内的活动，并强制第一个参与者承担资源的真正共享访问所需的缓存系统开销。

简单一点:缓存系统中是以缓存行(cache line)为单位存储的，当多线程修改互相独立的变量时，如果这些变量共享同一个缓存行，就会无意中影响彼此的性能，这就是伪共享。

下面直接看图为什么会产生伪共享:

在文章开头提到过，缓存系统中是以缓存行(cache line)为单位存储的。缓存行通常是64字节(命令`cat /proc/cpuinfo`可查看)，并且它有效地引用主内存中的一块地址。一个c++的int类型是 4 字节，因此在一个缓存行中可以存 16 个 int 类型的变量。所以，如果你访问一个 int 数组，当数组中的一个值被加载到缓存中，它会额外加载另外 15 个，以致你能非常快地遍历这个数组。事实上，你可以非常快速的遍历在连续的内存块中分配的任意数据结构。而如果你在数据结构中的项在内存中不是彼此相邻的（如链表），你将得不到缓存加载所带来的优势，并且在这些数据结构中的每一个项都可能会出现缓存未命中。

如果存在这样的场景，有多个线程操作不同的成员变量，但是相同的缓存行，这个时候会发生什么？。没错，伪共享（False Sharing）问题就发生了！有张 Disruptor 项目的经典示例图，如下：

![](../resource/false_sharing.png)

上图中，一个运行在处理器core1上的线程想要更新变量 X 的值，同时另外一个运行在处理器 core2 上的线程想要更新变量 Y 的值。但是，这两个频繁改动的变量都处于同一条缓存行。两个线程就会轮番发送 RFO 消息，占得此缓存行的拥有权。当 core1 取得了拥有权开始更新 X，则 core2 对应的缓存行需要设为 I 状态。当 core2 取得了拥有权开始更新 Y，则 core1 对应的缓存行需要设为 I 状态(失效态)。轮番夺取拥有权不但带来大量的 RFO 消息，而且如果某个线程需要读此行数据时，L1 和 L2 缓存上都是失效数据，只有 L3 缓存上是同步好的数据。读 L3 的数据非常影响性能。更坏的情况是跨槽读取，此时L3都要发生`cache miss`，只能从内存上加载。

表面上 X 和 Y 都是被独立线程操作的，而且两操作之间也没有任何关系。只不过它们共享了一个缓存行，但所有竞争冲突都是来源于共享。

下面是C语言示例代码：

第一个结构体`false_share`两个变量都是int型，显然声明一个这个类型的结构体其两个变量都放在一个cache line中。
而第二个结构体`real_share`两个int型变量中间增加了一个64字节的字段pack，这样保证`real_share`的两个int型变量`a`和`b`在两个cache line中，这样多线程操作时不会发生伪共享。下面运行程序，可以看到多线程环境下对`real_share`的加法操作耗时会比`false_share`短非常多。

下面的完整代码：

```c
#include <sys/times.h>
#include <pthread.h>
#include <stdio.h>
#include <time.h>

struct timespec begin1, end1, begin2, end2, begin3, end3, begin4, end4;

double compute_time_diff(struct timespec start, struct timespec end)
{
    double t;
    t = (end.tv_sec - start.tv_sec) * 1000;
    t += (end.tv_nsec - start.tv_nsec) * 0.000001;
    return t;
}

typedef struct false_share_
{
    int a;
    int b;
}false_share;

typedef struct real_share_
{
    int a;
    char pack[64];
    int b;
}real_share;

void *expensive_false_fun(void *param)
{
    false_share temp = *(false_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.a += 1;
        temp.b += 1;
    }
}

void *expensive_false_fun_add_a(void *param)
{
    false_share temp = *(false_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.a += 1;
    }
}

void *expensive_false_fun_add_b(void *param)
{
    false_share temp = *(false_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.b += 1;
    }
}

void *expensive_real_fun(void *param)
{
    real_share temp = *(real_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.a += 1;
        temp.b += 1;
    }
}

void *expensive_real_fun_add_a(void *param)
{
    real_share temp = *(real_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.a += 1;
    }
}

void *expensive_real_fun_add_b(void *param)
{
    real_share temp = *(real_share *)param;
    int i;
    for (i = 0; i < 1000000000; i++) {
        temp.b += 1;
    }
}

int main()
{
    false_share false_elem = {0};
    real_share real_elem = {0};

    double time1;
    double time2;
    double time3;
    double time4;

    pthread_t thread1;
    pthread_t thread2;

    // 串行
    // 1. 伪共享情况下的耗时统计 
    clock_gettime(CLOCK_REALTIME, &begin1);
    expensive_false_fun((void *)&false_elem);
    clock_gettime(CLOCK_REALTIME, &end1);
    // 2. 变量不在同一cache下的耗时统计,串行情况应该一致的
    clock_gettime(CLOCK_REALTIME, &begin2);
    expensive_real_fun((void *)&real_elem);
    clock_gettime(CLOCK_REALTIME, &end2);


    // 并行
    // 1. 伪共享
    clock_gettime(CLOCK_REALTIME, &begin3);
    pthread_create(&thread1, NULL, expensive_false_fun_add_a, (void *)&false_elem);
    pthread_create(&thread2, NULL, expensive_false_fun_add_b, (void *)&false_elem);
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    clock_gettime(CLOCK_REALTIME, &end3);
    // 2. 变量在不同cache line
    clock_gettime(CLOCK_REALTIME, &begin4);
    pthread_create(&thread1, NULL, expensive_real_fun_add_a, (void *)&real_elem);
    pthread_create(&thread2, NULL, expensive_real_fun_add_b, (void *)&real_elem);
    pthread_join(thread1, NULL);
    pthread_join(thread2, NULL);
    clock_gettime(CLOCK_REALTIME, &end4);

    time1 = compute_time_diff(begin1, end1);
    time2 = compute_time_diff(begin2, end2);
    time3 = compute_time_diff(begin3, end3);
    time4 = compute_time_diff(begin4, end4);

    printf("Time take with false sharing      : %f ms\n", time3);
    printf("Time taken without false sharing  : %f ms\n", time4);
    printf("Time taken in Sequential computing with false sharing : %f ms\n", time1);
    printf("Time taken in Sequential computing without false sharing: %f ms\n", time2);

}
```

## 结论

通过使用pack填充强制变量分布到不同cache line即可解决伪共享的问题。