---
layout: post
title:  "agent压测与思考"
date:   2018-11-08 14:34:56 
categories: jekyll update
icon: motorcycle

---

> 接上篇，agent完成后，为了保证基本性能，进行过一些压测，中间遇到一些问题，觉得值得记录下来



## 压测准备

> 由于agent使用unix socket自定协议通信，所以没有通用的压测工具可用，正好在看golang，于是用go写了一发，这里简称为goclient。



为了测试各种情况，goclient提供以下几个基本功能，这里简单介绍一下我的实现

1. 可调整的连接数

   >每个goroutine建立一个链接，并长久维持，那么routine数 == 连接数

2. 可调整的QPS（每秒的自定请求包数量）

   > 每个goroutine在一次`Write/Read`完后，`Sleep` M毫秒，由于进程间通信基本无延时/agent处理逻辑纯cpu计算，所以QPS基本满足：QPS = routinue数 * 1000 / M

3. 可调整的回包超时（感谢大佬提醒）

   >参看`SetReadDeadline` 不过后续发现有些问题，这里先跳过



## 压测表现

> 多线程，虽然有加读写锁，但是主要工作线程只有一个，理论上锁不会block，同时top观察agent cpu不应该超过100%；

测试环境

16核心32GB内存，核心频率2494MHz

50ms 回包超时

（有调整过链接数从500-2000，发现对结果影响不大）

| agent | QPS  | CPU（单核） |
| ----- | ---- | ----------- |
| 1     | 50K  | 34%         |
| 2     | 100K | 70%         |
| 3     | 140K | 97%         |

对比物理机上的openresty压测峰值，表现还是比较乐观，而且QPS与CPU增长基本成线性关系

这里说明一下，由于奔着万级的QPS去，就没测过10K下QPS的情况...



## 接入nginx

> goclient压测完毕后，考虑实际生产环境中，CPU内存不像压测环境如此空闲，而且CPU
>
> 主要占用应该是openresty，于是使用nginx再次测试

这里对比三种情况

1. goclient
2. 接入nginx，使用[vegeta](https://github.com/tsenart/vegeta)发送请求
3. 接入nginx，统一压测平台模拟线上请求

由于nginx与agent使用unix socket通信，必须同机器部署，访问agent只是nginx请求中的一环，所以QPS受nginx限制，无法压到前面的50K以上，30K时整体CPU已经达到峰值。

| agent | QPS  | CPU(goclient) | CPU(nginx vegeta) | CPU(压测平台) |
| ----- | ---- | ------------- | ----------------- | ------------- |
| 1     | 10K  | 6%            | 12%               | 14%           |
| 2     | 20K  | 11%           | 18%               | 24%           |
| 3     | 30K  | 17%           | 25%               | -             |

这里可以看出接入nginx后，agent表现还是下降不少

perf了一把

+ raw_spin_unlock_irqrestore (ep_poll) 自旋锁
+ finish_task_switch 进程调度

这两内核函数在nginx调用时占用更多的cpu，这里当时咨询过大佬，解释是当前系统CPU绝大部分已经被nginx占用，考虑到此时系统调度繁忙，以及nginx的http请求大量的网络收发包产生的中断，的确可能影响agent表现。



## 线上灰度

> 之前的压测告一段落了，于是找了一台线上机器灰度接入验证

然而线上结果表现为，2K QPS时agent占用cpu就达到5%，这么算下去跑满单核只有40K的QPS，这与压测结果相距甚远，于是又分别用goclient与压测平台测试2K QPS，结果分别的是2%与4.5%左右，这里说明压测平台表现基本与线上真实环境一致，能复现就能接着查下去，于是乎本着性能问题先perf的原则开始调查。

nginx压测时perf report --no-children

```c++
-   28.99%  kakyoin-agent  [kernel.vmlinux]    [k] _raw_spin_unlock_irqrestore                                                                               ◆
   - _raw_spin_unlock_irqrestore                                                                                                                             ▒
      - 25.01% __wake_up_sync_key                                                                                                                            ▒
         - 23.74% unix_write_space                                                                                                                           ▒
              sock_wfree                                                                                                                                     ▒
              unix_destruct_scm                                                                                                                              ▒
              skb_release_head_state                                                                                                                         ▒
              skb_release_all                                                                                                                                ▒
              consume_skb                                                                                                                                    ▒
              unix_stream_read_generic                                                                                                                       ▒
              unix_stream_recvmsg                                                                                                                            ▒
              sock_recvmsg                                                                                                                                   ▒
              SYSC_recvfrom                                                                                                                                  ▒
              sys_recvfrom                                                                                                                                   ▒
              system_call_fastpath                                                                                                                           ▒
              __libc_recv                                                                                                                                    ▒
         - 1.26% sock_def_readable                                                                                                                           ▒
            + 1.24% unix_stream_sendmsg                                                                                                                      ▒
      - 1.02% ep_poll                                                                                                                                        ▒
           sys_epoll_wait                                                                                                                                    ▒
           system_call_fastpath                                                                                                                              ▒
           0xf7d43                                                                                                                                           ▒
      + 0.91% hrtimer_start_range_ns                                                                                                                         ▒
      + 0.78% ep_scan_ready_list.isra.7                                                                                                                      ▒
      + 0.56% skb_queue_tail                                                                                                                                 ▒
+    2.58%  kakyoin-agent  [kernel.vmlinux]    [k] copy_user_generic_string                                                                                  ▒
+    2.39%  kakyoin-agent  [kernel.vmlinux]    [k] fput                                                                                                      ▒
+    2.18%  kakyoin-agent  [kernel.vmlinux]    [k] pvclock_clocksource_read                                                                                  ▒
+    1.83%  kakyoin-agent  [kernel.vmlinux]    [k] sock_alloc_send_pskb                                                                                      ▒
+    1.83%  kakyoin-agent  libc-2.17.so        [.] _int_malloc    
```



goclient压测时perf report  --no-children

```c++
-    7.95%  kakyoin-agent  [kernel.vmlinux]    [k] _raw_spin_unlock_irqrestore                                                                               ▒
   - _raw_spin_unlock_irqrestore                                                                                                                             ▒
      - 5.23% __wake_up_sync_key                                                                                                                             ▒
         - 4.50% unix_write_space                                                                                                                            ▒
              sock_wfree                                                                                                                                     ▒
              unix_destruct_scm                                                                                                                              ▒
              skb_release_head_state                                                                                                                         ▒
              skb_release_all                                                                                                                                ▒
              consume_skb                                                                                                                                    ▒
              unix_stream_read_generic                                                                                                                       ▒
              unix_stream_recvmsg                                                                                                                            ▒
              sock_recvmsg                                                                                                                                   ▒
              SYSC_recvfrom                                                                                                                                  ▒
              sys_recvfrom                                                                                                                                   ▒
              system_call_fastpath                                                                                                                           ▒
              __libc_recv                                                                                                                                    ▒
         + 0.73% sock_def_readable                                                                                                                           ▒
      + 0.63% skb_queue_tail                                                                                                                                 ▒
      + 0.52% ep_poll                                                                                                                                        ▒
+    4.18%  kakyoin-agent  [kernel.vmlinux]    [k] copy_user_generic_string                                                                                  ▒
+    3.35%  kakyoin-agent  [kernel.vmlinux]    [k] sock_poll                                                                                                 ▒
+    2.93%  kakyoin-agent  libc-2.17.so        [.] _int_malloc                                                                                               ▒
```



可以看到nginx压测时 `unix_write_space`比例升高，于是开始进行一些猜测与验证



> 1.recv数量增多

因为调用`unix_write_space`的多是些recv，与skb（socket buffer）相关内核函数，于是怀疑tcp recv数量更多占用了cpu，然而加上log统计表明，2K qps时，recv数量相同，那么并不是这个原因



> 2.中断增多

看到最底层的`_raw_spin_unlock_irqrestore`，也开始怀疑是中断导致，然而看到源码里`unix_write_space`引用处上方有这样一段注释

> ```
> /*
>  * AF_UNIX sockets do not interact with hardware, hence they
>  * dont trigger interrupts - so it's safe for them to have
>  * bh-unsafe locking for their sk_receive_queue.lock. Split off
>  * this special lock-class by reinitializing the spinlock key:
>  */
> ```

AF_UNIX并不会触发中断



后面咨询了下大佬，也有尝试减少nginx进程，发现效果并不明显，过程中倒是发现一个现象，nginx压测时，随着QPS的提高，`unix_write_space`比例反而降低，整体QPS并不是等比关系，例如2K--5%，20K--25%。

看了一天的perf，依然找不到问题所在，有点想放弃，开始网上随便找些linux下进程性能分析的文章看看，突然看到strace，这里才想起搞了半天还没有strace过一把。

于是man strace，看到了-c 可以统计系统调用次数，于是开始strace agent在不同压测环境下10s的系统调用情况

2K  QPS， 100+ unix socket 链接

timeout 10s strace -p pid -c -f

goclient 数据

```c++
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 99.05   10.360001       13097       791           epoll_wait
  0.30    0.031279           1     40502           gettimeofday
  0.27    0.028220           1     20046           recvfrom
  0.22    0.022966           1     20009           sendto
  0.10    0.010356          32       320           timerfd_settime
  0.05    0.005503           3      1962           clock_gettime
  0.01    0.000927           0      2874           write
  0.00    0.000301          38         8         1 futex
  0.00    0.000015           0        36           poll
  0.00    0.000005           1        10         9 epoll_ctl
------ ----------- ----------- --------- --------- ----------------
100.00   10.459573                 86558        10 total

```



nginx数据

```c++
% time     seconds  usecs/call     calls    errors syscall
------ ----------- ----------- --------- --------- ----------------
 99.50   15.468624         820     18869           epoll_wait
  0.14    0.021842           1     40492           gettimeofday
  0.11    0.017553           1     20048           recvfrom
  0.09    0.014027           0     38191           clock_gettime
  0.09    0.013814           1     20006           sendto
  0.06    0.009260          29       320           timerfd_settime
  0.01    0.000924           0      3191           write
  0.00    0.000034           1        40           poll
  0.00    0.000000           0        10        10 epoll_ctl
------ ----------- ----------- --------- --------- ----------------
100.00   15.546078                141167        10 total

```



这里明显看到了`recvfrom/sendto`基本对等，并且数量等于2K * 10，但是`epoll_Wait`竟然查了20倍之多！

突然觉得这里就是突破点，于是再次向大佬请教了一下

同样的数据包量，每个连接上请求是阻塞（等待上一个回包后继续），`epoll_wait`少，说明每次`epoll_wait`返回的唤醒fd更多。于是回顾goclient的代码，我的并发方式正好是全部routine sleep一个统一时间后在继续发包，虽然不能真正同时，但是数据的确像是"一批一批"到达agent，而nginx的请求可能更加分散，导致`epoll_wait`每次得到的fd数据更少，调用数量增多。



为了验证猜，我在goclient的每个routine后，额外`sleep rand.Intn(1000)` 微秒，再次压测，cpu升到5%，同时strace里`epoll_Wait`变成10K+...

原来最后竟然是我的压测工具数据不均匀导致的，这里也能解释为什么随着QPS增高，`epoll_Wait`数量反而降低，因为数据到来更加频繁。



## 一些想法

> + 后来回想整个过程，其实现象也很明显，只是查问题的思路不太对。
>
> + golang真的是方便鸭
>
> + 能有这样的时间查问题，感觉以前是不敢想的，业务需求都搞不完，哈