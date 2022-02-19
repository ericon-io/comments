---
title: "为啥 Paxos 这么难？"
author: xxchan
categories: xxchan
tags:
  - 分布式共识
  - 分布式系统
---

本文中，我将**忽略性能等实际问题**，主要聊点 general 的东西，如基本概念、表达力，或者说“怎么把它变为可能”，而不是“怎么把它搞得快的飞起”。另外，我会尽量写的让不了解分布式系统或共识的读者也能看懂。不过，如果你已经对 Raft 或 Paxos 之类的共识算法有了概念，那当然还是更好。

<!-- more -->

## 我学习共识的经历

[这个知乎回答：有没有哪一刻让你感受到了文献作者的恶意？](https://www.zhihu.com/question/65038346/answer/227674428)是我第一次听说 Paxos 和共识。当时看了感觉贼逗，同时也就在脑子里留下了 “Paxos 很难” 的初印象。

正好一年前，我在写 MIT 6.824 的 Raft lab。读 Raft 的时候我感觉它挺好懂的，把东西拆成了模块，把每个模块解释得都很清楚。里面还有一章 "What's wrong with Paxos?" ，我读的时候又情不自禁地乐了起来。总之，虽然我在 debug 的时候还是有点小困难的，但学 Raft 的认知过程算是很平滑。

上学期，我有个作业要实现 Paxos。我碰巧收藏过[这个博客](https://blog.openacid.com/algo/paxos/)，就拿出来看看。它用很简单的方式解释了 Paxos，我也就按照它讲的实现了（basic）Paxos，没再看论文或者别的文章。作业还要求我们用一个特定的方案实现 multiple decisions，所以我也没看 multi-Paxos 之类的东西。

上学期我还上了 Concurrent Algorithms 和 Distributed Algorithms 两门课。它们都比较理论，都讲了共识，我学的时候感觉共识这东西挺简单的。DA 课上的共识算法很像 Paxos，而且简单到一张 slide 就够了。另外 CA 课上的 Universal Construction 以及 DA 课上的 Total Order Broadcast 算法，这两个东西可以说证明了共识基本上可以用来做任何事情，而且这两个算法也不难理解。然后我就开始想，为什么人们会觉得 Paxos 难理解，这也是我想写这篇博客的原因。

最后，我读了 Paxos 的两篇论文，确认了它们真的都挺好懂的。但 Lamport 的语言太有意思了。*The Part-Time Parliament* 讲的特别详细又很直白，就是仿佛把名词术语做了一下简单的映射。*Paxos Made Simple* 则是用平实的语言谈直觉理解，它倒是有点像一篇博客。

话不多说，我们还是进入正题吧。

## 各种抽象：Consensus, Replicated Log 和 Replicated State Machine

首先，让我们先不管具体的共识算法，先来看看它的抽象（或接口）是怎么回事。

在分布式系统中，我们通常想要 *share* 东西。那么 shared objects 或叫 shared data structures 就很有用。它就像单机数据结构一样（像 queue, map），但可以被不同的机器（或线程）访问。也可以说是多台机器共同维护一个全局对象，每台机器复制了全局对象的状态。

Replicated state machine 可以说是最强的抽象了，算是所有 shared data structures 的推广。(或者也可以说 shared data structures 都可以用 replicated state machine 来实现）。你可能很容易想到，维护一个 replicated log 就是一种实现 replicated state machine 的自然的方法。完全没错。

顺便一提，replicated log 也可以等价成另一个抽象 total order broadcast：每个机器广播消息，保证所有人以相同的全局顺序收到消息。(很像，不过无所谓，不是很重要。）

那么什么是共识呢？最原始的 consensus 的定义来了。首先，它也就是一个 shared objects。它的 *sequential specification*（如果它被不同的机器按顺序访问（好比加了锁），怎么工作）是：

```
维护一个变量：prop
初始值: ⊥ (未初始化)

propose(v):
    if prop == ⊥:
        prop = v
    return prop
```

是不是很简单？有几个注意点：
- 它是一次性的，single decision。它可以被看作是一个 write-once variable。
- 这里用了同步风格的接口，用基于事件的异步接口也行。
- 通常假设每个机器都有东西要 propose，而 propose 是获得最终的 decision 值的唯一途径。我认为这种约定俗成只是为了简单起见。你可以很容易地修改接口，改成不用 propose 也能知道值，就像 Paxos 里分了 proposer, acceptor 和 learner，本质上其实一样，不太重要。  

我们也可以等价地通过给出它的性质来描述 consensus：
- Validity: A decided value is proposed.
- Agreement: No processes decide differently.
- Termination: Every process eventually decides.

## 用 Consensus 实现 Replicated State Machine

Consensus 是一个如此简单的抽象，但它足够强大，可以实现 replicated state machine，也就是说，它可以用来实现任何 shared object。这个结论有一个响亮的名字: **Universality of Consensus**。

首先我们可以考虑一个 naïve 的方案：consensus 已经算是个单条 replicated log entry 了，那为什么不直接用一个 consensus list 作为 replicate log 呢？当一台机器要 append entry 的时候，它就一直向前 propose，直到它的 proposal 在一个 consensus instance 中被 decide。

这个想法基本上差不多可行了，除了一个关键问题：可能会有人一直 propose 赢不了，也就是会 starve。解决方案也不是很复杂：现在，每个人不仅 propose 自己的请求，而且还**帮助**别人。具体来说，当一个新的想要 append 的 entry 出现时，先把它广播出去（同时每个人一直在收集别人的请求）。然后，每次 propose **所有收集到的 pending entries**（而不是一条 entry）。这就成了!

当然，这不是唯一的方法，而且没啥优化，不过完全 work！

其实在看这个算法之前你可能就觉得这个事儿肯定是能做到的。我就是展示了有一个简单的方法，让我们相信确实可以做到。我觉得从 consensus 到 replicated log 就好像从手动档切换到自动档。

## 为啥 Paxos 这么难？

如果你知道 Paxos 的工作原理，你可能已经发现它的核心算法就是实现了共识。许多复杂度来自于把它变成 replicated log。在论文中似乎没有明确说明这个关系（虽然我没细读这部分）。但是，脑子里带着上文的算法和关于 consensus 的知识，并忽略各种实现细节、性能问题（尽管这在工程师的思维中可能很难），你就可以说服自己，它确实 work，而且很简单。All you need is (single-decision) consensus! 换句话说，如果你已经知道了 consensus 这个抽象，然后想找一个实现，你立刻就能搞懂 Paxos 在干嘛。所以，复杂的不是 Paxos 算法本身，而是有很多概念混在一起没搞清楚。

还不了解 Paxos 的读者，可以回去读一读这篇论文，现在可以只关注 single decision 的部分了，不用头疼 multiple decision！不过我还是在这里简单地解释一下 Paxos。

[之前提到的 xp 的博客](https://blog.openacid.com/algo/paxos/)很清晰易懂，因为它也是主要专注讲了简单的东西，没有讲的太多把读者搞晕。但我认为这篇博客中的想法可以再进一步概括一下。

首先，slide 1-5 相当于是引入我们需要 (single-decision) consensus 这个抽象。然后讨论了许多失败的尝试（但都还挺重要的技术）。
- slide 6-9: Single writer *all-ack*. 不允许任何 crash。
- slide 10-12: Single writer majority-ack（或者叫 *majority-write*）。为了让这个方法 work，*Timestamp* 和 *majority-read* 也同时需要。这个算法不允许 writer crash。
- slide 14-17: Majority-write 也不能推广到 **multiple writers**.
- slide 18-19: (majority) Read before (majority) write.
- slide 20: Request (majority) promise before (majority) write.

没错，我认为 Paxos 的思想可以概括为一句话：**request promise before write**！它就是这么简单，不到 200 行代码就够实现了。

xp 以一种简单的方式告诉你 Paxos 是*如何*工作的，但你可能仍然想知道*为什么*我们要这么干，以及这么干是不是真的稳了。现在我应该说，在 Paxos 论文中，Lamport 是（从他想保证的性质中）**推导**出了这个想法。因此，如果你在已经了解了 Paxos 之后再重新阅读这篇论文，你可能就会发现它相当容易理解，而且有很精彩的推理和直觉。但也许恐怕这也是很多人一开始没能理解的原因，因为他们不习惯这种思维方式，在推导为什么 work 之前，就是想先知道它到底是具体怎么 work 的。

顺便一提，xp 的后续博客[用200行代码实现基于paxos的kv存储](https://drmingdrmer.github.io/algo/2020/10/28/paxoskv.html)实现了（single-decision）Paxos，然后手动管理 consensus instance。（还有一个更复杂的后续博客在[这里](https://blog.openacid.com/algo/mmp3/)。）所以你也可以通过看这个实现来学习（single-decision）Paxos。


## Raft 更简单吗？

让我们回头再看一下Raft。首先，"What's wrong with Paxos?" 部分。

> We hypothesize that Paxos' opaqueness derives from its choice of the single-decree subset as its foundation. Single-decree Paxos is dense and subtle: it is divided into two stages that do not have simple intuitive explanations and cannot be understood independently.

我现在觉得这个评价有点 **unfair**。Single-decision consensus 是分布式计算理论中一个成熟的抽象。Single-decree Paxos 也有很好的直觉在里面。

---

另一个评价是，Paxos 没有为在 *real-world* systems 中建立 *practical* 的实现提供良好的基础，这可能确实。然而，我认为这对 Paxos 来说根本就是个 **non-goal**。它就是在讲解决一个理论问题的一种优雅的算法。它就根本**不太关心**你在现实世界中如何实现它……我认为 Lamport 也是一个比较偏理论的 researcher。(他建立了分布式计算理论的基础。）现实世界的东东对 theory researchers 可能没有那么大的吸引力，如果问题在理论上没啥意思的话……

---

现在让我们来看看 Raft 本身。我也将尝试用几句话来描述它。

首先，它谈了 replicated state machine，并且**明确地告诉你我们正在造 replicated log**。工程师可能对 consensus 不熟悉，但对 log 一定非常了解。

第二，Raft 选择了 **leader-based** 方法：只有 leader 可以与外界交互以及 append entry。Paxos 则完全是 **peer-to-peer**。有一个 leader 通常会使主要流程或 “happy path” 更容易理解。Raft 自然地选择了 majority-write。那么唯一的问题是如何处理 leader crash。Raft 的方法是：限制节点不给 outdated 节点投票。这个技术很简单，但也需要花些功夫去理解它为什么 work，因为其实里面涵盖了很多 edge cases。要维护的性质其实是，新的 leader 必须*至少*包含所有 majority-written entries，但是否包含更多的 pending entries 则无所谓。

最后，replicated log 里会存在不一致的状态，leader 通过把它覆盖掉来修复。

所以 Raft 简单吗？它的核心思想也不难。它也就是 single-leader majority-write …… 再加上一些额外但也很简单的 trick（以确保 majority-write 能 work）。

简而言之，Raft
- 直接实现 replicated log。
- 谈论**具体的概念**，如 heartbeat, timeout, RPC and crash recovery，而不是像 theory researcher 一样谈论抽象概念。
- 谈论所有路径中的**所有细节**（状态转换），并且算是用一种 manageable way **暴露了复杂度**。(Paxos 不关心这些 **trivil** 的实现复杂度)。

所以读完 Raft 后，你基本上可以直接**翻译成代码**。相对的，Paxos 在理论上很简单，读完后，你可能会思考一阵子，然后大叫一声：**哦，它确实 work! 太简单了！**（就像一个数学家一样。）那也就难怪大多数人，尤其是工程师，会觉得 Raft 读起来更舒服。

## 更多关于 Consensus 的话题

关于 consensus，还有更多有趣的理论问题。除了 universality 之外，最重要的问题可能是共识的 **impossibility**！具体来说，FLP impossibility 说：在一个异步的分布式系统中，如果可能有一个进程 crash，共识是不可能实现的。

啊？那 Paxos 和 Raft 在干嘛？

*Don't panic.* 这里的**异步**系统是指：每个进程可以被 delay *任意长*的时间，而我们对别人是否活着一无所知。但在实践中，我们其实有一些（关于**时间**的）假设。现实世界更像是一个**最终同步的**（eventually synchronous）系统，这意味着我们可以有一个 eventually perfect failure detector：它可能会误诊把活人当死人，但是如果再多等等，它会自己改正错误的结论。其实 heartbeat 和 timeout 机制基本上就是在实现一个 eventually perfect failure detector。

另一个附带说明是，实际上，我们在上面假设了通过 message-passing 进行通信。如果我们切换到 shared memory，我们仍然会有 FLP impossibility。准确地说，只使用 **shared registers** 是不可能实现共识的。

其实 FLP impossibility 的证明也非常精彩。其核心思想是：首先，任何共识算法的执行都必须有一个**临界时间点**（critical timing）。从上帝视角看，整个系统当前并没有达成 decision，但立刻即将（在任何人执行一小步以后）达成 decision。所有后来的步骤都只是在走流程告知所有人结果罢了。(如果你懂命运石之门的话，这意思就是世界线马上要收束了）。然后我们断言，所有人的下一步都想访问同一个 shared register！

为了方便继续讨论，让我们假设我们有两个进程 `p1` 和 `p2`。如果他们不访问同一个对象，以下两种 possible histories 是等价的。
1. `p1` 走一步，然后 `p2` 走一步。
2. `p2` 走一步，然后 `p1` 走一步。

这说明这个 timing 不 critical 啊，矛盾！

现在假设他们都要访问 shared register `x`。
- Case 1：如果第一步是要读 `x`，我们在它读完以后马上把它干掉。那么另一个进程会认为一切都和原来一样啊，所以它做不出 decision！这意味着在 critical timing 必须对整个系统做出一些 **observable change** 才行。所以
- Case 2：两个进程的第一步都是写 `x`。那么以下两种 possible histories 是等价的：
  
  1. `p1` 写 `v1` 进 `x`，然后立刻被干掉。`p2` 将 `v2` 写入 `x`。
  2. `p1` 立刻被干掉。`p2` 将 `v2` 写入 `x`。
  
  同样，这与 critical timing 相矛盾！（谁先走不重要啊）


这就证完了！(我证了世界线收敛不了。)是不是也很简单？现在你可能也对为什么 impossiblity 能成立也有了点感觉。因为异步是一个太弱的假设。换句话说，它给了 *scheduler*（或者我们这些证定理的坏人）太大的权力。

更 exciting 的是，上述证明框架适用于更多情况。实际上，我们在上面证明了*2个进程*之间的共识只使用 register 是不可能的。我们同样可以证明*3个进程*之间的共识只用 register 和 queue 或 fetch-and-add 是不可能的！但是我们可以用 compare-and-swap 在任何数量的进程之间实现共识！

这些结论给了我们一种**比较 shared objects 的表达力的方法**：consensus number。Register 的 consensus number 为 1，queue 和 FAA 的为 2，CAS 的为 ∞。关于 consensus number 还有一些未解决的研究问题，但我想我们现在还是适可而止吧！

最后总结一句，分布式计算理论还是挺有趣的，值得学习一点。😄