## aio详解

### 介绍

Linux的AIO(Asynchronous Input/Output)接口运行并行提交许多个I/O请求，而无需为每个I/O请求增加一个线程的开销。换句话说可以异步的进行I/O操作。

本文主要讲如何使用Linux AIO接口，即函数`io_setup`，`io_submit`，`io_getevents`，`io_destroy`。这里要提到aio的一个缺陷，aio只支持`O_DIRECT`方式来读写文件。

`O_DIRECT`指直接I/O(Direct I/O)。无论是`buffered I/O`和`mmap`，其访问文件都是需要过内核的`page cache`。有的说法是要过缓冲区高速缓存。而直接I/O从用户空间将数据直接传递到文件或磁盘设备。

对于数据库系统这种`self-caching`自己实现缓存的系统来说，绕过内核直接I/O会有较大提升。

## AIO是什么

Linux AIO说白了就是将异步磁盘I/O公开给用户空间的程序。

在Linux上，所有磁盘操作都是阻塞的。无论执行的是 `open()`、`read()`、`write()`还是`fsync()`，如果所需的数据和元数据未在磁盘缓存中准备好，其实所需这个数据的线程将被操作系统停止(挂起)。这个逻辑没有什么问题。如果该线程只执行少量IO或有足够的内存，磁盘系统调用将逐渐填充满缓存(内存空间)并且平均速度相当快。

上面实际上即为同步IO的逻辑。

下面是同步模型和异步模型的区别。

- 同步模型(synchronous I/O model)中，应用程序从线程发出请求。线程阻塞直到操作完成。操作系统会给程序员或者程序一种错觉，即向设备发出请求并接收到结果就像任何其他仅在CPU上进行的操作一样。(但实际上，操作系统会切换到其他线程或进程以利用CPU资源并允许同一CPU并行向设备发出其他设备请求)
- 在异步`I/O`(asynchronous I/O，AIO)模型中，应用程序可以从一个线程提交一个或多个请求。提交请求不会导致线程阻塞，相反，线程可以继续进行其他计算，并在原始请求运行期间向设备提交更多请求。应用程序应自行处理完成和组织逻辑计算，而不依赖于线程来组织数据的使用。

异步`I/O`可以被认为是比同步I/O“更低级别”，因为它不使用系统提供的线程概念来组织其计算。由于线程的开销具有不确定性(在)，所以使用AIO通常比使用同步`I/O`更有效。

IO密集型程序(如数据库或缓存或http缓存代理服务器)往往使用AIO。在这样的应用程序中，如果整个服务器停止运行将是悲剧性的，而这仅仅是因为一些`read()`系统调用必须等待磁盘而操作系统就挂起了对应的工作线程。

要解决此问题，应用程序使用以下三种方法之一：

1. 使用线程池并将阻塞系统调用卸载到工作线程。这就是 glibc POSIX AIO(不要与 Linux AIO 混淆)包装器所做的。
2. 使用`posix_fadvise(2)`预取磁盘缓存并祈祷一切顺利。
3. 将Linux AIO 与 XFS文件系统(非必须)、使用 O_DIRECT 打开的文件一起使用，并避免未记录的陷阱(undocumented pitfalls)。

这些方法都不是完美的。即使是 Linux AIO，如果使用不当，也可能会阻塞 io_submit() 调用。

!!!
    Linux 异步 I/O (AIO) 层往往有很多批评者和很少的捍卫者，但大多数人至少期望它实际上是异步的。事实上，由于多种原因，AIO 操作可能会在内核中阻塞，这使得AIO难以在调用线程确实无法承受阻塞的情况下使用。

现在知道了 Linux AIO API 的不足之处，让我们看看它的亮点在哪里。

## Linux原生AIO的api(实现文件读写)

Linux不提供原生的aio，需要通过syscall来封装。

函数原型如下:

```c
int io_setup(unsigned nr_events, aio_context_t *ctx_idp);
int io_submit(aio_context_t ctx_id, long nr, struct iocb **iocbpp);
int io_getevents(aio_context_t ctx_id, long min_nr, long nr,
                 struct io_event *events, struct timespec *timeout);
int io_cancel(aio_context_t ctx_id, struct iocb *iocb,
                     struct io_event *result);
int io_destroy(aio_context_t ctx_id);
```

实现如下:

```c
inline static int io_setup(unsigned nr, aio_context_t *ctxp)
{
	return syscall(__NR_io_setup, nr, ctxp);
}

inline static int io_destroy(aio_context_t ctx)
{
	return syscall(__NR_io_destroy, ctx);
}

inline static int io_submit(aio_context_t ctx, long nr, struct iocb **iocbpp)
{
	return syscall(__NR_io_submit, ctx, nr, iocbpp);
}

inline static int io_getevents(aio_context_t ctx, long min_nr, long max_nr,
			       struct io_event *events,
			       struct timespec *timeout)
{
	// This might be improved.
	return syscall(__NR_io_getevents, ctx, min_nr, max_nr, events, timeout);
}
```

1. io_setup 创建一个能支持 nr_events 个操作的异步 I/O context。
2. io_submit 提交异步 I/O 请求。
3. io_getevents 获取已完成的异步 I/O 结果。
4. io_cancel 取消之前提交的某个异步 I/O 请求。
5. io_destroy 取消所有提交的异步 I/O 请求，并销毁异步 I/O context。

### AIO简单程序

编写AIO程序仅需要3步

1. 调用`io_setup()`来设置`aio_context`数据结构。内核会返回一个不透明的指针
2. 然后可以调用`io_submit()`提交一个"I/O控制块"(`struct iocb`)的结构进行处理。提交I/O请求。
3. 最后处理I/O调用请求的结果。调用`io_getevents()`来阻塞并等待结构 `io_event`结构体返回。 - 获取iocb的完成通知(iocb完成I/O请求后得到的结果)

AIO的系统调用发生后发生了什么主要如下图所示:

可以看到`I/O`请求执行完成，通过DMA直接将数据传输到user buffer。

下面是aio的一些细节：

### iocb支持的命令

一个 `iocb` 中可以提交 8 种命令。 4.18 内核中引入的两种不同的读取、写入、fsync和一种POLL命令：


```c
IOCB_CMD_PREAD = 0,
IOCB_CMD_PWRITE = 1,
IOCB_CMD_FSYNC = 2,
IOCB_CMD_FDSYNC = 3,
IOCB_CMD_POLL = 5,   /* from 4.18 */
IOCB_CMD_NOOP = 6,
IOCB_CMD_PREADV = 7,
IOCB_CMD_PWRITEV = 8,
```

### iocb结构

iocb完整的结构很大，可以在这里查阅 [iocb](https://github.com/torvalds/linux/blob/f346b0becb1bc62e45495f9cdbae3eef35d0b635/include/uapi/linux/aio_abi.h#L73-L107)

需要用户填充的核心字段其实为如下几个:

```c
struct iocb {
  __u64 data;           /* user data */
  ...
  __u16 aio_lio_opcode; /* see IOCB_CMD_ above */
  ...
  __u32 aio_fildes;     /* file descriptor */
  __u64 aio_buf;        /* pointer to buffer */
  __u64 aio_nbytes;     /* buffer size */
...
}
```

### io_getevents结构

io_getevents结构是为了获取对应的IO提交返回结果的。

实现如下:

```c
struct io_event {
  __u64  data;  /* user data */
  __u64  obj;   /* pointer to request iocb */
  __s64  res;   /* result code for this event */
  __s64  res2;  /* secondary result */
};
```

下面是一个aio的例子，读取`/etc/passwd`文件种的数据

核心代码如下

```c
fd = open("/etc/passwd", O_RDONLY);

aio_context_t ctx = 0;
r = io_setup(128, &ctx);

char buf[4096];
struct iocb cb = {.aio_fildes = fd,
                  .aio_lio_opcode = IOCB_CMD_PREAD,
                  .aio_buf = (uint64_t)buf,
                  .aio_nbytes = sizeof(buf)};
struct iocb *list_of_iocb[1] = {&cb};

r = io_submit(ctx, 1, list_of_iocb);

struct io_event events[1] = {{0}};
r = io_getevents(ctx, 1, 1, events, NULL);

bytes_read = events[0].res;
printf("read %lld bytes from /etc/passwd\n", bytes_read);
```

完整代码 [on github](https://github.com/helintongh/CWheel/commit/c9c8a8bd404e35341369c0875fa4dc5c0a865ade#diff-45e154fddbb523f2ca3999a0fe8bcc284b34cdeafd628dec87a05cbbe3764811)

而想要通过aio写入buf到文件代码类似同样完整代码在 [on github](https://github.com/helintongh/CWheel/commit/c9c8a8bd404e35341369c0875fa4dc5c0a865ade#diff-7d88865142e58c39dc00f5f91d3f9d0bfee82f0452fc5d3bb5eeedcd9f47ff8d)

用strace跟踪程序运行得到如下:

```
io_submit(0x7fe471dbe000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREAD, aio_fildes=3, aio_buf=0x7ffe2368d940, aio_nbytes=4096, aio_offset=0}]) = 1
io_getevents(0x7fe471dbe000, 1, 1, [{data=0, obj=0x7ffe2368d8d0, res=1892, res2=0}], NULL) = 1
```

上述代码可以很完美的运行，但是其实并不是异步的。`io_submit`函数执行时其实会阻塞住此时完成所有的IO操作。

而返回I/O提交的`io_getevents`调用则是立即完成。我们可以尝试使磁盘读取异步，但这需要跳过缓存的 O_DIRECT 标志。`

## AIO批量提交socket

`io_submit`的实现相当保守。除非传递的描述符是`O_DIRECT`文件，否则它会阻塞然后执行I/O请求。在网络套接字的情况下，它意味着：

- 对于阻塞套接字，`IOCB_CMD_PREAD`将挂起，直到数据包到达。
-  对于非阻塞套接字，`IOCB_CMD_PREAD` 将返回-11(EAGAIN)。

从语义来讲aio和系统调用`read()`完全相同。对于套接字`io_submit`和读写系统调用差不了太多。

要注意传递给内核的`iocb`请求是按顺序的。

虽然 Linux AIO 对异步可能操作没有帮助，但它绝对可以用于系统调用批处理。

如果有一个 Web 服务器需要从数百个网络套接字发送和接收数据，那么使用 `io_submit` 可能是个好主意。这将避免调用 `send` 和 `recv` 数百次。这将提高性能 -- 从用户态陷入内核态的开销是昂贵的。

下面是有关I/O读取操作的一个列表:

|                | 一条文件读取流              |  多条文件读取流                |
|  :----:        | :----:                     | :----:                        |
| 一个文件描述符  | read()                     | readv()                       |
| 多个文件描述符  | io_submit + IOCB_CMD_PREAD |   io_submit + IOCB_CMD_PREADV |

写操作类似，多个流和单个流对应的函数是差不多的。AIO其实只是提供了针对多个文件描述符的相关功能。

为了说明 `io_submit` 的批处理方面作用，我创建一个小程序，将数据从一个 TCP 套接字转发到另一个。在最简单的形式中，如果没有 Linux AIO，程序将有一个像这样的简单流程：

```c
while True:
  d = sd1.read(4096)
  sd2.write(d)
```

可以用Linux AIO表达同样的逻辑。代码将如下所示：

```c
struct iocb cb[2] = {{.aio_fildes = sd2,
                      .aio_lio_opcode = IOCB_CMD_PWRITE,
                      .aio_buf = (uint64_t)&buf[0],
                      .aio_nbytes = 0},
                     {.aio_fildes = sd1,
                     .aio_lio_opcode = IOCB_CMD_PREAD,
                     .aio_buf = (uint64_t)&buf[0],
                     .aio_nbytes = BUF_SZ}};
struct iocb *list_of_iocb[2] = {&cb[0], &cb[1]};
while(1) {
  r = io_submit(ctx, 2, list_of_iocb);

  struct io_event events[2] = {};
  r = io_getevents(ctx, 2, 2, events, NULL);
  cb[0].aio_nbytes = events[1].res;
}
```

此代码将两个文件描述符提交到`iocb`数组中。当提交I/O请求时(`io_submit`调用时)。首先请求向sd2写入数据，然后从sd1读取数据。读取完成后，代码就能够确定写入缓冲区的大小然后再次循环。该代码利用了一个技巧---第一次写入的大小为 0。这样做是因为我们可以在一个 `io_submit` 中融合写入+读取（但不是读取+写入）。读取完成后，我们必须修改写入缓冲区的大小。

这段代码比简单的读/写版本更快吗？还没有。两个版本都有两个系统调用：`read`+`write` 和 `io_submit`+`io_getevents`。幸运的是，我们可以改进它。

### 优化1:无需系统调用,用户态实现io_getevents

运行`io_setup()`时，内核会为进程分配几页内存。这是这个内存块在`/proc/<pid>/maps`中的样子：

```
...
7f7db8f60000-7f7db8f63000 rw-s 00000000 00:12 2314562     /[aio] (deleted)
...
```

[aio程序]的内存区域（在我的例子中是 12KB）由 `io_setup` 分配。此内存范围用于存储完成事件后对应信息的环形缓冲区。在大多数情况下，没有任何理由调用真正的 `io_getevents` 系统调用。无需进行内核调用即可轻松从环形缓冲区中检索完成数据。这是代码的固定版本：

```c
int io_getevents(aio_context_t ctx, long min_nr, long max_nr,
                 struct io_event *events, struct timespec *timeout)
{
    int i = 0;

    struct aio_ring *ring = (struct aio_ring*)ctx;
    if (ring == NULL || ring->magic != AIO_RING_MAGIC) {
        goto do_syscall;
    }

    while (i < max_nr) {
        unsigned head = ring->head;
        if (head == ring->tail) {
            /* There are no more completions */
            break;
        } else {
            /* There is another completion to reap */
            events[i] = ring->events[head];
            read_barrier();
            ring->head = (head + 1) % ring->nr;
            i++;
        }
    }

    if (i == 0 && timeout != NULL && timeout->tv_sec == 0 && timeout->tv_nsec == 0) {
        /* Requested non blocking operation. */
        return 0;
    }

    if (i && i >= min_nr) {
        return i;
    }

do_syscall:
    return syscall(__NR_io_getevents, ctx, min_nr-i, max_nr-i, &events[i], timeout);
}

```

通过此代码重写`io_getevents` 函数，TCP 代理程序的Linux AIO版本每个循环只需要一个系统调用，这样就会比普通读写代码快一点点。

以上代码是摘录的fio中的，原代码段 [code](https://github.com/axboe/fio/blob/702906e9e3e03e9836421d5e5b5eaae3cd99d398/engines/libaio.c#L149-L172)

### 优化2:替代epoll

在内核4.18中添加 `IOCB_CMD_POLL`，可以将 `io_submit` 用作 `select`/`poll`/`epoll`的等价物。例如，下面的代码实现了epoll的功能，不断轮询sd文件描述符，有数据时就执行`io_submit`。

```c
struct iocb cb = {.aio_fildes = sd,
                  .aio_lio_opcode = IOCB_CMD_POLL,
                  .aio_buf = POLLIN};
struct iocb *list_of_iocb[1] = {&cb};

r = io_submit(ctx, 1, list_of_iocb);
r = io_getevents(ctx, 1, 1, events, NULL);
```

完整代码 [full code](https://github.com/helintongh/CWheel/blob/master/aio_example/aio_poll.c)

下面是strace程序的函数示踪。

```
io_setup(128, [0x7f9a9752d000])         = 0
io_submit(0x7f9a9752d000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_POLL, aio_fildes=3, aio_buf=POLLIN}]) = 1
io_getevents(0x7f9a9752d000, 1, 1, [{data=0, obj=0x7fff88d0d570, res=1, res2=0}], NULL) = 1
```

正如代码展示的那样，"异步"部分工作正常，`io_submit`立即完成，获取结果的 `io_getevents`在等待数据时阻塞了1s。这非常强大，`io_getevents`实质上代替了`epoll_wait()`系统调用。

此外，通常处理 `epoll` 需要处理 `epoll_ctl` 系统调用。应用程序开发人员竭尽全力避免过于频繁地调用此系统调用。使用 `io_submit` 进行轮询可以解决epoll的复杂性，并且不需要任何的系统调用。只需将套接字推送到 `iocb` 请求数组里即可，然后调用 `io_submit` 一次并等待完成就完成了`epoll`的替换。

## 总结

在这篇博文中，回顾了 Linux AIO API。虽然最初被认为是一个仅限磁盘的 API，但它似乎以与网络套接字上的正常读/写系统调用相同的方式工作。与读/写不同的是，`io_submit` 允许系统调用批处理，可能会提高性能。

从内核 4.18 开始，`io_submit` 和 `io_getevents` 可用于等待网络套接字上的 `POLLIN` 和 `POLLOUT` 等事件。这很棒，可以用作事件循环中 `epoll()` 的替代品。

想象一个网络服务器可以只执行 `io_submit` 和 `io_getevents` 系统调用，而不是通常的读、写、`epoll_ctl` 和 `epoll_wait` 组合。通过这样的设计，`io_submit` 的系统调用批处理方面可以真正发挥作用。这样的服务器会更快。


Linus 评价AIO接口设计是: "But AIO was always really really ugly."

然而，实质上多年来，Linux内核曾多次尝试创建更好的批处理和异步接口，不幸的是，缺乏连贯的愿景。例如，近几年添加的 `sendto`(MSG_ZEROCOPY) 允许真正的异步传输操作，但没有批处理。 `io_submit` 允许批处理但不允许异步。

更糟糕的是 --- Linux 目前有三种传递异步通知的方式 --- 信号、`io_getevents` 和 `MSG_ERRQUEUE`。这大大分裂了内核提供的实现。

话虽如此，Linux下的开发者仍应该很高兴看到允许新的异步接口(可以用来提高服务器性能)。现在就可以编写代码，用 `io_submit` 替换古董级的 `epoll` 事件循环！