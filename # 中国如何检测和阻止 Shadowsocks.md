o画# 中国如何检测和阻止 Shadowsocks

  

**作者：**匿名者、匿名者、匿名者、David Fifield、Amir Houmansadr

日期：2019 年 12 月 29 日，星期日

[中文版：Shadowsocks是如何被检测和封锁的](https://gfw.report/blog/gfw_shadowsocks/zh.html)

[_Shadowsocks_](https://shadowsocks.org/en/)是中国最流行的规避工具之一。自 2019 年 5 月以来，已有大量关于中国用户屏蔽 Shadowsocks 的轶事报道。本报告包含有关中国防火墙 (GFW) 如何检测和阻止 Shadowsocks 及其变体的初步研究结果。通过测量实验，我们发现 GFW**被动监测网络**中可能是 Shadowsocks 的可疑连接，然后**主动探测**相应的服务器以测试其猜测是否正确。Shadowsocks 的封锁很可能**受到人为因素的控制，这些因素**在政治敏感时期增加了封锁的严重性。我们建议一种**解决方法**——在 Shadowsocks 握手期间改变网络数据包的大小——这（目前）有效地减轻了对 Shadowsocks 服务器的主动探测。我们将继续与开发人员合作，使 Shadowsocks 和相关工具更能抵抗阻塞。

## 主要发现[](https://gfw.report/blog/gfw_shadowsocks/#main-findings)

-   长城防火墙 (GFW) 已开始使用**主动探测**来识别 Shadowsocks 服务器。GFW 结合了被动和主动检测：首先它监视网络中可能是 Shadowsocks 的连接，然后将它自己的探测发送到服务器（就好像它是另一个用户一样）以确认它的猜测。众所周知，GFW 会对各种规避工具进行主动探测，现在 Shadowsocks 也是该组织的成员。
-   有源探测系统发送多种探测类型。有些是基于之前记录的真实 Shadowsocks 连接的**回放，而另一些则与之前的连接没有明显关系。**
-   与之前的研究一样，主动探测来自中国**不同的源 IP 地址**，因此很难被过滤掉。同样与之前的研究一样，网络侧信道证据表明，这数千个明显的探测器不是独立的，而是集中控制的。
-   只有少量真正的客户端连接（超过 13 个）足以触发对 Shadowsocks 服务器的主动探测。只要合法​​客户端尝试连接到它，服务器就会继续被探测。第一个重放探测通常在真正的客户端连接后几秒钟内到达。
-   一旦主动探测识别出 Shadowsocks 服务器，GFW 可能会通过丢弃服务器未来发送的数据包来阻止它——无论是来自特定端口还是来自服务器 IP 地址上的所有端口。或者服务器可能**不会**立即被阻止，尽管被探测。在政治敏感时期，Shadowsocks 服务器的屏蔽程度很可能受到一些人为因素的控制。
-   防火墙对可疑连接的初始被动监控至少部分基于网络数据包的大小。修改数据包大小，例如通过在 Shadowsocks 服务器上安装[_brdgrd_](https://github.com/NullHypothesis/brdgrd)，通过中断分类的第一步来显着减轻主动探测。

## 我们怎么知道呢？[](https://gfw.report/blog/gfw_shadowsocks/#how-do-we-know-this)

我们建立了自己的 Shadowsocks 服务器并从中国内部连接到它们，同时捕获双方的流量以进行分析。所有实验均在 2019 年 7 月 5 日至 2019 年 11 月 11 日期间进行。大部分实验是[_在报道从 2019 年 9 月 16 日开始大规模封锁 Shadowsocks 之后_](https://github.com/net4people/bbs/issues/16)进行的。

在大多数实验中，我们使用[_shadowsocks-libev_](https://github.com/shadowsocks/shadowsocks-libev) [_v3.3.1_](https://github.com/shadowsocks/shadowsocks-libev/tree/v3.3.1)作为客户端和服务器，因为它是一个积极维护且具有代表性的 Shadowsocks 实现。我们相信我们发现的漏洞适用于许多 Shadowsocks 实现及其变体，包括[_OutlineVPN_](https://getoutline.org/)。

除非明确指定，否则所有客户端和服务器的使用都不会对其网络功能进行任何修改，例如防火墙规则。Shadowsocks 可以配置不同的加密设置。我们测试了同时运行 Stream 密码和 AEAD 密码的服务器。

## 关于有源探针的详细信息[](https://gfw.report/blog/gfw_shadowsocks/#details-about-active-probes)

Shadowsocks 是一种加密协议，其设计目的是在数据包内容中没有任何静态模式。它有两种主要的操作模式，均由主密码键入：[_Stream_](https://shadowsocks.org/en/spec/Stream-Ciphers.html)（不推荐）和[_AEAD_](https://shadowsocks.org/en/spec/AEAD-Ciphers.html)（推荐）。这两种模式都是要求客户端在使用服务器之前知道主密码；但是在 Stream 模式下，客户端仅经过弱身份验证。除非采取单独的措施来防止重放，否则这两种模式都容易重放以前看到的经过身份验证的数据包。

### 探测有效载荷类型和审查员的意图[](https://gfw.report/blog/gfw_shadowsocks/#probe-payload-types-and-censors-intentions)

我们观察到 5 种类型的有源探针：

基于重放：

1.  相同的重放（合法连接中第一个携带数据的数据包）；
2.  重放字节 0 改变；
3.  重放字节 0–7 和 62–63 更改；

看似随机的（不是我们可以识别的任何真实联系的重播）：

4.  长度为 7-50 字节的探测，约占随机探测的 70%；
5.  长度正好为 221 字节的探测，约占随机探测的 30%。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image1.png "CDF：Outline 服务器接收到的 PSH/ACK 的有效载荷长度")

我们怀疑主动探测系统通过比较服务器对其中几个探测的响应来识别 Shadowsocks 服务器及其变体。

Shadowsocks-libev 有一个[_回放过滤器_](https://github.com/shadowsocks/shadowsocks-org/issues/44)；但是大多数其他 Shadowsocks 实现都没有。回放过滤器仅阻止精确回放，而不阻止已修改的回放，并且其本身不足以阻止主动探测将响应与几个稍微不同的探测进行比较。

### 触发主动探测需要多少个连接？[](https://gfw.report/blog/gfw_shadowsocks/#how-many-connections-are-required-to-trigger-active-probing)

似乎需要一定阈值的真正同时连接才能触发主动探测。例如，在一项实验中，只有 13 个连接就足以触发主动探测。初步结果还表明，使用 AEAD 密码的 Shadowsocks 服务器可能需要稍微多一点的连接才能被探测到。

### 真正的连接和主动探测之间的关系[](https://gfw.report/blog/gfw_shadowsocks/#relationship-between-genuine-connections-and-active-probings)

我们让客户端每 5 分钟与 Shadowsocks 服务器建立 16 个连接。尽管我们的连接触发了大量主动探测，但 Shadowsocks 服务器从未被阻塞，原因我们并不完全了解。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image6.png "跨时间接收的 SYN 数")

上图显示，当合法客户端尝试连接服务器时，它会收到主动探测；当他们停止尝试连接时，主动探测大多停止。每个合法连接发送的活动探测数是可变的，而不是 1:1。

### 重放攻击延迟[](https://gfw.report/blog/gfw_shadowsocks/#delay-of-replay-attacks)

主动探测系统可以保存真正的连接有效负载并在以后重放它，甚至响应单独的未来连接。下图显示了合法连接和随后的基于重放的探测之间的延迟变化。因为一个合法连接可能会导致许多（最多 47 个）重放攻击，我们提出了两种不同的情况：橙色线仅对特定合法连接的第一个基于重放的探测进行采样；蓝线是所有基于回放的探针的样本。

结果表明，超过 90% 的重放探测是在合法客户端连接后的一小时内发送的。观察到的最小延迟为 0.4 秒，而最大延迟约为 400 小时。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image2.png "CDF：基于重放的探测延迟")

### 探针的起源[](https://gfw.report/blog/gfw_shadowsocks/#origin-of-the-probes)

在我们迄今为止进行的所有实验中，我们已经看到来自**10,547 个**唯一 IP 地址的 35,477 个主动探测都属于中国。

**起源 AS。**占 Shadowsocks 探针大部分的两个自治系统，AS 4837 (CHINA169-BACKBONE CNCGROUP China169 Backbone,CN) 和 AS 4134 (CHINANET-BACKBONE No.31, Jin-rong Street, CN)，与之前的相同记录在以前的工作中。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image3.png "所有实验中唯一探测 IP 的 ASN")

**集中式结构。**尽管来自数千个唯一 IP 地址，但似乎所有主动探测行为仅由少数进程集中管理。这一观察的证据来自网络侧渠道。下图显示了[_TCP 时间戳_](https://tools.ietf.org/html/rfc7323#section-3)附加到每个探针的 SYN 段的值。TCP 时间戳是一个以固定速率增加的 32 位计数器。它不是一个绝对时间戳，而是相对于 TCP 实现在操作系统上次启动时的初始化。该图显示，最初看起来有数千个独立探测器实际上只共享少量线性 TCP 时间戳序列。在这种情况下，至少有九个不同的物理系统或进程，其中九个中的一个占绝大多数的探测。我们说“至少”九个进程是因为我们可能无法区分两个或多个共享非常接近的拦截值的独立进程。序列的斜率表示 250 Hz 的时间戳增量频率。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image5.png "来自 Probers 的 SYN 段的 TCP TSval")

## 我们如何绕过阻塞？[](https://gfw.report/blog/gfw_shadowsocks/#how-can-we-circumvent-the-blocking)

Shadowsocks 的检测分两步进行：

1.  被动识别可疑的 Shadowsocks 连接。
2.  主动探测可疑连接的服务器。

因此，为避免阻塞，您可以 (1) 避开无源探测器，或 (2) 以不会导致阻塞的方式响应有源探测器。我们将通过安装改变数据包大小的软件来展示如何做 (1)。

[_Brdgrd_](https://github.com/NullHypothesis/brdgrd) 是您可以在 Shadowsocks 服务器上运行的软件，它会导致客户端将其 Shadowsocks 握手分解为更小的数据包。它最初的目的是通过强制 GFW 进行复杂的 TCP 重组来破坏 Tor 中继的检测，但这里我们利用了 brdgrd 对从客户端到服务器的数据包大小的整形。似乎 GFW 至少部分依赖数据包大小来被动检测 Shadowsocks 连接。修改数据包大小可以通过破坏分类的第一步来显着减轻主动探测。

![“找不到图片”](https://gfw.report/blog/gfw_shadowsocks/images/image4.png "brdgrd 在服务器上的作用")

该图显示了正在进行主动探测的 Shadowsocks 服务器，然后在激活 brdgrd 的几个小时内探测到零。一旦我们禁用了 brdgrd，主动探测就会恢复。我们第二次启用 brdgrd 时，探测完全停止了大约 40 小时，但随后又出现了一些探测。

另一个实验表明，如果在第一次探测服务器之前从一开始就使用 brdgrd 可能会更有效。

Brdgrd 通过将服务器的 TCP 窗口大小重写为一个很少小的值来工作。因此，很可能检测到正在使用 brdgrd。所以虽然brdgrd暂时可以有效减少主动探测，但不能算是解决Shadowsocks阻塞的永久解决方案。

## 未解决的问题[](https://gfw.report/blog/gfw_shadowsocks/#unresolved-questions)

虽然主动探测发生的事实很清楚，但我们仍然不清楚主动探测如何影响 Shadowsocks 服务器的阻塞。也就是说，我们在全球拥有 33 台 Shadowsocks 服务器。虽然它们中的大多数都经历过大量的主动探测，但其中只有 3 个曾经被阻止。更有趣的是，其中一个被阻塞的服务器只使用了很短的时间，因此没有像其他一些没有被阻塞的服务器那样收到那么多的探测。

我们提出了三个假设，试图解释这个有趣的现象：

-   Shadowsocks 服务器的阻塞可能是由一些人为因素控制的。也就是说，GFW 可能会维护一份高度可疑的 Shadowsocks 服务器列表，而已知服务器是否被阻止（或未阻止）取决于人为因素。这一假设也可以部分解释为什么在政治敏感时期报告了更多的封锁。
    
-   另一个假设是，主动探测对我们在大多数实验中使用的特定 Shadowsocks 实现无效。事实上，所有三台被阻塞的服务器都在运行与其他服务器不同的实现。如果 GFW 一直在利用一些独特的服务器反应，这些反应只是特定的 Shadowsocks 实现集的特征，则可能会出现这种情况。
    
-   第三个假设是审查中存在一些地理位置不一致。被封锁的所有三台服务器都在与其他服务器不同的数据中心中运行，并从不同的住宅网络连接。如果 GFW 特别注意属于某些已知数据中心的地址范围，和/或特别注意来自住宅网络的连接，则可能会出现这种情况。
    

## 谢谢[](https://gfw.report/blog/gfw_shadowsocks/#thanks)

我们要感谢这些人对此主题的研究和有益的讨论：

-   Shadowsocks-libev 开发者
-   Vinicius Fortuna 和 Jigsaw 的 Outline VPN 开发人员
-   Eric Wustrow 和来自 CU Boulder 的许多其他研究人员

## 联系人[](https://gfw.report/blog/gfw_shadowsocks/#contacts)

该报告首次出现在[GFW Report](https://gfw.report/blog/gfw_shadowsocks)上。[我们还在net4people](https://github.com/net4people/bbs/issues/22)和[ntc.party](https://ntc.party/t/how-china-detects-and-blocks-shadowsocks/289)上维护报告的最新副本。

我们鼓励您公开或私下分享您对我们的发现和假设的问题、评论或证据。我们的私人联系信息可以在[GFW 报告](https://gfw.report/)的页脚找到。