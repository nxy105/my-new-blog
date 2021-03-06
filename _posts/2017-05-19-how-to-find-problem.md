---
title: 问题分析方法论小结
date: '2017-05-19 08:04:00' 
tags:
- 总结
- 问题分析
---

一个复杂系统运行的过程中，我们总会面对各式各样的问题。有的问题的原因显而易见，而有的问题的原因却隐藏在表象之后。我们常常注意到系统里的奇怪现象，却为不能找到其根源所在，而深感困扰。今天，我们通过一个案例来探讨，当面对一个问题时，我们应该如何分析。

## 现象

这是一个发生在服务器上的问题。我们有多台服务器，运行 PHP，负责处理请求。机器负载均衡，且配置相同，都使用 FPM 运行 PHP 程序。我们发现，相对于其中一台机器，其他的机器平均负载高出其10%-15%。为了方便叙述，我们称这些机器为问题机器。

我们通过监控工具 vmstat 观察到：问题机器的 CPU 进程队列会间歇性增高，到达20-30左右，然后又在下一个时刻恢复到正常值。

这样异常的现象背后一定隐藏着什么，我们要做的就是找到它。

## 推论法

我们很容易地意识到，机器与机器之间是有差异的，就是这样的差异导致了上述的现象。那么差异可能出现在什么地方呢？我们假设了很多情况：

- 负载均衡器的压力不均？
- 编译 PHP 的 GCC 版本不同导致编译的结果不一致？
- 防火墙配置的差异？
- 外部请求的性能差异？

当我们论证这些假设的时候，所有的假设最后都被推翻，证明于负载差异的问题无关。

- 负载均衡确实有差异，但差异是随机的。
- 升级了 GCC 版本重新编译后，现象依然存在。
- 统一了防火墙设置后，现象依然存在。
- 抓取外部请求，进行汇总统计，性能并没有明显区别。

### 事后总结

这个阶段，我们使用的是推论法。我们首先假设问题出现的原因，然后论证我们的假设是否成立。

当问题复杂时，我们会使用多段推论，即先证明我们第一个假设，再做一个让我们第一个假设成立的假设，继续证明我们的第二个假设，直到确认问题所在。这其中要求假设与假设之间存在足够强的逻辑关联性。若非如此，我们容易在推论的道路上误入歧途。

## 演绎法

当所有的假设都被推翻，我们陷入思路枯竭的尴尬境地。这时，我们决定扭转思路。既然现象所指的是 CPU 时钟占用的问题，那我们就来监控 CPU 的活动。

这时就需要合适的工具，有幸（很多时候能发现合适的工具都算是上天眷顾的结果）我们了解 pref 可以监控 CPU 时钟占用情况。

通过 pref 分析的结果发现，问题机器在 `mutex_spin_on_owner` 和 `_spin_unlock_irqrestore` 上占用的很多时间，这两个的操作和利用多核 CPU 计算有关。同时，我们发现问题机器上，libgomp.so.1.0.0 库的 `gomp_barrier_wait_end` 函数占用了很多时间。

接下来，我们就需要找到谁在调用这个库函数。首先，我们选取一个 FPM 进程查看其使用的动态库。

```bash
lsof -p `ps aux | grep "[p]hp-fpm: pool"| head -n 1 | awk '{print $2}'` | awk '{print $NF}'| sort |uniq | grep so | xargs ldd
```

通过搜索动态库的调用链，我们发现 `imagick.so` 动态库是调用的源头，在编译的参数中，问题机器没有关闭多核功能。这就造成了问题机器在处理图片时，会造成一个短暂的波峰，和我们最初观察到的现象一致。

### 事后总结

当推论的思路遇到瓶颈时，我们可以试着转变思路。问题出现在什么地方，就使用相应的工具剖析现象背后的直接原因以及更深的原因。这是一个推理的过程，推演的每一步的原因都是上一步推理的结果，我称之为演绎法。

当完成推理后，如果推理链较长，我们需要做一个复盘。不同于总结，复盘旨在检验我们推理的每一步是否足够合理。

## 结语

通过这一个问题，我们尝试了两种不同的思路分析。为了区分，我们分别总结了这两个思路的特征。但解决实际问题时，我们往往是结合这两种思路进行分析。当推理到了死路，我们可以通过假设寻找新的线索并加以证明。当假设遇到断崖，我们可以通过推理得到新的现象以助于进一步推演。

方法是死的，人是活的，灵活运用以解决问题才是我们的终极目标。