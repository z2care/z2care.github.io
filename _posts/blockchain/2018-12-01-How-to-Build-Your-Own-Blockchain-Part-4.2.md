---
layout: post
title: "如何构建自己的区块链 第四二部分 - 以太坊工作证明难度解释"
date: 2018-12-01 22:27:00 +0800
categories: 开发与技术
tag: Blockchain
---
独立翻译自:[How to Build Your Own Blockchain Part 4.2 — Ethereum Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/21/how-to-build-your-own-blockchain-part-4-2-ethereum-proof-of-work-difficulty-explained/)
<!-- more -->
* content
{:toc}

我们又回到了工作困难范围的证明，这一次是通过以太坊的困难是如何随时间变化的。这是第4部分系列的第4.2部分，第4.1部分是关于比特币的PoW难度，接下来4.3部分是关于jbc的PoW难度。

## 长话短说

为了计算下一个以太坊块的难度，你需要计算开采前一个块所花费的时间，如果这个时间差大于目标时间，那么难度就会下降，从而使开采下一个块更快。如果它小于时间目标，那么难度就会上升，将会更快地开采下一个区块。

确定新难度有三个部分:偏移量，它决定了从一个难度到下一个难度的标准更改量;符号，决定难度是上升还是下降;和炸弹，这增加了额外的难度取决于块的数量。

对于不同的分叉、Frontier, Homestead, 和 Metropolis，这些数字的计算略有不同，但计算下一个困难的总体公式是

```
target = parent.difficulty + (offset * sign) + bomb
```

## 本文其他系列文
- [Part 1 — Creating, Storing, Syncing, Displaying, Mining, and Proving Work](https://bigishdata.com/2017/10/17/write-your-own-blockchain-part-1-creating-storing-syncing-displaying-mining-and-proving-work/)
- [Part 2 — Syncing Chains From Different Nodes](https://bigishdata.com/2017/10/27/build-your-own-blockchain-part-2-syncing-chains-from-different-nodes/)
- [Part 3 — Nodes that Mine](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
- [Part 4.1 — Bitcoin Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)

## 预注
对于下面的代码示例，这将是块的类。

```
class Block():
  def __init__(self, number, timestamp, difficulty, uncles=None):
    self.number = number
    self.timestamp = timestamp
    self.difficulty = difficulty
    self.uncles = uncles

```
我用来显示代码正确的数据是从[Etherscan](https://etherscan.io/)获取的。

该代码在计算难度时也不包括边缘情况。在大多数情况下，边缘情况不涉及计算难度目标。通过不包含它们的方式，可以使下面的代码更容易理解。我将在最后讨论边缘情况以确保它们没有被完全忽略。

最后，对于开始的分叉，我将讨论不同的变量和函数执行什么。对于后来的分叉，Homestead和Metropolis，我只谈变化。

此外，[这是我在Github repo中添加的所有代码](https://github.com/jackschultz/ethana)!如果不想自己编写所有代码，至少应该在本地克隆并运行它。如果你想添加更多的部分，或者认为我把代码格式化错了，你可以随时pull request。

## 起点-Frontier

一开始，有Frontier。我将通过一个要点部分介绍配置变量来直接开始。

- `DIFF_ADJUSTMENT_CUTOFF` — 表示以太坊挖掘一个区块的目标秒数。
- `BLOCK_DIFF_FACTOR` — 帮助计算当前难度可以更改多少。
- `EXPDIFF_PERIOD` — 表示炸弹数量更新后的块数。
- `EXPDIFF_FREE_PERIODS` — 在引入bomb到难度计算之前有多少个`EXPDIFF_PERIODS`被忽略了。

如下是函数的描述。


`calc_frontier_sign` — 计算下一个难度值应该上升还是下降。在Frontier的情况下，如果前一个block被挖掘的速度快于13秒的`DIFF_ADJUSTMENT_CUTOFF`，那么符号将是1，这意味着我们希望难度更大，从而使得下一个block被挖掘的速度更慢。如果前一个块被挖掘的时间超过13秒，那么符号将是-1，下一个难度将降低。总的来说，块挖掘时间的目标是`~12.5`秒。 看看Vitalik Buterin的文章[他谈到了在最小平均块挖掘时间选择12秒](https://blog.ethereum.org/2014/07/11/towa-12second块时间/)。

`calc_frontier_offset` — 偏移量是决定难度变化多少的值。 在Frontier的情况下，这是除以`BLOCK_DIFF_FACTOR`得到的父级困难整数。因为它被2018除尽，即2^11，如果你想要按照移位查看它的话，`offset`也能通过`parent_difficulty >> 11`计算得到。由于`1.0 / 2048 == 0.00048828125`， 它意味着offset每次变化将仅仅改变`0.0488%`的难度。没有太大，这很好，因为我们不希望每个不同的挖掘区块有很大的难度变化。但是如果时间持续低于13秒，时间就会慢慢增加。

`calc_frontier_bomb` — 即bomb. 每`EXPDIFF_PERIOD`个块被挖掘后，bomb增加的难度增大了一倍。在Frontier的世界里，这个值增加的很小。例如，一个1500000的块，bomb值是`2 ** ((1500000 // 100000) - 2) == 2 ** 15 == 32768`。块的难度是34982465665323. 这是一个巨大的差异，意味着bomb没有任何影响。这将在后面发生改变。

`calc_frontier_difficulty` — 一旦你得到了sign,offset,和bomb的值, 新的难度就是`(parent.difficulty + offset * sign) + bomb`. 假设挖掘父结点的block花费的时间是15秒。在这种情况下，难度将下降`offset * -1`, 然后在最后加入少量的bomb。如果在父级区块上开采的时间是8秒，难度将增加`offset + bomb`。

为了完全理解它，请通读下面的代码并查看计算过程。

```
config = dict(
  DIFF_ADJUSTMENT_CUTOFF=13,
  BLOCK_DIFF_FACTOR=2048,
  EXPDIFF_PERIOD=100000,
  EXPDIFF_FREE_PERIODS=2,
)

def calc_frontier_offset(parent_difficulty):
  offset = parent_difficulty // config['BLOCK_DIFF_FACTOR']
  return offset

def calc_frontier_sign(parent_timestamp, child_timestamp):
  time_diff = child_timestamp - parent_timestamp
  if time_diff < config['DIFF_ADJUSTMENT_CUTOFF']:
    sign = 1
  else:
    sign = -1
  return sign

def calc_frontier_bomb(parent_number):
  period_count = (parent.number + 1) // config['EXPDIFF_PERIOD']
  period_count -= config['EXPDIFF_FREE_PERIODS']
  bomb = 2**(period_count)
  return bomb

def calc_frontier_difficulty(parent, child_timestamp):
  offset = calc_frontier_offset(parent.difficulty)
  sign = calc_frontier_sign(parent.timestamp, child_timestamp)
  bomb = calc_frontier_bomb(parent.number)
  target = (parent.difficulty + offset * sign) + bomb
  return offset, sign, bomb, target
```

## 中场 — Homestead

这个Homestead分叉, 是在[块号1150000](https://etherscan.io/block/1150000) 于2016年3月, 在计算`sign`上面有几个大的变化。


`calc_homestead_sign` — Homestead分叉不同之处是使用`DIFF_ADJUSTMENT_CUTOFF`，而不是一个单独的数字，它会增加或减少难度，Homestead采用了一种稍微不同的方法。 如果你看一下代码，你会发现符号有分组，而不只是1或-1。如果祖父级和父级之间的time_diff在[0,9]之间，则符号将为1，这意味着需要增加难度。如果time_diff是在[10,19]之间, 则符号将为0，这意味着难度应该保持不变。 如果time_diff在[20,29]范围内，则符号变为-1。如果time_diff在[30,39]范围内，则符号为-2，以此类推。

这有两种作用。首先，Homestead不想把一个50秒挖的区块和一个15秒挖的区块等同起来。如果它挖了一个块用50秒，那么下一个困难实际上需要更容易一些。其次，与表示目标时间的`DIFF_ADJUSTMENT_CUTOFF`不同，它将目标点转换为`time_diffs`范围的中点，符号为0。[10, 11, 12, 13, 14, 15, 16, 17, 18, 19]。意味着大概是 `~14.5`秒，不包括bomb。

```
config = dict(
  HOMESTEAD_DIFF_ADJUSTMENT_CUTOFF=10,
  BLOCK_DIFF_FACTOR=2048,
  EXPDIFF_PERIOD=100000,
  EXPDIFF_FREE_PERIODS=2,
)

def calc_homestead_offset(parent_difficulty):
  offset = parent_difficulty // config['BLOCK_DIFF_FACTOR']
  return offset

def calc_homestead_sign(parent_timestamp, child_timestamp):
  time_diff = child_timestamp - parent_timestamp
  sign = 1 - (time_diff // config['HOMESTEAD_DIFF_ADJUSTMENT_CUTOFF'])
  return sign

def calc_homestead_bomb(parent_number):
  period_count = (parent_number + 1) // config['EXPDIFF_PERIOD'] # or parent.number + 1 >> 11 if you like bit notation
  period_count -= config['EXPDIFF_FREE_PERIODS']
  bomb = 2**(period_count)
  return bomb

def calc_homestead_difficulty(parent, child_timestamp):
  offset = calc_homestead_offset(parent.difficulty)
  sign = calc_homestead_sign(parent.timestamp, child_timestamp)
  bomb = calc_homestead_bomb(parent.number)
  target = (parent.difficulty + offset * sign) + bomb
  return offset, sign, bomb, target
```

## 当前 — Metropolis
Homestead有几个不同之处。首先，`DIFF_ADJUSTMENT_CUTOFF`现在是9，这意味着，如果没有叔节点，目标块时间是[9、10、11、12、13、14、15、16、17]的中点或即`~13`秒。

第二种考虑是否有叔节点包含在块中。在以太坊语言中，叔节点指的是两个节点从同一个祖父级处挖掘出一个子节点的时间点。所以，如果你从一个有“兄弟姐妹”的父级节点那里挖掘一个子块，你可以从中选择兄弟姐妹中的一个进行挖掘，但也包括你通知到的其他块。在这种情况下，以太坊想要加大新的难度，买入另一个offset，以确保两个天然叉获得更长时间的可能性更小。

现在最大的不同是如何消除bomb值的影响。查看`calc_metropolis_bomb`的代码，在这里我们不仅要减去`EXPDIFF_FREE_PERIODS`的值，还要减去`METROPOLIS_DELAY_PERIODS`的值，后者是30个时间段。一个巨大的数字。这里不讨论bombs，在这之后我会有一个专门的章节。

```
config = dict(
  METROPOLIS_DIFF_ADJUSTMENT_CUTOFF=9,
  BLOCK_DIFF_FACTOR=2048,
  EXPDIFF_PERIOD=100000,
  EXPDIFF_FREE_PERIODS=2,
  METROPOLIS_DELAY_PERIODS=30,
)

def calc_metropolis_offset(parent_difficulty):
  offset = parent_difficulty // config['BLOCK_DIFF_FACTOR']
  return offset

def calc_metropolis_sign(parent_timestamp, child_timestamp):
  if parent.uncles:
    uncles = 2
  else:
    uncles = 1
  time_diff = child_timestamp - parent_timestamp
  sign = uncles - (time_diff // config['METROPOLIS_DIFF_ADJUSTMENT_CUTOFF'])
  return sign

def calc_metropolis_bomb(parent_number):
  period_count = (parent_number + 1) // config['EXPDIFF_PERIOD']
  period_count -= config['METROPOLIS_DELAY_PERIODS'] #chop off 30, meaning go back 3M blocks in time
  period_count -= config['EXPDIFF_FREE_PERIODS'] #chop off 2 more for good measure
  bomb = 2**(period_count)
  return bomb

def calc_metropolis_difficulty(parent, child_timestamp):
  offset = calc_metropolis_offset(parent.difficulty)
  sign = calc_metropolis_sign(parent_timestamp, child_timestamp)
  bomb = calc_metropolis_bomb(parent.number)
  target = (parent.difficulty + offset * sign) + bomb
  return offset, sign, bomb, target
```

## 深入研究Bombs
如果你在线查看[一个难度图表](https://www.coinwarz.com/difficulty-charts/ethereum-difficulty-chart)，你会看到最近每100000个区块就有一个巨大的增长，然后在一个月前就有一个巨大的下降。不想点击链接的用户看看截屏:

![image](https://bigishdata.files.wordpress.com/2017/11/ethereum_time_screenshot.png)

每条水平线表示挖掘一个块每3秒的时间变化。

拥有这样一个bomb值有什么意义?以太坊的一个大目标是摆脱工作证明，它需要精力和时间来创建和验证一个新的块，进入[在Ethereum Wiki中描述的](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ中]的股份证明。为了迫使节点移动到股份实现的证明，每100000个块中就会有一颗“炸弹”，它对难度的影响会翻倍，很快就会使挖掘新块花费很长时间，在旧分支上运行的节点将无法再运行。这就是“冰河世纪”一词的由来;区块链会及时冻结。

这是管理未来变化的一种好方法，但我们也遇到了一个问题，即称为Casper的新股份证明的实现，在图表中显示的巨大困难出现之前没有及时准备好。这就是Metropolis分叉发挥作用的地方——它消除了`bomb`对难度的影响，但几年后它将再次发挥作用，在这里，切换到Casper(希望如此)将做好准备。话虽如此，要预测诸如此类的功能何时准备好投入生产和采用是很困难的，所以如果Casper还没有准备好足够快地推出，您可以创建另一个分支，再次将bomb移动回过去。

### Bomb数学

除了这些图表中所示的bombs对难度的影响，我们还应该说明为什么难度会上升到这些水平，以及这意味着挖掘一个区块需要多长时间。

回顾过去，请记住`offset`是一个指示节点挖掘一个块需要多少时间的指标。

例如，如果之前块之间的时间差是[18-26]，我们说如果我们将难度减少一个`offset`，我们将能够将挖掘块的时间移动回[9-17]范围，或者将`DIFF_ADJUSTMENT_CUTOFF`秒数降低。

所以如果我们通过`offset`增加难度，万恶们希望
So if we increase the difficulty by `offset`, 我们期望通过`DIFF_ADJUSTMENT_CUTOFF`来增加挖掘一个块的时间。如果我们将难度增加`offset`的一半，则挖掘时间应该增加大约`DIFF_ADJUSTMENT_CUTOFF / 2`。

如果我们想知道一个bomb值对挖掘时间的影响有多大，我们需要bomb和offset的比值。

`(bomb / offset) * DIFF_ADJUSTMENT_CUTOFF`

下面的代码展示了我们如何计算挖掘时间并与实际平均时间进行比较，实际平均时间是从[这个图表](https://bitinfocharts.com/compareon/ethere-confirmationtime.html)中收集的，方法是将光标悬停在这些块被挖掘的时间点附近。

```
def calc_mining_time(block_number, difficulty, actual_average_mining_time, calc_bomb, calc_offset):
  homestead_goal_mining_time = 14.5 #about that.
  bomb = calc_bomb(block_number)
  offset = calc_offset(difficulty)
  bomb_offset_ratio = bomb / float(offset)
  seconds_adjustment = bomb_offset_ratio * config['HOMESTEAD_DIFF_ADJUSTMENT_CUTOFF']
  average_mining_time = 0.4 * 60
  calculated_average_mining_time = homestead_goal_mining_time + seconds_adjustment
  print "Bomb: %s" % bomb
  print "Offset: %s" % offset
  print "Bomb Offset Ratio: %s" % bomb_offset_ratio
  print "Seconds Adjustment: %s" % seconds_adjustment
  print "Actual Avg Mining Time: %s" % actual_average_mining_time
  print "Calculated Mining Time: %s" % calculated_average_mining_time
  print


block_number = 4150000
difficulty = 1711391947807191
actual_average_mining_time = 0.35 * 60
calc_mining_time(block_number, difficulty, actual_average_mining_time, calc_homestead_bomb, calc_homestead_offset)

block_number = 4250000
difficulty = 2297313428231280
actual_average_mining_time = 0.4 * 60
calc_mining_time(block_number, difficulty, actual_average_mining_time, calc_homestead_bomb, calc_homestead_offset)

block_number = 4350000
difficulty = 2885956744389112
actual_average_mining_time = 0.5 * 60
calc_mining_time(block_number, difficulty, actual_average_mining_time, calc_homestead_bomb, calc_homestead_offset)

block_number = 4546050
difficulty = 1436507601685486
actual_average_mining_time = 0.23 * 60
calc_mining_time(block_number, difficulty, actual_average_mining_time, calc_metropolis_bomb, calc_metropolis_offset)
```
When run:

```
Bomb: 549755813888
Offset: 835640599515
Bomb Offset Ratio: 0.657885476372
Seconds Adjustment: 6.57885476372
Actual Avg Mining Time: 21.0
Calculated Mining Time: 21.0788547637

Bomb: 1099511627776
Offset: 1121735072378
Bomb Offset Ratio: 0.980188330427
Seconds Adjustment: 9.80188330427
Actual Avg Mining Time: 24.0
Calculated Mining Time: 24.3018833043

Bomb: 2199023255552
Offset: 1409158566596
Bomb Offset Ratio: 1.56052222062
Seconds Adjustment: 15.6052222062
Actual Avg Mining Time: 30.0
Calculated Mining Time: 30.1052222062

Bomb: 8192
Offset: 701419727385
Bomb Offset Ratio: 1.16791696614e-08
Seconds Adjustment: 1.16791696614e-07
Actual Avg Mining Time: 13.8
Calculated Mining Time: 14.5000001168
```

看看后来的Homestead区块的bomb offset率有多大，Metropolis区块之后的bomb offset率又有多小!由于炸弹的冲击力减弱，时间差异下降了一大截。

到目前为止，bomb对难度影响不大。不过它会回来的。如果以太坊继续以`~14.5`秒的速度开采矿块，他们将在17天内开采出10万个矿块。将这段时间乘以30，也就是所谓的`METROPOLIS_DELAY_PERIODS`，我们将回到这样一种状态:无论哈希功率增加多少，bomb都会在大约一年半的时间内产生影响。

### Bomb平衡

Bomb的最后一部分是快速解释一下为什么`bomb`的轻微增加会使整体难度比`bomb`的价值大得多。

这种方法的工作原理是通过计算期望挖掘时间，并由此计算出当前哈希率在该时间段内对挖掘块所需的难度。当bomb数量发生变化时，难度会继续上升或下降，达到那个时候的难度，然后在那个平衡点上徘徊。如果您在`bomb`更改之后查看这些块，您将看到需要多个块才能达到新的级别。

## 边界情况

这里有一些在上面的代码中没有提到的边缘情况。

-MIN_DIFF=131072, 一个块可以有的最小难度值。考虑到难度比这要大得多，考虑这个问题是没有意义的。但是当以太坊产生后，有一个最小的难度可能是有用的。也可以参考 2 ** 17.

-最小的sign值 = -99. 假设有一种情况，由于某种原因，计算下一个块的sign值是-1000。下一个方块难度的符号将以难以置信的速度下降，这意味着大约需要1000个方块才能回到所需的`~14` 秒的开采时间，因为增加难度最大的sign是1(如果包括叔叔节点，则是2)。低于-99的概率非常非常小，但仍然需要涵盖。

## Final Questions
和以前一样，这是我在写这篇文章时遇到的一些问题，这些问题并没有放在主要的帖子里。  
Like before, here’s a list of questions I had when writing this that didn’t get put in the main post.

**头部里有什么？**  
**What’s in the header?**

我相信当我们只讨论工作难度的证明时，这个问题会经常出现。我们知道如何计算难度是很好的，但是如何验证一个块有一个有效的头部超出了这篇文章范围。
I’m sure this question will come up a lot when only talking about Proof of Work difficulty. It’s great that we know how difficulty is calculated, but how to validate a block has a valid header is beyond this post.

我不会在这里解释它，但可能会在以后的文章中决定如何在实现交易之后计算jbc的头。
I’m not going to explain it here, but probably in a future post when I decide how jbc’s header should be calculated after I implement transactions.

我要指出的是，比特币的头部非常简单，在这里值被集合在一起(确保比特的组合方式有正确的尾端)。以太坊的方法要复杂得多，它使用现金方法而不是默克尔树来处理交易。  
I will note that Bitcoin’s header is incredibly simple where the values are smashed together (making sure that the way the bits are combined have the right endian). Ethereum’s is much more complicated by dealing with transactions using a cash method rather than a Merkel tree.

**你是怎么搞清楚的?**  
**How do you go through and figure this out?**

像这样的帖子有很多，但坦白地说，大多数都是非常高级的描述，以及数据，但没有多少显示计算这些数据的代码。我的目标是做到所有这些。  
There are tons of posts out there like this one, but frankly, most are very high level with either descriptions, numbers, but not many showing code that calculates those numbers. My goal is to do all of those.

这意味着为了完全理解并讨论它们，我将翻看介绍并查看源代码。这是来自Python repo的`calc_difficult`函数，来自c++的calcdifficult函数，以及来自Go的calcdifficult函数。我一直在讨论的另一个更重要的问题是，查看代码并不能做什么，您需要做的是自己实现类似的代码。  
That means that to get to complete understanding to talk about them all, I go through and look at the code bases. Here’s the calc_difficulty function from the Python repo, calcDifficulty from c++, and calcDifficulty from Go. Another bigger point that I keep talking about is that looking at the code doesn’t do much at all, what you need to do is implement similar code yourself.

**所有这些时间估计都包含一个数字前的`~`。为什么需要这样做？**  
**All of those time estimations include a ~ before a number. Why is that necessary?**

这个问题问得很好。最主要的答案是不确定有多少节点试图挖掘块，以及随机性。如果你看一下[难度表](https://www.coinwarz.com/difficulty-charts/ethereum-difficulty-chart)，放大到几天的时间跨度，你会发现它是多么随机。这与特定一天的[平均挖掘时间](https://bitinfocharts.com/comparison/ethereum-confirmationtime.html#3m)相同。它们是波动的，所以我们不能准确地说出我们预计的时间。  
That’s a really good question. And the main answer is uncertainty in how many nodes are trying to mine blocks, as well as randomness. If you look at the [difficulty chart](https://www.coinwarz.com/difficulty-charts/ethereum-difficulty-chart) and zoom in to a timespan of a couple days, you’ll see how random it gets. It’s the same with [the average mining time](https://bitinfocharts.com/comparison/ethereum-confirmationtime.html#3m) for a specific day. They fluctuate, and so we can’t say exactly what time we expect.

**工作证明是我经常听到人们谈论到的，但它不是通用的，对吗？**  
**Proof of Work is all I hear people talk about, but it isn’t universal, right?**

正确的!我在上面提到过，我敢打赌，如果你读了这么多关于以太坊的文章，而且你一直把这篇文章读到底，你一定听说过[股票证明](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ#what-are-the-benefits-of-proof-of-stake-as-opposed-to-proof-of-work)这个词。这是一个主要加密货币区块链尚未实现的新选项。还有其他类型的块验证。区块链即将推出的大型企业版本可能根本不会完全基于工作证明。  
Correct! I mentioned it above, and I’m betting that if you’re reading this much about Ethereum and you’re all the way at the bottom of this post that you’ve heard of the term [Proof of Stake](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ#what-are-the-benefits-of-proof-of-stake-as-opposed-to-proof-of-work). This is the new option that a major cryptocurrency blockchain hasn’t implemented yet. There are other types of block validation out there. The big upcoming Enterprise versions of blockchains probably won’t be completely based in Proof of Work at all.

验证区块链将使用哪种类型的最大指示器取决于用例。加密货币需要一些完全不可欺骗的东西。区块链也一样，它可能存储关于谁拥有哪块土地的信息。我们不希望人们能够改变所有权。但是一个用来存储不那么值钱的东西的区块链不需要浪费所有的能量。我知道，在今后几年内，一些其他证明其有效性方法将发挥作用。  
The biggest indicator of what type of validation blockchains will use depends on the use cases. Cryptocurrencies need something completely un-fraudable. Same with a blockchain that might store information on who owns what piece of land. We don’t want people to be able to change ownership. But a blockchain that’s used to store something less insanely valuable doesn’t need to waste all that energy. I know some of these other Proofs of validity will come into play in the next few years.

**你写了一些令人困惑的东西/解释得不够好。我该怎么办？**  
**You wrote something confusing / didn’t explain it well enough. What should I do?**

联系上我。说真的，有很多帖子都在谈论结果，但却没有说明他们是如何计算结果的，并假定每个人都和他们一样聪明，知道发生了什么。我试着做相反的事情，在没有充分解释的情况下，我不会说什么。所以，如果有什么困惑或错误，请联系我，我保证搞定它。  
Get in freaking contact. Seriously, tons of posts out there that talk about results but don’t say how they calculated it and assume that everyone is as smart as them and know what’s going on. I’m trying to be the opposite where I don’t say something without fully explaining. So if there’s something confusing or wrong, contact me and I’ll make sure to fix the issue.

**我可以在Twitter上和你聊天吗**  
**Can I DM you on Twitter?**

[@jack_schultz](https://twitter.com/jack_schultz)

**你更喜欢奸臣还是忠臣？**  
**Do you like cats or dogs better?**

跟你没关系  
Cats.
