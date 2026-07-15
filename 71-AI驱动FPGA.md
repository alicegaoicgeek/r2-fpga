# 实测 Fable 5 比 Opus 4.8 强多少

> **公众号**: AI驱动FPGA **作者**: JieL **发布时间**: 2026-07-03 19:25 **原文链接**: [https://mp.weixin.qq.com/s/IzG1J3pyEfVYtNVlhvTwgg](https://mp.weixin.qq.com/s/IzG1J3pyEfVYtNVlhvTwgg) **摘要**: 同一道 256 抽头 FIR，同一套框架、同一台机器。Opus 4.8 好几次说「差距是结构性的，认命吧」；换成 Fable 5，两小时做到跟厂商 IP 打平。实测差在哪。

---

![图片](file:///Users/gaoyuan/Workbuddy/2026-07-15-15-13-46/%E5%AE%9E%E6%B5%8BFable5%E6%AF%94Opus4.8%E5%BC%BA%E5%A4%9A%E5%B0%91/images/image_1.png?lastModify=1784100648)

1820 → 502

同一道 FIR，只换了个模型

这篇不聊跑分。我手里有一道折腾了好几天的硬件题，先是用 **Opus 4.8** 做，卡住了，它好几次跟我说「这差距是结构性的，别做了，直接调厂商 IP 吧」。后来我把模型切成 **Fable 5**，同一套框架、同一台机器、同一个对照标准，大概两个小时，它把这道题做到了跟厂商 IP 打平。

两个模型做的是完全同一道题，我只在中间敲了一次 `/model`。 所以这回不是我拿两个模型各做各的活儿再硬凑一个对比，而是一道题、一个现场，看它俩到底差在哪儿。

## 1这道题到底是什么

题目是一个 256 抽头的相关器。往通俗了说，就是拿进来的信号，去跟一段固定的 256 个系数逐点相乘再累加，看它们有多「像」。这在 5G 里是用来找同步信号的。

![图片](file:///Users/gaoyuan/Workbuddy/2026-07-15-15-13-46/%E5%AE%9E%E6%B5%8BFable5%E6%AF%94Opus4.8%E5%BC%BA%E5%A4%9A%E5%B0%91/images/image_2.png?lastModify=1784100648)

这道题的目标：让生成的折叠 FIR 追上厂商 IP

我不想要 256 个乘法器全铺开，太费硅片。所以做折叠：让 32 个乘加单元轮流干 8 拍，用时间换面积。折叠 8 倍，256 兆的时钟，对应 32 MSPS 的吞吐。

对照标准是现成的：Xilinx 官方的 FIR Compiler IP。我把它真的跑了一遍综合布线，不是拍脑袋估的，拿到的是 492 个 LUT、1233 个触发器、32 个 DSP、0 块 BRAM，跑到 524.9 兆。这就是我要追的那条线。它厉害的地方在于，那 256 拍的延迟整个藏在了 DSP 的级联链里，外面的通用逻辑几乎什么都不干。

> ℹ Note

几个词先说清楚：DSP 是 FPGA 里的硬核乘加单元，一个顶好多查找表；LUT（查找表）和触发器是通用逻辑，越省越好；SRL 是移位寄存器，用来存延迟；BRAM 是块存储。追平 IP，就是用尽量少的 LUT / 触发器，把 DSP 也压到 32。

## 2Opus 4.8 卡在哪

Opus 的第一版，用了一个循环缓冲区（pointer-RAM）存样本。问题是折叠以后，32 条通道每拍都在缓冲区的不同地址上读数，综合工具没办法，只能给我摆出一个 256 选 1 的大选择器，还乘以 32 条通道。

结果是 12239 个 LUT、6393 个触发器、64 个 DSP。是 IP 的二十五倍。而且这一版连 256 兆都跑不到，只有 205 兆。

![图片](file:///Users/gaoyuan/Workbuddy/2026-07-15-15-13-46/%E5%AE%9E%E6%B5%8BFable5%E6%AF%94Opus4.8%E5%BC%BA%E5%A4%9A%E5%B0%91/images/image_3.png?lastModify=1784100648)

Opus 一路往下压，压到 1820 就压不动了

这里第一个关键的点子是我给的：延迟别用带地址的 RAM，用固定位置的移位寄存器。改完之后 LUT 从 12239 掉到 2589，触发器也降下来了。但 DSP 还是 64，时钟还是没到 256（大概 203 兆）。

接着 Opus 想把那一堆部分和塞进 DSP 里算，方向对，做法错了：它额外拉了 31 个 DSP 来做加法，变成 1820 个 LUT、925 个触发器、95 个 DSP。这是 Opus 整个过程里最好的一版。可 95 个 DSP，是 IP 的三倍。

它还试过把系数轮着塞进移位寄存器里省地方，结果是 1916 个 LUT，反倒比原来多花了 96 个 LUT。系数那部分从来就不是瓶颈，这一刀砍空了。

> 「剩下的差距是结构性的，直接上厂商 IP 吧。」这句话它前后说了好几遍。

到 2589、到 1820，每卡一次它就下这个结论。我不太信。每一次那个「剩下的差距」都还能再拆：是那个窗口选择器？是部分和？还是 DSP 用多了？只要拆开，单看一块，都是能治的。真正让这道题没死掉的，是我每次都没接受那个结论。

## 3换成 Fable 5 之后

---
先找桌面上最新的截图。
找到两张最新截图，让我读取它们的内容。
# 4 差在了方法，不是马力

我复盘了一下，两个模型的差别，大头不在「谁更聪明」，在做事的方式。

**真正的差别，是做实验的方式**
同一个「工具会怎么综合」的问题，两种问法

| **Opus：10 分钟全量综合** | **Fable：60 秒微探针** |
| --- | --- |
| · 每验证一个想法都跑一次 | · 每个问题缩成一个小实验 |
| · 32 通道一起上 | · 1 到 4 通道 |
| · 常常一次改两个地方 | · 一次只动一个变量 |
| · 只看报告里的总数 | · 读日志里那行模式串 |
| 「64 个 DSP」摆了好久<br>却没读日志里为什么翻倍 | 动手前先在纸上算下标<br>第一版 RTL 就是对的 |

**快十倍 · 结论可分离**

> *60 秒的微探针 vs 10 分钟的全量综合*

Opus 每验证一个想法，都要跑一次 32 通道的完整综合，十分钟起步，而且经常一次改两个地方，最后分不清是哪个变量起了作用。报告它也只看总数：「64 个 DSP」这行字在那儿摆了好久，可综合日志里明明有一行行写着「这个 DSP 是纯乘法，那个是累加」的模式串，能直接告诉你 DSP 为什么会翻倍，它一直没去读。

Fable 5 反过来。它把每个「工具到底会怎么综合」的问题，都缩成一个一到四通道、六十秒就出结果的小实验，一次只动一个变量，答案直接从那行模式串里读。它还在动手写 RTL 之前，先在纸上把级联的下标递推算清楚，所以第一版 RTL 跑出来就是对的，延迟正好是它算出来的那个数。

> ✅ **Tip**
> 一句话总结这个方法：如果你说不出是哪块硬件把这个操作吞掉了，那你还没真正搞懂这个修法。在硬件资源这一层想问题，比在架构名词那一层想，差了整整一个数量级。

Anthropic 自己对 Fable 5 的说法，是「推理比 Opus 4.8 明显强一档」「会回头检查、验证自己的活儿」「能从第一性原理出发」。我这边题算是从旁边给这几句话做了个实证：它确实没在错的判断上一条道走到黑。

但话说得全。移位寄存器那个根因，是我给的；「部分和可以躲在级联上」这个方向，也是我推的。那套等折叠对比的测量脚手架、还有把厂商 IP 真跑一遍拿实数，都是在 Opus 那个阶段搭好的。Fable 5 的功劳，是把这些方向落到具体的硬件资源上，变成能跑的结构，而且基本一次就对了。

---

# 5 强多少

摆数字最干净，都是同一道题、同一块芯片、同一个折叠倍数下，布局布线后的实测。

**强多少：Opus 最好的一版 vs Fable**
同一道题、同一块芯片、折叠 8 倍、布局布线后实测

| 指标 | Opus 4.8 | Fable 5 | 厂商 IP |
| --- | ---: | ---: | ---: |
| **LUT 用量** | 1820 | 502 | 492 |
| **DSP 用量** | 95 | 32 | 32 |

> **LUT 少 72% · DSP 少 66%**
> *Opus 最好的一版 vs Fable 追平 IP*

Opus 最好的一版是 1820 个 LUT、95 个 DSP。Fable 做到 502 个 LUT、32 个 DSP，LUT 少了 72%，DSP 少了 66%，而且落在了 Opus 判死刑的那个厂商 IP DSP 打平点上。从最初那版算起，是 12239 压到 502。

**从 12239 到 502，一整条路**
*纵轴 LUT 用量（按对数，方便看清每一步）· 越低越好*

```
12239  ──┐
         │
         └── 2589 ── 1820 ── 502
         (移位寄存器)         (Fable 试完 ILA)
        ◀Opus 4.8▶         ◀Fable 5▶
                             
         ───── 厂商 IP = 492 (基线) ─────
   起点
```

> *从 12239 到 502，一整条路*

要说 Fable 5 比 Opus 4.8 强多少，就是这个数：同一道题，一个说「做不到，认命吧」，另一个把它做到了跟二十年打磨的厂商 IP 打平。

---

# 6 一句话结论

差距不在「同样的思路谁跑得更猛」，差在 Fable 5 把问题往下拉了一层——拉到具体的硬件资源，再配上快十倍的单变量小实验。这个方法我已经写进了框架里，写成了硬规则和一套探针工具。所以下次不管换成哪个模型来跑，这套做法都在。

**模型会一直换，能沉下来的方法。**

---

> **说明**：你提到的「4，5 我截图了」，但实际第二张截图末尾包含了第 6 节「一句话结论」的内容，我按截图原样一并转录了。所有文字均**原封不动**保留作者原文，一个字都没改。
>
> **问**：要不要把这三节内容追加到之前保存的 `实测Fable5比Opus4.8强多少/实测Fable5比Opus4.8强多少.md` 里，把文章补完？

# AI 调 FPGA 也会遇事先怪硬件，追了二十轮

原创 JieL AI驱动FPGA

 _2026年7月15日 09:30_ _英国_

![封面](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqvxp3rPde0wQPIPUGY4ekOXzd9rFbsu8e2bu5un028xtWIaIapq77pIvtetl3ibHwxpvHdhCatT1lUK72EBJv1I57T064fI2Qiac/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

先验数据通路

再调计算，不要猜

在 FPGA 板子上调试，AI 表现得特别像人。

遇到一个一时半会解释不清的问题，第一反应几乎都是：是不是芯片有问题？是不是板子有问题？然后一头扎进去排错，绕很久很久，最后发现芯片和板子从头到尾都是好的。

额度重启的那一刻，我把这一轮 AI 烧掉的 token 翻了一遍，看到的就是这么个故事。主角是 AI，账单是 token。大部分 token，不是花在解决问题上，而是花在追一个根本不存在的问题上：AI 把一个数据搬运的小 bug，当成了芯片本身的毛病，来来回回折腾了将近二十轮。

这篇讲的就是它是怎么一步步走进这条死胡同的。

  

## 1任务很简单，症状很干脆

  

任务是把一个 802.11 无线接收机放到一块 RFSoC 开发板上跑起来。这块芯片里有两部分：一部分是 PL，也就是能自己搭电路的 FPGA 逻辑，接收机就跑在这里，负责收数据、算结果；另一部分是 PS，一颗 ARM 处理器，跑着 Linux，负责把结果读走。两部分在同一颗芯片上，中间靠一条数据搬运通道把结果从 PL 搬到 PS。

这套接收机之前在另一块板子上验证过，逐位对得上标准答案。

换到新板子上，症状很干脆：PS 那头（跑 Linux 的 ARM）读回来的，是一片全零。接收机像是死了，一个字节的有效数据都吐不出来。

板子能上电，能烧程序，能加载比特流。PL 里的逻辑也不是新写的，是搬过来的。所以第一反应，问题应该出在两者之间某个地方。

  

## 2AI 一上来就怪芯片

  

这里犯的第一个错，是最贵的那个，也最像人。

AI 的记忆里有一条现成的假设：这套设计里的乘法器，在这一代芯片上综合出来的行为和上一代不一样，之前踩过类似的坑。于是它几乎没做测量，就认定是芯片器件的问题。这一步和人遇到怪事就先怀疑硬件，是一模一样的条件反射。然后它围着这个假设开始改、综合、烧板、再看。

  

> ⛔ Caution
> 
> 每一轮，一次综合布局布线要跑二十分钟，加上烧板、上板测试的等待。这个循环重复了将近二十次，而它追的那个「器件问题」，从头到尾就不存在。

  
一个 token 上的教训：记忆里的一条老经验，不是结论，只是一个待验证的猜想。AI 把它当成了结论。而所有的仿真都是通过的，这本该是个强烈的信号，它却读反了。

  

**▍ 为什么所有仿真都放过了它**

因为所有仿真层都共享同一个盲区：没有一层去模拟 PL 和 PS 之间那条数据搬运通道。它们都在验证 PL 里的计算，而 bug 恰恰在计算之外那条搬运通道上。一堆共享同一个前提的验证，对那个前提本身是零信息量的。它们一起通过，只说明那个前提没被测到，不说明它是对的。

  

## 3更糟的是，AI 的尺子本身是歪的

  

如果说怪芯片是主线，那这一条就是把主线越拖越长的帮凶。

为了定位问题，AI 写了一批调试脚本，把 PL 内部每一级的中间数据抓到 PS，和仿真的标准答案逐级对比。听起来很对，正是该做的事。问题是：这批脚本把注入数据的字节序搞反了。

标准答案里的采样点是按大端打包的，脚本却用小端去解。结果就是每一个采样点，字节顺序都被打乱了。两千多个采样点里，两千多个是乱的。

![字节序把尺子弄歪了](https://mmbiz.qpic.cn/mmbiz_png/57vPiach3kqtCHL9ic6xiaE1GbO8ibAOKwt27EtQBIIsNx13MzULWtibBeg77PHBbccqK5sMrRDWN2xsicb0MGklh3yib6WwVyx3Dn50ol4qD9UbKI/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

字节序把尺子弄歪了

于是发生了最坏的情况：AI 拿被打乱的输入喂进 PL，再拿 PL 的输出去对一份正确输入算出来的标准答案。每一级当然都对不上。而这些「对不上」，反过来又被它当成了「芯片有问题」的铁证。

  

> ⚠ Warning
> 
> 一把没校准的尺子，量出来的每一个数都是错的，但它们看起来像证据。AI 对着一批假证据，一次次加深了那个错误的判断。测量工具在被信任之前，必须先自证它是准的。

  

## 4真正的 bug，只是一个没写完的结束标志

绕了一大圈，真正的原因小得可笑。

PL 里的接收机把结果通过数据搬运通道送给 PS，数据是一帧一帧发的，每帧末尾要有一个结束标志，PS 看到结束标志才认为这一次传输完成了。这一点在 AXI 数据搬运里是硬性要求：没有对齐的结束标志，接收端就会一直挂着等，传输永远不算完。

这套接收机一次只解出一个很短的包，短到凑不满一整帧。而那段不足一帧的尾巴，没有被打上结束标志。于是 PS 那头的传输永远等不到结束，它就把那块还没被写入的内存原样返回了，那块内存里是一片零。

![真正的 bug：一帧没写完](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqtGZViavX8Bfj1tGto6W0SL3mILib9k7Kzlj1w3CKpNbia7jKRLyDWl0r3ts83vqqicicj7QnQgTo97qjCo7ia38RnLGONNCCpJJJE24/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=2)

真正的 bug：一帧没写完

接收机其实一直是好的。同样这段数据，在仿真里逐位解得出正确结果，在上一块板子上也跑得通。之所以上一块板子没暴露这个问题，是因为那次跑的是连续的多包数据流，帧一个接一个凑满了，尾巴问题被盖住了。新板子上跑单个短包，一下就顶到了这根刺上。

修复本身，是在数据搬运那一级，给最后那段不足一帧的尾巴补上一个结束标志。就这么点事。芯片没坏，PL 没坏，PS 也没坏。

  

## 5Token 到底烧在哪了

  

把这一轮拆开看，绝大部分消耗，集中在两件本可以避免的事上。

![Token 都花在了追一个不存在的问题上](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqsXibMnYUnQazBibrjzFiblN8bAnpACKtdyaZBoibtOhK1X23aJ6icNrWLBCZuZtjQiax83hSPKI2l6ejOJEBeyUfia93yHXWtMLrFRA4/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=3)

Token 都花在了追一个不存在的问题上

- ▶**追不存在的器件问题：**
    
     将近二十轮「改设计 · 综合布局布线 · 烧板 · 上板看结果」，每一轮都是实打实的算力和等待
    
- ▶**和坏掉的工具缠斗：**
    
     字节序、抓到的陈旧脏数据、打包顺序、一层层的脚本 bug，AI 在这些自造的坑里反复填坑
    
- ▶**真正有用的部分：**
    
     一旦方向对了，定位加修复只花了很小一部分，因为问题本来就小
    

一句话总结这张账单：token 不是被难题烧掉的，是被错误的方向和坏掉的工具烧掉的。

  

## 6本该走的那条路

  

复盘最扎心的地方在于：正确的做法，框架里其实早就写着，只是当时是「建议」，没有被强制，AI 在压力下就跳过了它，直奔那个更刺激的器件猜想。直到人把方向拽回来，让它先去验证数据通路、把逐级对比做对，事情才扭过来。

事后把这套方法固化成了必须执行的步骤，核心就三条。

![如果一开始就走这条路](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqvojmTmGab1sIaTichibPaz6YibcClJ8hbkmNPQzGtj5o94k5JrLnpl6TzTX0OTfTc9W9hgGgsNEiaX2RUAIzDvX0a2fXJTchjiaXias/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=4)

如果一开始就走这条路

> ❗ Important
> 
> 先验证数据通路，再调计算。在证明「PS 能把一段已知数据原样送进 PL、再原样读回来」之前，不许碰任何计算层面的调试，更不许去归咎于芯片。

三层，按顺序来，下面一层不过，上面一层免谈。

- ▶**第一层，通道自证：**
    
     一段已知的图案（比如一段斜坡）灌进去、读回来，不带任何设计逻辑，先证明 PL 和 PS 之间的搬运本身是通的
    
- ▶**第二层，数据通路加仪器自检：**
    
     用已知数据双向对拍，而且要包含那段不足一帧的短数据，同时这一步顺便给测量工具做校准，字节序、打包、结束标志，都在这里被证明是对的
    
- ▶**第三层，逐级对比：**
    
     只有前两层通过了，才开始一级一级抓中间数据、对标准答案，而且每一次都用同一份数据，保证可比
    

![三层方法论：数据通路优先](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqstf0JjmkxbFhLPxJrqukcO0vnZeuPsIc7e4AT98Ad6OgicNgOicxxb7IL682RTODJ1ofvJyMGET92PuohBCmfj6MGoCp96FMAfs/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=5)

三层方法论：数据通路优先

还有一条，是这次上板复盘里补上的：定位到一个「出错的级」，先别急着下结论，它可能只是一次偶发的抓取抖动。先把它单独复现几遍，确认这个错是稳定的、可重复的，再动手。这次就差点又把一个偶发的抓取误差，当成一个真实的 bug 去修。先把现象刻画清楚，再假设根因，这一步第一遍没做，第二遍补上了。

  

## 7写在最后

  

芯片没坏，板子没坏。坏的是方向，和手里那把尺子。而「遇事先怪硬件」这个毛病，人和 AI 都一样容易犯。

> 所有仿真都过、板子还是不对的时候，第一个该怀疑的，是那条没人测过的搬运通道和手里那把尺子，不是芯片，也不是记忆里那条老经验。

方向错了，工具还歪，再多的算力也只是在把错误挖得更深。把尺子校准，把通道走通，这两步做在前面，后面的问题往往小得让人意外。

  

往期文章：

[走 10G 链路烧写 FPGA：远程自动更新](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484836&idx=1&sn=865ab354974c162b4ba2e6ced8549606&scene=21#wechat_redirect)

[AI 调通两块 FPGA 之间的 10G 链路：一条错误记录骗过了所有仿真](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484801&idx=1&sn=487289bae5578b681e119a7d7ff148d6&scene=21#wechat_redirect)

[AI 用一天, 把 Linux 跑上了我们这块 RFSoC 试用板](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484791&idx=1&sn=49adcd500826c3d05b766549dbae970d&scene=21#wechat_redirect)

[用 AI 自动生成一台 5G 接收机：从空口信号里读出小区信息](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484784&idx=1&sn=5d1a4e4a767195ad626e404d9bf0d0a7&scene=21#wechat_redirect)

[实测 Fable 5 比 Opus 4.8 强多少](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484727&idx=1&sn=d2d29e791e2efb3146998126e3536546&payreadticket=HMmzvbc4CoCDwiIFZjbkwdDDO91FqxnjovVSxjXB29LqJNuiWr8dL2QDN6RRXuFV643_GWY&scene=21#wechat_redirect)

[在软件无线电（SDR）平台上快速部署自定义硬件：Wi-Fi 802.11a 接收机在 ADALM-Pluto 上的实测](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484687&idx=1&sn=fc4a67b9f4ecbd935757a9f3c56519c4&scene=21#wechat_redirect)

钟意作者

 创作不易，感谢赞赏☕ 

作者提示: 个人观点，仅供参考

阅读 95

​

**留言**

写留言

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

2

8

推荐

写留言

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

# 在软件无线电（SDR）平台上快速部署自定义硬件：Wi-Fi 802.11a 接收机在 ADALM-Pluto 上的实测

原创 JieL AI驱动FPGA

 _2026年6月26日 09:30_ _英国_

![封面](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqu7RfUt9qvMia4FfDkicZZuD6GeRFrESibXYSL9wKGCak2rfmN5OJ1TKCrqQpof65QzGsen5icGWCKCibQIAC8ib3QhfvibF5XTWAZCCY/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

一套参数配置驱动的软件无线电部署框架，换个参数就换个设计

以 802.11a 为例：同步前端跑在 FPGA 上，其余交给主机，空口图像链路逐比特恢复

这篇讲的是一套把生成的硬件部署到软件无线电上的框架。它是参数配置驱动的：一个生成器、一套拼接和构建脚本、一条从烧写到主机校验的流水线，对每个设计都一样，换个设计只改一个参数文件。下面拿一个 802.11a 接收机当例子，把它真正跑到一颗便宜的板子上。这颗板子叫 ADALM-Pluto，核心是一颗很小的 Zynq-7010，配一颗 AD9363 射频芯片，通过 USB 接主机。

![真实部署：手里这块蓝色的就是 ADALM-Pluto，背后是实时链路分析器](https://mmbiz.qpic.cn/mmbiz_jpg/57vPiach3kquofjmIE0XrYHibeUWDlP4Vj2UoTCfvqRqOxnY5PFK4qAnrDficriaMtvlwicu48n0EhD5jQz7uLicdBhOsuvgVawU5KRicuicpHfT158/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

真实部署：手里这块蓝色的就是 ADALM-Pluto，背后是实时链路分析器

坦白地说：这颗 Zynq 芯片的 FPGA 资源太有限，装不下整台接收机。所以我们这次部署只能把对速率最敏感的同步前端放进 FPGA，剩下的解码只能交给主机来处理。下面这段画面，是从真实芯片上采下来、再在浏览器里回放的：星座图从 BPSK 一路收敛到 64-QAM，图像随着每个包被解出来，一点点拼回来。先看效果，再讲怎么搭的。

![链路分析器回放真实采集的数据，从 BPSK 一路走到 64-QAM](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

链路分析器回放真实采集的数据，从 BPSK 一路走到 64-QAM

跑通了：空口两根天线连续收，图像逐比特恢复，从 BPSK 到 64-QAM 八种速率全过。下面讲两件事：这套框架长什么样，以及怎么把一整套接收机拆到三层处理器上。

> 一套参数配置驱动的流程，一次只换一组参数；把同步放进 FPGA、其余交给主机。

---

## 1一套流程，换个参数就换个设计

这套框架的核心是一条固定的流水线：从一个参数文件出发，自动生成封装、拼进 Pluto 原厂的设计里、构建出比特流、打包成固件、烧写上板，最后在主机上拿基准结果校验。换一个设计，比如从 Wi-Fi 换到 5G 的同步信号，只改那个参数文件，生成器、拼接、构建、烧写脚本一行都不用动。

![一套流程，任意设计：只换一组参数](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一套流程，任意设计：只换一组参数

为了不让上板一上来就栽跟头，我们一级一级点亮：先只回读一段已知数据，把整条工具链验通；再验实际采样数据怎么进来、时钟和缓冲对不对；最后才把设计本身放上去。每一级都是一道硬件关，先在板上过了关，才动下一级。上层一出问题，底层没点亮根本没法查。

![逐级点亮，每一级都是一道硬件关](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

逐级点亮，每一级都是一道硬件关

---

## 2三层处理器：同步放进 FPGA，其余交给主机

这颗芯片上其实有三层算力：一块 FPGA 逻辑（PL）、一个跑着嵌入式 Linux 的 ARM 处理器（PS），还有 USB 另一头的主机（Host）。为什么非得分三层？因为整台接收机加上无线电自己的接口逻辑，根本塞不进这块小芯片：光完整的接收机就要约 8500 个 LUT，射频接口又要约 8500 个，而整颗器件总共才 17600 个。两个一起放，放不下。

于是这么分：

- ▶FPGA（PL）
    
     放最吃速率的同步前端，包检测、频偏校正、精定时、采样环和 FFT，输出频域符号
    
- ▶处理器（PS）
    
     负责把数据搬出来，DMA 引擎、射频芯片的驱动，还有连到主机的 USB 桥
    
- ▶主机（Host）
    
     接着做接收机的后端，信道估计、均衡、软解映射、卷积译码，最后把图像拼回来
    

FPGA 到处理器走 DMA，处理器到主机走 USB。关键是，FPGA 和主机两侧跑的是同一个基准模型，所以这么一拆，整条链路还是逐比特一致。

![三层处理器：同步放进 FPGA，其余交给主机](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

三层处理器：同步放进 FPGA，其余交给主机

这不是示意图。下面是 Vivado 里真实的工程截图：我们的同步核（绿框）确确实实嵌在了 Pluto 原厂的 FPGA 设计里，旁边是射频接口、送往处理器的 DMA，还有那颗 Zynq 处理器。

![Vivado 工程截图：同步核嵌入 Pluto 的 FPGA 设计](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Vivado 工程截图：同步核嵌入 Pluto 的 FPGA 设计

---

## 3结果：每种速率都逐比特恢复

开头那段画面，回放的就是空口实测：一张图进、原样一张图出，在两根天线上做，采到的采样直接从无线电上存下来，没有任何后期修饰。下面每个数字，都是 FPGA 同步加主机解码这条完整链路一起跑出来的。

![空口实测，64-QAM：真实芯片上的星座图与重建出来的图像](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

空口实测，64-QAM：真实芯片上的星座图与重建出来的图像

- ▶图像 100% 正确
    
     交付的图像逐比特一致，八种速率全部如此
    
- ▶零误码
    
     标准波形 38400 比特，逐比特恢复
    
- ▶有线 100%
    
     换成 SMA 线、干净配置下连跑 90 秒，1390 个包一个不丢
    
- ▶空口 99.88%
    
     原始单包成功率 831 / 832，差的那一个是真实信道抖动，不是主机丢数据
    
- ▶162.7 MHz
    
     同步核布局布线收敛的时钟，板上以 100 MHz 运行（12.5 Msps × 8）
    
- ▶八种速率
    
     从 BPSK 到 64-QAM，全部逐比特一致
    

这里几个百分比，得分清楚，别混在一起：

> ℹ Note
> 
> 交付的图像是 100% 正确的：每一片都重发很多遍，校验会挑出干净的一份，所以无论哪种速率，最后拼出来的图都逐比特一致。空口 99.88% 说的是"原始单包"的成功率（832 个里收对 831 个）；差的那一个，是真实无线信道的一次抖动（多径、噪声、2.4G 干扰），实测当时主机 0 次卡顿，所以它不是主机丢数据，而是空中那一下没收干净，协议靠重发补回了。换成 SMA 线、干净配置时，连原始单包都是 100%（1390/1390）。

---

## 4能带走的方法

这套框架的价值，是把"把生成的硬件搬上软件无线电"这件事，做成一条可复制的流程：换个设计只改一组参数。具体的 Wi-Fi 只是个例子，里面这两条最通用：芯片装不下，就按算力分层，把最吃速率的留在 FPGA，其余沿一个又窄又单向的接口交给处理器和主机；上板则一级一级点亮，每一级都先在真实硬件上过了关，再动下一级。这些才是值得带到下一个设计里去的东西。

  

[用 AI 自动生成一台 Wi-Fi 接收机：从 MATLAB 参考设计到 ADALM-Pluto SDR](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484511&idx=1&sn=34acd5e9cf8dcf5c2f3fe4180fa825fb&scene=21#wechat_redirect)

喜欢作者

 创作不易，感谢赞赏☕ 

1人喜欢

![头像](https://wx.qlogo.cn/mmopen/V9n0c0XaBL5iad7DenZBUc2uUTx3wj5LTJ8wGVr5EjNFOr3D8UMrZTicoJ4RzYoFn2SrEgXbSiaCAkXvttxgaZZJ6qlhQpN0Uq7/132)

作者提示: 个人观点，仅供参考

阅读原文

阅读 5795

​

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

44

282

26

5

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论


# 在软件无线电（SDR）平台上快速部署自定义硬件：Wi-Fi 802.11a 接收机在 ADALM-Pluto 上的实测

原创 JieL AI驱动FPGA

 _2026年6月26日 09:30_ _英国_

![封面](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqu7RfUt9qvMia4FfDkicZZuD6GeRFrESibXYSL9wKGCak2rfmN5OJ1TKCrqQpof65QzGsen5icGWCKCibQIAC8ib3QhfvibF5XTWAZCCY/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

一套参数配置驱动的软件无线电部署框架，换个参数就换个设计

以 802.11a 为例：同步前端跑在 FPGA 上，其余交给主机，空口图像链路逐比特恢复

这篇讲的是一套把生成的硬件部署到软件无线电上的框架。它是参数配置驱动的：一个生成器、一套拼接和构建脚本、一条从烧写到主机校验的流水线，对每个设计都一样，换个设计只改一个参数文件。下面拿一个 802.11a 接收机当例子，把它真正跑到一颗便宜的板子上。这颗板子叫 ADALM-Pluto，核心是一颗很小的 Zynq-7010，配一颗 AD9363 射频芯片，通过 USB 接主机。

![真实部署：手里这块蓝色的就是 ADALM-Pluto，背后是实时链路分析器](https://mmbiz.qpic.cn/mmbiz_jpg/57vPiach3kquofjmIE0XrYHibeUWDlP4Vj2UoTCfvqRqOxnY5PFK4qAnrDficriaMtvlwicu48n0EhD5jQz7uLicdBhOsuvgVawU5KRicuicpHfT158/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

真实部署：手里这块蓝色的就是 ADALM-Pluto，背后是实时链路分析器

坦白地说：这颗 Zynq 芯片的 FPGA 资源太有限，装不下整台接收机。所以我们这次部署只能把对速率最敏感的同步前端放进 FPGA，剩下的解码只能交给主机来处理。下面这段画面，是从真实芯片上采下来、再在浏览器里回放的：星座图从 BPSK 一路收敛到 64-QAM，图像随着每个包被解出来，一点点拼回来。先看效果，再讲怎么搭的。

![链路分析器回放真实采集的数据，从 BPSK 一路走到 64-QAM](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

链路分析器回放真实采集的数据，从 BPSK 一路走到 64-QAM

跑通了：空口两根天线连续收，图像逐比特恢复，从 BPSK 到 64-QAM 八种速率全过。下面讲两件事：这套框架长什么样，以及怎么把一整套接收机拆到三层处理器上。

> 一套参数配置驱动的流程，一次只换一组参数；把同步放进 FPGA、其余交给主机。

---

## 1一套流程，换个参数就换个设计

这套框架的核心是一条固定的流水线：从一个参数文件出发，自动生成封装、拼进 Pluto 原厂的设计里、构建出比特流、打包成固件、烧写上板，最后在主机上拿基准结果校验。换一个设计，比如从 Wi-Fi 换到 5G 的同步信号，只改那个参数文件，生成器、拼接、构建、烧写脚本一行都不用动。

![一套流程，任意设计：只换一组参数](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

一套流程，任意设计：只换一组参数

为了不让上板一上来就栽跟头，我们一级一级点亮：先只回读一段已知数据，把整条工具链验通；再验实际采样数据怎么进来、时钟和缓冲对不对；最后才把设计本身放上去。每一级都是一道硬件关，先在板上过了关，才动下一级。上层一出问题，底层没点亮根本没法查。

![逐级点亮，每一级都是一道硬件关](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

逐级点亮，每一级都是一道硬件关

---

## 2三层处理器：同步放进 FPGA，其余交给主机

这颗芯片上其实有三层算力：一块 FPGA 逻辑（PL）、一个跑着嵌入式 Linux 的 ARM 处理器（PS），还有 USB 另一头的主机（Host）。为什么非得分三层？因为整台接收机加上无线电自己的接口逻辑，根本塞不进这块小芯片：光完整的接收机就要约 8500 个 LUT，射频接口又要约 8500 个，而整颗器件总共才 17600 个。两个一起放，放不下。

于是这么分：

- ▶FPGA（PL）
    
     放最吃速率的同步前端，包检测、频偏校正、精定时、采样环和 FFT，输出频域符号
    
- ▶处理器（PS）
    
     负责把数据搬出来，DMA 引擎、射频芯片的驱动，还有连到主机的 USB 桥
    
- ▶主机（Host）
    
     接着做接收机的后端，信道估计、均衡、软解映射、卷积译码，最后把图像拼回来
    

FPGA 到处理器走 DMA，处理器到主机走 USB。关键是，FPGA 和主机两侧跑的是同一个基准模型，所以这么一拆，整条链路还是逐比特一致。

![三层处理器：同步放进 FPGA，其余交给主机](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

三层处理器：同步放进 FPGA，其余交给主机

这不是示意图。下面是 Vivado 里真实的工程截图：我们的同步核（绿框）确确实实嵌在了 Pluto 原厂的 FPGA 设计里，旁边是射频接口、送往处理器的 DMA，还有那颗 Zynq 处理器。

![Vivado 工程截图：同步核嵌入 Pluto 的 FPGA 设计](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

Vivado 工程截图：同步核嵌入 Pluto 的 FPGA 设计

---

## 3结果：每种速率都逐比特恢复

开头那段画面，回放的就是空口实测：一张图进、原样一张图出，在两根天线上做，采到的采样直接从无线电上存下来，没有任何后期修饰。下面每个数字，都是 FPGA 同步加主机解码这条完整链路一起跑出来的。

![空口实测，64-QAM：真实芯片上的星座图与重建出来的图像](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

空口实测，64-QAM：真实芯片上的星座图与重建出来的图像

- ▶图像 100% 正确
    
     交付的图像逐比特一致，八种速率全部如此
    
- ▶零误码
    
     标准波形 38400 比特，逐比特恢复
    
- ▶有线 100%
    
     换成 SMA 线、干净配置下连跑 90 秒，1390 个包一个不丢
    
- ▶空口 99.88%
    
     原始单包成功率 831 / 832，差的那一个是真实信道抖动，不是主机丢数据
    
- ▶162.7 MHz
    
     同步核布局布线收敛的时钟，板上以 100 MHz 运行（12.5 Msps × 8）
    
- ▶八种速率
    
     从 BPSK 到 64-QAM，全部逐比特一致
    

这里几个百分比，得分清楚，别混在一起：

> ℹ Note
> 
> 交付的图像是 100% 正确的：每一片都重发很多遍，校验会挑出干净的一份，所以无论哪种速率，最后拼出来的图都逐比特一致。空口 99.88% 说的是"原始单包"的成功率（832 个里收对 831 个）；差的那一个，是真实无线信道的一次抖动（多径、噪声、2.4G 干扰），实测当时主机 0 次卡顿，所以它不是主机丢数据，而是空中那一下没收干净，协议靠重发补回了。换成 SMA 线、干净配置时，连原始单包都是 100%（1390/1390）。

---

## 4能带走的方法

这套框架的价值，是把"把生成的硬件搬上软件无线电"这件事，做成一条可复制的流程：换个设计只改一组参数。具体的 Wi-Fi 只是个例子，里面这两条最通用：芯片装不下，就按算力分层，把最吃速率的留在 FPGA，其余沿一个又窄又单向的接口交给处理器和主机；上板则一级一级点亮，每一级都先在真实硬件上过了关，再动下一级。这些才是值得带到下一个设计里去的东西。

  

[用 AI 自动生成一台 Wi-Fi 接收机：从 MATLAB 参考设计到 ADALM-Pluto SDR](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484511&idx=1&sn=34acd5e9cf8dcf5c2f3fe4180fa825fb&scene=21#wechat_redirect)

喜欢作者

 创作不易，感谢赞赏☕ 

1人喜欢

![头像](https://wx.qlogo.cn/mmopen/V9n0c0XaBL5iad7DenZBUc2uUTx3wj5LTJ8wGVr5EjNFOr3D8UMrZTicoJ4RzYoFn2SrEgXbSiaCAkXvttxgaZZJ6qlhQpN0Uq7/132)

作者提示: 个人观点，仅供参考

阅读原文

阅读 5795

​

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

44

282

26

5

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论

# 用 AI 自动生成一台 5G 接收机：从空口信号里读出小区信息

原创 JieL AI驱动FPGA

 _2026年7月6日 09:30_ _英国_

![封面](https://mmbiz.qpic.cn/mmbiz_png/57vPiach3kqsvQsbGUE4uBuzoCCTzuStMrLPicWuCXfgHFsjkMhG3CZcLIdS06CNuGFP08ol1NmGctoqwNbfdiaLibXVicXFM71Z1ArvzGxGib8ibo/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

0 误差

AI 生成的接收机跑在真实芯片上，从空口信号读出广播消息

手机一开机，最先做的一件事就是"找基站"。它并不知道附近有哪些小区、每个小区的编号是多少、系统帧号走到了哪里，于是它扫过整段频谱，去捕捉每个基站每隔一小段时间就发出来的一小块同步信号。这一小块信号在 5G 里叫做 SSB，它把主同步、辅同步、和广播信道打包在一起，占 4 个 OFDM 符号、240 个子载波。手机靠它对上时间和频率、认出小区编号、再把广播消息读出来。

MATLAB 有一份参考设计，叫 5G NR Signal Detection，专门演示这一整套小区搜索的流程。我手上则有一段真实录下来的空口信号，是一个 30 kHz 子载波间隔的 5G 下行，里面确实含着可以解出来的广播消息。我想做的，是让 AI 从这份参考出发，自动把整台接收机搬到 FPGA 上，然后拿这段真信号，把里面的小区编号和广播消息一字不差地读出来。

## 1AI 是怎么把参考设计变成硬件的

整台接收机不是我一行行写的，是 Python2Verilog 框架自动生成的。它把每个模块拆成三层，层层之间机器自动核对：

- ▶数学模型
    
     AI 从标准定义出发生成一份可执行的浮点参考，它的每一步都对着 3GPP 的 TS 38.211 和 38.212，以及 MATLAB 5G 工具箱的实现，把"算法就是标准"钉死
    
- ▶周期级模型
    
     浮点参考被转成低位宽定点，并补上逐拍的硬件时序，机器核对它跟数学模型逐位一致
    
- ▶Verilog RTL
    
     AI 把周期级模型翻译成 Verilog，机器核对 RTL 跟周期级模型 0 比特误差
    

这套链子里，每一层都知道自己在跟谁对照，而且这个对照是机器做的，不是人嘴上保证的。

## 2接收机到底要做哪些事

一台 5G SSB 接收机要走完这么一条链子：

- ▶盲检小区
    
     开机时并不知道小区属于哪一组，于是同时拿 3 个候选的主同步序列，各做一路匹配滤波，一起在信号里找相关峰。因为主同步的匹配滤波系数是复数，每一路又拆成 3 个实数子滤波来算，所以前端一共并行跑着 9 个相关器
    
- ▶定位与提取
    
     一边算信号能量、设一条自适应门限，一边在相关峰里挑出真正的 SSB 起点，把这一小块信号从长长的采样流里切出来
    
- ▶OFDM 解调
    
     把切出来的每个符号去掉循环前缀，再做 256 点 FFT,得到 240 个子载波上的频域数据，也就是 SSB 的资源栅格
    
- ▶广播译码
    
     先用辅同步序列把小区编号精确定位，再用广播信道的导频估出信道、做均衡，把星座点软解调成比特可信度，解扰之后送进极化码译码器，过一遍 24 位 CRC 校验，最后还原出 32 比特的广播消息，解析出系统帧号、子载波间隔、频域偏置这些参数
    

![整台接收机的信号链。绿色的一段现在跑在 KV260 可编程逻辑里，紫色的一段暂时放在主机上，红色虚线是频谱这个切分点。这只是开发阶段的分工，整机接下来会全部搬上 FPGA。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

整台接收机的信号链。绿色的一段现在跑在 KV260 可编程逻辑里，紫色的一段暂时放在主机上，红色虚线是频谱这个切分点。这只是开发阶段的分工，整机接下来会全部搬上 FPGA。

## 3现在先上 KV260，前端在芯片、译码暂放主机

先说为什么用 KV260。我们平时把这类射频设计放在 ADALM-Pluto 这种小巧的 SDR 上跑，但 Pluto 那颗 FPGA 太小，装不下整台 5G NR 接收机，所以这次先借 KV260 这块更大的板子，把 5G 设计放到真硅片上验证起来。

再说为什么现在分成两半。检测和解调是定速的流式运算，每一拍都在动，先在 KV260 上满速跑了起来；广播译码是控制密集的活，要在 336 个候选里搜辅同步、要跑极化码列表译码，这部分暂时放在主机上用软件先跑通，开发起来快。这些都是临时的，等把译码也搬进硬件，整台接收机就完全在 FPGA 上跑了。

当前的切分点选在整条链子最窄的接口，也就是频谱这个位置：FPGA 前端把 SSB 的频域栅格算出来送出来，主机接着把广播消息译出来。FPGA 前端送出来的是 16 位定点的频谱，精度已经被量化过了，但主机这一侧照样能把广播消息 CRC 校验通过地译出来，因为导频均衡和极化码的编码增益足够把量化噪声吃掉，这也说明把译码搬到定点硬件上不会掉性能。

![KV260 上的整机集成框图：DDR 里的采样经 mm2s 读出、过 AXIS 交换进接收机前端；前端算出的频谱再经同一个交换、由 s2mm 写回 DDR。AXIS 交换是 2×2 交叉，两条数据流都从它中间走。按 Vivado 里的实际连接画出。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

KV260 上的整机集成框图：DDR 里的采样经 mm2s 读出、过 AXIS 交换进接收机前端；前端算出的频谱再经同一个交换、由 s2mm 写回 DDR。AXIS 交换是 2×2 交叉，两条数据流都从它中间走。按 Vivado 里的实际连接画出。

![上面是重画的整机框图；这张是实际从 Vivado 导出的整机集成设计，我加了标注：Zynq 处理器、AXI SmartConnect 互联，加上 mm2s、AXIS 交换、nr5g0、s2mm 四个核,全部连在一起。点开可放大。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

上面是重画的整机框图；这张是实际从 Vivado 导出的整机集成设计，我加了标注：Zynq 处理器、AXI SmartConnect 互联，加上 mm2s、AXIS 交换、nr5g0、s2mm 四个核,全部连在一起。点开可放大。

## 4跟 MATLAB 对照，再拿真信号验一遍

验证分两头说。

一头是拿 MATLAB 5G 工具箱当独立裁判。用工具箱自动生成上千个严格符合标准的 SSB 波形，每一个都带着已知的小区编号和广播消息，再让这台接收机去译，逐比特核对译出来的广播消息跟发出去的是不是一模一样。

另一头是拿真实空口信号验。那段录下来的 5G 下行里含着 3 个小区，我们这台接收机把 3 个小区的 SSB 全都译了出来，读出的是同一份广播消息，系统帧号都是 788、子载波间隔都是 30 kHz。这件事本身就说明了问题：同一片区域里同步的几个小区，广播消息本就该一致。

顺带还揪出了参考脚本里的一个坑。参考里的那个译码函数返回的是加扰域的数据，没有解扰，直接拿去解析广播消息，得出来的系统帧号是错的、而且每个小区还各不相同。我们这台接收机补上了 TS 38.212 里规定的解扰和解交织，读出来的才是真正一致的广播消息。也就是说，我们的译码比那份参考更对。

  

## 5弱到什么程度还能解出来

  

真信号只是一个工作点，要说稳健，得看它在各种信道下的表现。我们用 MATLAB 工具箱生成了 1460 个多样化的测试向量，覆盖全部 1008 个小区编号、所有 SSB 位置、任意广播消息内容、两种子载波间隔，还加上白噪声扫频、载波频偏、频率选择性衰落。结果是：

- ▶全参数空间
    
     干净信道下 300 个随机配置全部逐比特正确，小区编号从 0 到 1007、各种 SSB 位置和广播内容都没漏
    
- ▶弱信号门限
    
     白噪声扫频下，信噪比降到每个资源单元 -2 dB 以上时误块率为零；再往下到 -6 dB 才开始明显掉，这正是极化码低码率的广播信道该有的抗噪门限
    
- ▶真实损伤
    
     频率选择性衰落加载波频偏再加噪声，200 个向量全部译对
    
- ▶定点也满分
    
     把栅格量化到 16 位、甚至更粗的 12 位，300 个向量依然全对，说明部署上去的定点前端并不掉性能
    

真信号那段的星座图误差向量幅度大约 52%,看着很脏，但照样 CRC 校验通过地译出来。这个 52% 我们跟 MATLAB 自己算出来的 52.5% 对上了，是信号本身的质量，不是我们的问题。广播信道就是为了在小区边缘也能读到而设计的，码率极低、编码增益极大，所以星座点糊成一团也能把消息稳稳收回来。

![广播消息译码的抗噪门限。信噪比到 -2 dB 以上误块率为零，再往下到 -6 dB 才开始明显掉，这正是极化码低码率广播信道该有的表现。数据来自上千个测试向量的扫频。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

广播消息译码的抗噪门限。信噪比到 -2 dB 以上误块率为零，再往下到 -6 dB 才开始明显掉，这正是极化码低码率广播信道该有的表现。数据来自上千个测试向量的扫频。

## 6FPGA 实现结果

所有数字都来自布局布线后的实测报告。

- ▶检测器资源
    
     部署在 KV260 上，查找表 LUT 5124、触发器 FF 5944、乘法器 DSP 298、块存储 BRAM 1,时序收敛。跟我之前用 HLS 手写的同款检测器比，在同一颗芯片上 查找表少 38%、触发器少 75%、块存储 1 比 7,只有 DSP 多了 7%。这不是巧合，是我们把 9 个相关器都做成了框架里最优的级联 FIR 结构
    
- ▶整机资源
    
     检测器加解调合到一起，查找表 6175、触发器 6709、乘法器 314、块存储 4,在 256 MHz 下时序收敛，余量 +0.451 ns
    
- ▶部署实测
    
     打成完整的 KV260 overlay(加上数据搬运和平台外围)后，在 200 MHz 下布局布线收敛，余量 +0.403 ns
    
- ▶硅上逐比特验证
    
     把这段真信号通过 DMA 灌进 KV260 的前端，再把算出来的频谱读回来，一共 2540 拍，跟参考模型 逐比特一致、0 误差
    

![检测器资源，同一颗芯片上跟我早先手写的 HLS 实现对比：查找表少 38%、触发器少 75%、块存储 1 比 7。数字来自布局布线后的报告。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

检测器资源，同一颗芯片上跟我早先手写的 HLS 实现对比：查找表少 38%、触发器少 75%、块存储 1 比 7。数字来自布局布线后的报告。

## 7一个能实时看的 demo

我给这台接收机做了个网页仪表盘。它把上千个多样化测试向量当成一段实时到来的信号流，一帧一帧地收、一帧一帧地译，像一台真接收机那样跑起来。

每一帧都能看到当下的信号质量：误差向量幅度、信噪比、译码成功还是失败、小区编号。信噪比高的时候星座图是干净的四个点；随着信道变差，星座点一圈圈散开，信号质量的历史曲线跟着爬高；一直扫到 -8 dB,星座图糊成一团红色噪声云，译码这才翻红失败。这一路看下来，就是广播信道抗噪能力的极限被一点点逼出来的过程。

仪表盘里还有一个按钮，一按就真的驱动 KV260:重新加载 overlay、把信号灌进 fabric 前端、把硅上算出来的频谱拉回来、在主机上译出小区信息。看到的不是仿真，是真芯片跑出来的结果。

![网页仪表盘的实时回放：从干净信号扫到弱信号，星座图一圈圈散开，信号质量、信噪比、译码结果和小区号逐帧更新，右边的历史曲线跟着爬高。](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

网页仪表盘的实时回放：从干净信号扫到弱信号，星座图一圈圈散开，信号质量、信噪比、译码结果和小区号逐帧更新，右边的历史曲线跟着爬高。

## 8小结

这台 5G NR 小区搜索接收机，从 MATLAB 参考设计出发，由 AI 一层层生成：数学模型、带时序的定点周期模型、再到 Verilog RTL,每一层都跟上一层逐比特核对过，没人手写一行。它现在跑在真实的 KV260 芯片上，把一段真实空口信号里的 3 个小区全都译了出来，读出同一份广播消息；检测器还比我早先手写的 HLS 少用 38% 的查找表。

眼下用 KV260、译码放主机，都是临时的：常用的 Pluto SDR 装不下整台 5G 接收机，先借 KV260 把设计放到真硅片上验证起来；译码这部分控制密集，先放主机图开发快。整台接收机接下来会全部搬上 FPGA。这套从标准到硬件、每层逐比特核对的方法本身，并不是 5G 专属的。

前序文章：

[实测 Fable 5 比 Opus 4.8 强多少](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484727&idx=1&sn=d2d29e791e2efb3146998126e3536546&payreadticket=HJr8F7K9bCUxKKywuen5bykZ2e7WOSIf8Qlob75HATEYzukVrGT9Cj1NB87l688CqVWb1Qw&scene=21#wechat_redirect)

[从 Python 算法到上板产品：用 AI 把这条路自动化](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484700&idx=1&sn=165350c91250df6c27ae85ecbbb46203&scene=21#wechat_redirect)

喜欢作者

 创作不易，感谢赞赏☕ 

1人喜欢

![头像](https://wx.qlogo.cn/mmopen/ajNVdqHZLLDoWcK7UXGyIoS85BhK4aCVs1kewIOraS1iccbgA4JwIBzaKZVPO5oehTCjKwtrmJfNTiciaG1FvGVT1NicDnhY6RTlmGRziaXSTicLg/132)

作者提示: 个人观点，仅供参考

阅读 1162

​

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

30

122

10

4

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论

# AI 调通两块 FPGA 之间的 10G 链路：一条错误记录骗过了所有仿真

原创 JieL AI驱动FPGA

 _2026年7月11日 16:02_ _英国_

![封面](https://mmbiz.qpic.cn/mmbiz_png/57vPiach3kqtBaIxnWia932FRF1wwnb9vDAVKd0oLa3wZzVRoMt4Umc9kzehW6JwTESXs0EBAF40ibKvvWgmfdxKJOMKOdvVXaz9bF68y7l46U/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

三关全过，逐字节精确

链路、远程算力、远程内存；根因不在代码里，是一条写反的通道映射记录

上一篇里[AI 用一天, 把 Linux 跑上了我们这块 RFSoC 试用板](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484791&idx=1&sn=49adcd500826c3d05b766549dbae970d&scene=21#wechat_redirect)，AI 用一天把 Linux 跑上了这块 RFSoC 试用板。板上两颗大芯片，RFSoC（ZU67DR）和 KU115，中间八条 10G 高速串行线。这一篇是下一步：把 KU115 变成 RFSoC 的"扩展仓"，模块放过去算，内存用它旁边那 4GB 的 DDR4。整个过程由 AI 在 Python2Verilog 框架下完成，人只在关键节点给方向。

> 仿真全对、网表全对、眼图全对，链路就是不通。这种时候该怀疑的不是电路，是你手里的那张地图。

---

## 1一个方向死活不通

链路走 64B/66B 编码：每 64 位数据配 2 位同步头，自同步扰码，接收端连续看到合法同步头就锁定。KU115 发、RFSoC 收的方向很快通了；反过来，KU115 那边的锁定检测永远在滑动，永远差一点。

不通就查，每一步都有测量结果：

- ▶官方例程仿真
    
     同一份收发器配置，85 微秒建链，通过
    
- ▶自环仿真
    
     31 微秒锁定，一帧数据逐位精确
    
- ▶跨芯片仿真
    
     按真实接线连好，两个方向都锁定，回环逐字节精确
    
- ▶线上比特分析
    
     仿真导线上 40 万比特，唯一正确偏移上同步头 100% 合法
    
- ▶布线后网表对比
    
     好坏设计逐属性、逐引脚对比，完全相同
    

五关全过，硬件还是老样子。眼图也干净：

![实测眼图](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqviaEfUdXN9IicM6yKaV5ykiaicxBDKXDuGXGibfHiaouAz1tYsQhMDvplHibU11QuQvQKLFfhjuaTJRl5gNRbyj9AU02SEribMwnZy5nc/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

实测眼图

水平张开度 77.8%，垂直 93.6%，信号质量无可挑剔，问题恰恰不在信号质量。电路每一层都查过没问题，那导线上跑的到底是什么？

---

## 2抓到导线上的真实字节

之前的观察全是统计出来的数字。AI 加了一个一次性抓取器，把锁定状态下连续三拍的原始 64 位字通过 JTAG 读回：

```
0x66666666666666660x99999999999999990x6666666666666666
```

这不是数据。扰码后的数据该像随机数，这却是 0011 循环的方波，相当于一个 2.5 GHz 的时钟。巧的是 66 除以 4 余 2：按 66 位切帧，每帧相位走 2 位，同步头永远落在 01 或 10 上，两种都合法。

![方波如何骗过锁定检测](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

方波如何骗过锁定检测

一个纯时钟图案，喂饱了时钟数据恢复（CDR）锁定、眼图、同步头合法率的全部指标，唯独没有数据。

![导线上的原始比特：时钟图案与真实数据](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

导线上的原始比特：时钟图案与真实数据

上面是 30 次定格抓取每次都出现的方波，下面是后来在正确通道上抓到的真实扰码数据，解扰后就是标准空闲块。这一次抓取，把"数据通路有问题"这一整类假设全部排除了。

---

## 3地图是反的

AI 重新量了一次通道映射：只认物理通道名，不认工具序号；每条通道发不一样的二进制编码标记，三轮解出全部八条。结果和记录正好相反。

![通道映射：记录与实测](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

通道映射：记录与实测

发送端一直往一条没人监听的通道上发正确的数据；监听端连着一条从未配置的通道，它自由振荡出那个方波。所有仿真都照这份记录接线，记录内部自洽，所以每次都通过。

读过上一篇的人会问：八条线不是验过误码率吗，七条零误码，怎么没发现接反？因为误码率测试所有通道发同一个伪随机序列，不管怎么接，每个接收端都能锁定、都零误码，它对"谁连着谁"天生是盲的。铜线确实每条都好，错的是"谁对谁"那张表。

这张表存在框架的板卡描述文件里，仿真接线、上板选通道全依据它。它是排查开始的前一晚随当天的实验提交进版本库的，可以原样调出来：

![出错的那条映射记录，版本库原文](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

出错的那条映射记录，版本库原文

红框里正向被记成"正着连"，还带着"上过硅验证"的注记，那个"验证"只是第一条通道的巧合锁定。同一份记录里反向明明写着"反着连"，当时没人追问一句：同一束铜线，凭什么两个方向不一样？

没人去查原理图，是因为带"实测"注记的结论比 75 页的 PDF 图纸更让人放心，而跨页手工追八对差分线恰恰最容易出错。后知后觉的是，答案一直画在图上：

![底板原理图上的跨芯片串行线](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

底板原理图上的跨芯片串行线

红框里，KU115 这端 127 号收发器组的 0 号引脚，接的就是编号 7 的那条线。

换到正确的两条通道重建，一次就通，全部逐字节精确。顺手修掉的三个真实 bug（锁定搜索滑动太急、回环 FIFO 少接读时钟、测试台溢出和竞态）都留下了，但都不是根因。

---

## 4从原理图 PDF 里抠出 DDR4 引脚表

轮到远程内存。KU115 旁的 4GB DDR4 没有任何机器可读的引脚文件，只有 75 页的原理图 PDF：

![原理图里的 DDR4 引脚页](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

原理图里的 DDR4 引脚页

引脚号一列、网络名一列，64 位接口 119 个引脚，手抄错一根线就是校准失败。红框三行就是下面流程图里用的例子。AI 把它变成一个几何问题：

- ▶文字带坐标
    
     提取每页每个词和它的坐标框
    
- ▶同行配对
    
     FPGA 符号页上同一行的引脚号和网络名配成一对
    
- ▶零冲突硬闸
    
     每个网络画两次，两次必须配到同一个引脚
    
- ▶清单核对
    
     A14、A15、A16 藏在 WE、CAS、RAS 复用名里也要对上
    

117 个网络零冲突。验证交给厂商内存控制器 IP 的字节组和 bank 检查当裁判：

![从原理图到引脚表](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

从原理图到引脚表

烧进板子，DDR4 第一次上电校准就通过。

---

## 5第二块芯片成了扩展仓

三步验收，全部在真实硬件上跑：

- ▶回环
    
     256KB 往返逐字节精确
    
- ▶远程算力
    
     Wi-Fi 标准里的 LDPC 解码器放到 KU115 上，1296 字节软信息进、164 字节硬判决回，与基准模型比对零错，连续两次不复位仍零错
    
- ▶远程内存
    
     跨芯片写 4KB 读回逐字节精确，换个深地址再来一遍仍精确
    

从此 KU115 对 RFSoC 就是三样东西：一条 10G 管道、一个算力节点、一块 4GB 远程内存。

---

## 6留下的三条准则

已写回 Python2Verilog 框架，成为强制检查：

- ▶锁定不等于数据
    
     一个方波能同时满足同步头锁定、CDR 锁定、眼图、合法率四项指标，链路验收必须抓真实载荷解码，逐字节比对
    
- ▶地图只认物理名字
    
     通道映射用两端物理名记录、用二进制标记实测，工具序号不算数
    
- ▶引脚表只信机器
    
     原理图引脚用几何配对提取，零冲突才放行，厂商 IP 检查当裁判
    

最贵的教训：当电路每一层都查过没问题时，该检查的是测量本身。那条写反的记录能存活一整天，就是因为所有仿真都照着它连线。

---

_链路层、远程节点 RTL 与全部排查均由 AI 在 Python2Verilog 框架下完成，所有数字来自当次会话的工具报告与真机测量。_

相关文章：

[AI 用一天, 把 Linux 跑上了我们这块 RFSoC 试用板](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484791&idx=1&sn=49adcd500826c3d05b766549dbae970d&scene=21#wechat_redirect)

[从 Python 算法到上板产品：用 AI 把这条路自动化](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484700&idx=1&sn=165350c91250df6c27ae85ecbbb46203&scene=21#wechat_redirect)

[用 AI 自动生成一台 5G 接收机：从空口信号里读出小区信息](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484784&idx=1&sn=5d1a4e4a767195ad626e404d9bf0d0a7&scene=21#wechat_redirect)

[实测 Fable 5 比 Opus 4.8 强多少](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484727&idx=1&sn=d2d29e791e2efb3146998126e3536546&payreadticket=HAMHaOdl3oNKvKPcnKrFuj627fQiAmb3WdJG6Hfsmjtes8SKlHkn4ohDYhgsFdae0Y8hELY&scene=21#wechat_redirect)

喜欢作者

 创作不易，感谢赞赏☕ 

作者提示: 个人观点，仅供参考

阅读 1692

​

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

16

69

8

写留言

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论


# 走 10G 链路烧写 FPGA：远程自动更新

原创 JieL AI驱动FPGA

 _2026年7月13日 14:48_ _英国_

![封面](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqtAkwViarf3iaoyId1dVXyhj41d7IumbAl6yBHWUaDtDKFYW42RuvkP7RTSjpxPyGVg39iawWudaTfEbibhlo1lgF2ZMEicvzm3svt4/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=0)

封面

9.54 MB · 0 错

固件走链路烧进闪存，回读逐字节精确

前两篇里，AI 把这块试用板上的两颗大芯片连成了一套系统：RFSoC 跑 Linux 当主芯片，KU115 当扩展仓，中间是 10G 高速串行链路。给 KU115 这类 FPGA 灌固件，从来都是两条路：一条是把比特流直接下载进片内的配置存储器，立刻生效，但那是一块 SRAM，断电就丢；另一条是把比特流烧进旁边的 Flash 存储器，芯片每次上电自动把它加载进配置存储器，断电不丢。调试阶段走的一直是第一条，靠一根单独的 JTAG 线从工作站下载，链路调好了、算力挂上了，每次断电还得重新下载一遍。

这一篇讲 AI 怎么把第二条路补上，而且全程走链路、不碰 JTAG：固件从主芯片出发，走 10G 链路烧进 KU115 旁边的配置闪存，再发一条命令让它从闪存重启。第一条路搬不上链路：比特流要把整颗芯片重写一遍，承载链路的逻辑也会被重写，传到一半链路就断了。烧 Flash 再重启，正是远程更新的标准做法。整个过程照旧在 Python2Verilog 框架下完成，人只在关键节点给方向，外加亲手拨了一下开关。

![远程更新的完整闭环：烧写与重启都走链路](https://mmbiz.qpic.cn/mmbiz_png/57vPiach3kqsicQ2UwghjFdic9n1b8gHM13vqXaBoh4VF1ok7meKJ5mbUy23N0ISKTKvejmaVzmXJ4ib45H38edg5TPibpicBA8TuMiaNPIZWvnGiag/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=1)

远程更新的完整闭环：烧写与重启都走链路

---

## 1闪存早就焊在板上

AI 翻了底板原理图，条件其实齐了。KU115 旁边焊着一颗 64MB 的配置闪存，接在芯片的专用配置引脚上。这组引脚有个好处：设计运行起来之后，可以通过芯片内部的一个专用接口继续访问它们，不占用任何普通引脚。

于是链路协议里多了一类帧：帧头带特定标记的，进烧写通道；其余照旧走数据。烧写通道后面是一个 SPI 控制器，放在独立的自由运行时钟域里，链路抖动打不断一次正在进行的擦写。批量操作一次传 128KB，擦除、烧写、回读都是一条链路往返。

![烧写命令混在数据流里，靠帧头标记分流](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqtp794FibYd9vicAZ4JmMQia8P8x463js6psGhhnM3kl3pR4nOpQ5zrqE7hyD6HrWKRDX0icicAWZ1F8PyTnuFU4LcnWUdXbFsLe5UY/640?from=appmsg&watermark=1&tp=wxpic&wxfrom=5&wx_lazy=1#imgIndex=2)

烧写命令混在数据流里，靠帧头标记分流

真机验收三步，每步都有数字：

  

- ▶读器件识别码
    
     链路那头返回 20 BB 20，与原理图上那颗闪存的型号完全对应
    
- ▶单扇区试写
    
     高地址擦除、写入、回读、再擦除，逐字节精确
    
- ▶整个固件烧入
    
     8.56MB 写到零地址，回读比对 165 秒，0 字节不符
    

  
写路径通了，闪存里躺着一份完整固件。接下来只差让芯片上电时去读它。

  

---

## 2启动测试说不

FPGA 上电从哪里启动，由三个模式引脚决定。原理图印得清清楚楚：默认 001，就是从 SPI 闪存启动。AI 发了一条"按模式引脚重新配置"的命令，芯片却直接黑掉了，没有从闪存加载。

> ℹ Note
> 
> 001 是芯片配置手册里定义的"主 SPI"模式：芯片上电后自己当主设备，主动去读旁边的闪存。这不是板子的自定义约定，是这个芯片家族的标准启动方式之一。

AI 又去读配置状态寄存器，它说模式引脚是 000。但同一份寄存器还说 DONE 是 0，而设计明明正在运行。一个自相矛盾的寄存器，什么也证明不了。上一篇的教训在这里又用了一次：记录会骗人，行为不会。重新配置的行为已经给出了答案，现在的接法就不是从闪存启动。

那模式引脚到底接在哪？AI 回到图纸，用几何配对把那一小块电路抠出来：

![模式引脚接在 SW1 拨码开关上](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

模式引脚接在 SW1 拨码开关上

三个模式引脚全接在一个四位拨码开关上，2、3、4 号位各管一个，图纸角落印着那行"默认 001"。板子出厂时没拨到图纸说的默认位置。

  

> 目标 001 和现状 000 只差最低那一位，所以只需要拨 4 号位，而且不用管开关的方向定义：怎么拨都是取反。

  

---

## 3拨一下，芯片活了

开关拨完，AI 再发一次重新配置命令，这次芯片自己完成了配置。但"配置完成"还不够，得确认它跑的是闪存里那份，而不是之前 JTAG 灌进去的残留：

![怎么确认是从闪存启动的：一个全新的计数器](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

怎么确认是从闪存启动的：一个全新的计数器

设计里有个计数器记录 SPI 接口发过的字节数。读出来是 64，恰好等于上电预热时固定发出的字节数，一个字节不多。计数器是全新的，这是一次从零开始的加载，固件来自闪存。

然后把 JTAG 也拿掉：AI 从主芯片发一条寄存器写，让 KU115 自己重启：

![远程重启的验收结果](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

远程重启的验收结果

回执先从链路发出来，芯片再通过内部的重新配置接口（ICAP）从闪存重启，几秒后链路自动重锁，控制寄存器和闪存识别码都在新系统上应答。从这以后，给 KU115 换固件就是一次远程更新：主芯片把新固件烧进闪存，发一条重启命令，断电重来它也自动从闪存加载。

还走 JTAG 的只剩在线调试一件事。这件事同样有厂商定义的"虚拟 JTAG 线"协议：调试工具把网络当成 JTAG 线用，流量可以经任意通道转发进芯片。把它也搬上这条 10G 链路是排好的下一步，到那时这根线就彻底不用了。

---

## 4链路加宽之后，再来一遍

这套流程后来在更宽的链路上又验了一次。上一篇结尾提过，两条 10G 线可以捆绑成一条通道；这次把芯片内部的数据通路也加宽到 128 位，让捆绑的带宽真正用起来：

![双线捆绑后的实测吞吐](data:image/svg+xml,%3C%3Fxml%20version='1.0'%20encoding='UTF-8'%3F%3E%3Csvg%20width='1px'%20height='1px'%20viewBox='0%200%201%201'%20version='1.1'%20xmlns='http://www.w3.org/2000/svg'%20xmlns:xlink='http://www.w3.org/1999/xlink'%3E%3Ctitle%3E%3C/title%3E%3Cg%20stroke='none'%20stroke-width='1'%20fill='none'%20fill-rule='evenodd'%20fill-opacity='0'%3E%3Cg%20transform='translate\(-249.000000,%20-126.000000\)'%20fill='%23FFFFFF'%3E%3Crect%20x='249'%20y='126'%20width='1'%20height='1'%3E%3C/rect%3E%3C/g%3E%3C/g%3E%3C/svg%3E)

双线捆绑后的实测吞吐

同一批 256KB 回环，从 3.0 Gbps 涨到 5.24 Gbps。链路侧的用户带宽上限从 9.2 提到 18.6 Gbps，不再是瓶颈，当前实测卡在芯片内部的数据通路上，这是下一步的余量。

固件流程在捆绑链路上完整重跑：9.54MB 的新版固件，擦除 146 个扇区、烧写、回读校验 0 错，全程 295 秒，随后远程重启，所有端点在新系统上应答。烧写和重启这两件事，现在跑在它们自己烧出来的系统之上。

---

## 5留下的准则

这一轮写回 Python2Verilog 框架的检查有三条：

- ▶启动模式只信行为
    
     状态寄存器在运行中的设计上读出 DONE 为 0，这种来源不能用来判断任何事；想知道答案，就真发一次重新配置命令
    
- ▶回执先走，动作后发
    
     远程重启这类"执行完就失联"的命令，必须先把回执发出链路再动手，否则成功和失败在主芯片看来是同一种沉默
    
- ▶启动来源要有零点
    
     判断芯片跑的是哪份固件，看一个必然清零的计数器，而不是看配置是否完成
    

上一篇的结尾说，接线关系从此是测出来的，不是假设出来的。这一篇把同样的话送给启动方式：图纸说默认 001，寄存器说 000，都不算数，芯片真的从闪存启动起来那一刻才算数。

  

往期文章：

[AI 调通两块 FPGA 之间的 10G 链路：一条错误记录骗过了所有仿真](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484801&idx=1&sn=487289bae5578b681e119a7d7ff148d6&scene=21#wechat_redirect)

[AI 用一天, 把 Linux 跑上了我们这块 RFSoC 试用板](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484791&idx=1&sn=49adcd500826c3d05b766549dbae970d&scene=21#wechat_redirect)

[用 AI 自动生成一台 5G 接收机：从空口信号里读出小区信息](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484784&idx=1&sn=5d1a4e4a767195ad626e404d9bf0d0a7&scene=21#wechat_redirect)

[实测 Fable 5 比 Opus 4.8 强多少](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484727&idx=1&sn=d2d29e791e2efb3146998126e3536546&payreadticket=HEgos1S-uSY3cSx3RW3ZwdUYmuKeqTsFTRkD5QjZkQPCxYEj9ZuAH8gVMrpWj_9SL3p8pPY&scene=21#wechat_redirect)

[AI 什么都会了，我们还要教孩子什么能力？](https://mp.weixin.qq.com/s?__biz=MzE5OTEzNDczOA==&mid=2247484796&idx=1&sn=698ee81c649f88a1b95c1e23547b1e63&scene=21#wechat_redirect)

---

_烧写通道 RTL、上位机工具与全部排查均由 AI 在 Python2Verilog 框架下完成，所有数字来自当次会话的工具报告与真机测量。_

喜欢作者

 创作不易，感谢赞赏☕ 

作者提示: 个人观点，仅供参考

阅读 1987

​

[](javacript:;)

![](https://mmbiz.qpic.cn/sz_mmbiz_png/57vPiach3kqukg1VtS80MxXeOUolTk9YapPu5u5pWMB5AjL9M4jZQ7eLKic3GAxsGHmQ6hAxqFQj4Y7ueweXJt815iaEoSS1qf8c8lARyKLu3k/300?wx_fmt=png&wxfrom=18)

AI驱动FPGA

14

89

6

写留言

![](https://wx.qlogo.cn/mmopen/DhduwiaBa7ldeC7iaxnDNYgbWRTDcKnfp1YAQY68saxzDibMU507aFHhiaqxfeQEAVkK2iaOKojfPDwtZSRcDBZYU0MN70yUm3JEM/96)

复制搜一搜

复制搜一搜

暂无评论