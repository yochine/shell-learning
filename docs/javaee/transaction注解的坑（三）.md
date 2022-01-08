你好呀，我是why。是这样的，我周一的时候不是发了[《仔细思考之后，发现只需要赔6w》](https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247521955&idx=1&sn=8fe1b866980b5d3352df8c280a964c80&scene=21#wechat_redirect)这篇文章吗。好家伙，我以为这事写个上下集就算是大结局了。没想到，还需要补一篇来说明一下。首先，给大家说一声对不起：我错了，行了吧![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMGyoTSP7hWg3fUMvaeicRskjNTkiaal3QRJicj4UKibu6PLuj1g9BCDzWYA/640?wx_fmt=png)。

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMJ1BPLibAnYksWjaoUQHfPXnf1lt2QccrRIrM2zx3sOWAefj7IicF7N7w/640?wx_fmt=png)

那篇文章里面有一个基本的论点是错误的，就是这里：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMkVqUlBKf3N4lTD1IEP4fwwSR5FodotXyenKuINoFsZcnVibKhAcc7yw/640?wx_fmt=png)

是的，错在了天王老子这一句，脸打的啪啪的，老疼了。

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcM2s6cFfs7YNkCFnWU2bUD9Qye2pQkWiciaH83ic9XnYu9fWtDkHSBG0HGw/640?wx_fmt=png)

现在，我重新描述一下天王老子这一句：

> 在 10 个库存，100 个并发的前提下，就算是王母娘娘来了，绝大部分情况下订单数都是 20 个，但绝不可能超过 20 个。

下面我解释一下什么情况下订单数会小于 20 个。还是拿这个图片来说事：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMphoeia3YS5RP6EyXaOIYV66DX1L5fIc6riaBF8pPmpVoRykwPLcoyYrg/640?wx_fmt=png)

首先，这个图片我是截取了一部分日志，根据日志画出来的图：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMRK5ib5fj3EibrDQabJ1QgCDBbMJguyUVXaQsqwyY4NwicKlNMDjicFeVUw/640?wx_fmt=png)

日志里面打印的 Thread-107 的库存是 2，于是我画到图中。所以我上篇文章里面的推理过程是没有错的，换句话说，那篇文章解释了为什么大多数的情况下订单跑出来都是 20 个，即日志为什么是这样打印的。虽然我自己跑了不下 100 次，每次都是稳定 20 个。但是日志一直都这样打，就能推理出订单数一定是 20 个吗？不能。是的，不能啊。打完脸之后，我给你分析一下。问题就出在 Thread-107 查询出库存为 2 这一步，我把情况最简化，重新打上时间标记，基于这个图去说：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcM17fTNQt9GoFuzwfcsyQCRQeSr6hS180ibVtV3L3ccDYhQ463Ap5jndw/640?wx_fmt=png)

首先，T2、T3 时刻一定是晚于 T1 时刻，T2、T3 时刻是推导不出先后关系的，这个不再多说。那么假设，最开始的情况下库存就是 2 个。T2 时刻事务还没来得及提交，就执行到 T3 时刻，那么 T3 时刻查询到的库存就为 2。如果 T2 时刻事务已经提交完成，然后执行到了 T3 时刻，那么 T3 时刻查询到的库存就为 1。现在，假设 T3 时刻查询到的库存是 1，那么同理，下面的 T4 时刻是不是有可能为 1，也可能为 0：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcM7wzkAMbEneuMHYjgPfkK4OXU7FGMZUFPJkR1zNZI4NhLmjvmrIbkQA/640?wx_fmt=png)

如果查出来为 0，那么就和前面的文章里面的这个结论冲突：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMYWb5SZeGqCUxtLJe0ZHmsKkFuiarjoORrrMXKHAEAc87MTYcs4FA8rg/640?wx_fmt=png)

因为前面文章的这个结论是基于程序的运行日志得出来的。现在理论有了，那么我怎么模拟一下订单数少于 20 个的情况呢？想要模拟出这个情况，根据我们前面的分析只需要保证 T3 查询库存的操作晚于 T2 提交事务的操作。那么自然而然的就想到了在查询库存之前加入睡眠时间：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMPzsBBsalicJMYTl6307Qs3Owax6fqQhlLlLAWnTr9TMohtCAVIdZCWA/640?wx_fmt=png)

但是，你会发现这样加，订单每次都是 10 个了呀，这个情况没啥好分析的，就类似于每隔一秒发一个下单请求。锁早就释放了，事务也早就提交了。所以，我改成了这样：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMpkFTuc9OH4EFBQxUibYNZbZ5dB9oZ5yQbUXNibE2NTXPnwFKr09bxV8A/640?wx_fmt=png)

意思就是如果抢到锁的线程名称包含 1，就休眠一下。比如模拟程序进入了 GC，发生了 stop the world。再次执行程序，日志就变成了这样：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMu7qZ04cSnB8LfRMbH2sQYaicicBxWibdm9odjnR6PMjDSsIH2AoeDNPSg/640?wx_fmt=png)

可以看到，没有出现连续两个库存为 5，总订单数也变成了 19 个。验证完成。甚至，我可以扩大线程名称的命中范围，比如这样，就只有 12 单了：

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMsWmathHms56mF90z25cSU90iaddPJl8qlO0DZ9ibsZRGosW7qKiaMaGAg/640?wx_fmt=png)

另外，再提一下另外一个问题。有的同学用分布式锁去做了验证，发现并没有出现超卖的情况。可以在释放锁之后加上一个睡眠时间试一试。这样做的目的是延迟事务提交的时间，以保证下一个抢到锁的线程读到的是未提交之前的库存。好了，上面说了这么多，就是纠正一下之前文章中说的过于绝对的地方，确实是我写的时候被绕进去了。我也狡辩一下。

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMDR5LIozV5O99konnticowrKPv7ZibjC7r22WupzUNyeATUicF0ycSXHPg/640?wx_fmt=png)

你都不知道，我周末写那篇文章的时候晚上洗澡都在想着去证明一个点：

> 订单数会不会出现超过 20 的情况？

![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcML8MovQ07uWL2J779bkfv9tdYqz9k49Sibb8OHPl2FlhH701iaJ68wBaw/640?wx_fmt=png)

害，没想到它在 10 到 20 之间埋伏了我一手，防不胜防啊。我个人是觉得分析小于 20 单的情况比较简单，逻辑也很清楚，还是分析等于 20 单的情况有意思。最后，给大家分享一下我的这篇文章[《当我看技术文章的时候，我在想什么？》](https://mp.weixin.qq.com/s?__biz=Mzg3NjU3NTkwMQ==&mid=2247508753&idx=1&sn=564694b2269ba7d1ec1a8fdd4657976d&scene=21#wechat_redirect)。里面表达了我对于看技术博客的态度：**看技术文章的时候多想一步，有时候会有更加深刻的理解。** **带着怀疑的眼光去看博客，带着求证的想法去证伪。** **多想想 why，总是会有收获的。** 另外，写到这里我想起之前知乎看到的一个故事，和大家分享一下。通过我自己的验证，我跑了上百次的实验，每次都是 20 单。因为相对于查询语句，事务提交是一个比较重的过程。所以，20 单应该是一个绝大部分同学都会遇到的情况。但是真的就有一个同学他说他曾经跑出过 19 单的情况，然后他来问我为什么。原因前面我解释了就不再赘述了。那假设，如果有人连续 100 次、1000 次甚至上千万次，都跑出了小于 20 的这样的小概率事件。那么，你的程序的环境的某个环节一定出了大问题。小概率事件的发生，说明很可能出了大问题。下面这个知乎回答，分享给你：![](https://mmbiz.qpic.cn/mmbiz_png/ELQw2WCMgt2iaQqeFQcBlZeG6aBHNwLcMAEq56xwhWhePpH226UTvqsmo7VAt8VJvX1bSCNYicToUt4RtXicZvicPg/640?wx_fmt=png)

  

  
  