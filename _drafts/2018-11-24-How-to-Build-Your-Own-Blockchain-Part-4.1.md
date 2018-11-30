---
layout: post
title: "如何构建自己的区块链 第四一部分 - 比特币工作证明难度解释"
date: 2018-11-24 22:27:00 +0800
categories: 开发技术
tag: Blockchain
---
独立翻译自:[How to Build Your Own Blockchain Part 4.1 — Bitcoin Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/13/how-to-build-a-blockchain-part-4-1-bitcoin-proof-of-work-difficulty-explained/)
<!-- more -->
* content
{:toc}

如果您想知道为什么这是第4.1部分而不是第4部分，以及为什么我不讨论继续构建本地jbc，那是因为在较低的层次上解释比特币的工作困难证明需要大量篇幅。所以不像这个标题说的，这篇文章在第4部分不是如何建立一个区块链。它是关于现有的区块链是如何构建。

我在这个第4篇文章的主要目标是有一个关于比特币的部分，下一个是关于Ethereum的部分，最后讨论jbc将如何运行和验证工作量证明。在写完第1部分的全部内容来解释比特币的PoW难度之后，它不适合放在一个单独的部分中。人们，包括我在内，在阅读一篇冗长的文章时，往往会感到无聊而无法完成。

因此，第4.1部分将介绍比特币的PoW难度计算。第4.2部分将通过以太坊的工作量证明来计算。然后第4.3部分将是我决定我想要的jbc PoW如何做以及做时间计算来看看挖掘将花费多长时间。

本文分为以下几个部分：

1. Calculate Target from Bits//从位计算目标
2. Determining if a Hash is less than the Target//确定哈希是否小于目标
3. Calculating Difficulty//计算难度
4. How and when block difficulty is updated//如何以及何时更新块难度
5. Full code//完整代码
6. Final Questions//最后的问题

# 长话短说
难度的总称是指一个节点需要做多少工作才能找到比target小的hash值。在一个块中有一个值是关于难度的——它就是`bits`。为了计算散列转换为十六进制值时必须小于`target`的值，我们使用`bits`字段并通过一个返回target的方程运行它。然后，我们使用`target`来计算`difficulty`，其中`difficulty`只是一个数字，人类可以通过这个数字来理解该块的工作证明有多困难。

如果您继续阅读，*我将介绍区块链如何确定挖掘块的散列需要小于什么样的target数才能有效，以及如何计算该taget值*

# 从Bits计算Target值
为了完成比特币的PoW，我需要使用实际块上的值并解释其计算，这样读者就可以自己验证所有这些代码。首先，我要取一个随机的块号然后用它进行计算。

```
>>>import random
>>> random.randint(0, 493928)
111388
```

[块号11138](https://blockchain.info/block/00000000000019c6573a179d385f6b15f4b0645764c4960ee02b33a3c7117d1e) 就是它! 时间追溯到2011年3月.
我们将使用块中的`bits`和`difficulty`值以及`hash`来分配变量，这样我们可以在最后测试它的有效性。

```
>>> bits = '453062093'
>>> difficulty = float('55,589.52'.replace(',',''))
>>> block_hash = '00000000000019c6573a179d385f6b15f4b0645764c4960ee02b33a3c7117d1e'
```

下一部分将展示如何从bits字段到target。至少有两种不同的方法可以做到这一点—*第一种涉及字符串操作和字符串到整数的转换，另一种使用位操作*

对于第一个字符串操作部分，我们将位字符串转换为整数，然后转换为十六进制字符串。hex_bits字符串的前两个字符在一个变量中，我称之为`shift`，其余六个称为`value`。然后，我们在整数方程中使用这些值来计算`target`的整数值，我们可以将其转换为十六进制字符串，看看它有多漂亮!

请注意，`target`和`hex(target)`后面的`L`只是Python告诉您，numbexr太长，不能作为int存储，它是存储为[long](https://docs.python.org/2/library/functions.html#long)。

```
>>> hex_bits = hex(int(bits))
>>> hex_bits
'0x1b012dcd'
>>> shift = '0x%s' % hex_bits[2:4]
>>> shift
'0x1b'
>>> shift_int = int(exponent, 16)
27
>>> value = '0x%s' % hex_bits[4:]
>>> value
'0x012dcd'
>>> value_int = int(coefficient, 16)
77261
>>> target = value_int * 2 ** (8 * (shift_int - 3))
>>> target
484975157177710342494716926626447514974484083994735770500857856L
>>> hex_target = hex(target)
>>> hex_target
'0x12dcd000000000000000000000000000000000000000000000000L'
```

当我们像上面那样分割字符串来计算target值时，这看起来有点假。当查看[c++ bitcoin实现](https://github.com/bitcoin/bitcoin/blob/87e69c2549c44b862558f1c025dc0c4449fca272/src/_uint256.cpp#L206)时，它显示了使用`bits`计算`target`的另一种更技术的主要方法。但这里是Python实现。

我将逐行打印这里的值，以便更好地显示发生了什么。在阅读之前需要知道的一件事是十六进制字符串中的一个字符表示4位。因此，当我们用`bits >> 24`将位变量移位24位时，有效地删除了十六进制字符串的最后6个字符，只留下前两个。为了得到值变量，我们使用位和运算符&取得最后23位的值。这23位通常是十六进制字符串的最后6个字符(由0x7fffff中的位数定义)，

现在，`8 * (shift - 3)`的值决定了我们要移位多少位。在这种情况下，它被确定为192位。如果我们把192除以4，我们就会得到target数的十六进制字符串表示的值后面的零字符个数。你会看到它是48个0。另一件让人困惑的事情是`value << y`和`value * 2**y`是一样的。这就是方程之间的关系。


```
>>> bits = '453062093'
>>> hex(int(bits))
'0x1b012dcd'
>>> shift = bits >> 24
>>> shift
27
>>> hex(shift)
'0x1b'
>>> value = bits & 0x007fffff
>>> hex(value)
'0x12dcd'
>>> 8 * (shift - 3)
192
>>> 192 / 4 #each character in a hex string represents 4 bits
48
>>> value <<= 8 * (shift - 3)
>>> hex(value)
'0x12dcd000000000000000000000000000000000000000000000000L'
>>> hex(value).count('0') - 1 #don't count the leading zero
48
```

完整的函数版本只有几行。

```
>>> def get_target_from_bits(bits):
...   shift = bits >> 24
...   value = bits & 0x007fffff
...   value <<= 8 * (shift - 3)
...   return value
...
>>> target = def get_target_from_bits(int(bits))
>>> hex(target)
'0x12dcd000000000000000000000000000000000000000000000000L'
```

太好了。如果您查看上面的字符串操作代码，会发现它比上面的函数更令人困惑，而且还有许多额外的行。通过这两种方法来真正理解它的含义是很好的，但是要坚持使用简单的方法。前进吧。

# 确定hash是否小于target

通常，有两种简单的方法可以查看块的哈希是否对于target有效。第一种方法是使用完整的，64个字符，256位，十六进制字符串并比较它们。这很有用，因为我们可以很容易地看到这一点。为了说明这种情况，我们将用0填充hex_target来填满所有64个字符。

```
>>> hex_target = hex(target)
>>> len(hex_target)
56
>>> len(hex_target[2:-1]) #[2:-1] removes the leading '0x' and ending 'L'
53
>>> num_padded_zeros = 64 - hex_target_len
>>> num_padded_zeros
11
>>> padded_hex_target = "0x%s%sL" % ('0' * (64-num_padded_zeros), hex_target[2:-1])
>>> padded_hex_target
'0x0000000000012dcd000000000000000000000000000000000000000000000000L'

```

最后，我们希望验证块的散列是否小于target。而且，因为字符串可以被认为小于或大于，我们就这么做吧。

```
>>> len(block_hash)
64
>>> padded_block_hash = '0x%sL' % block_hash
>>> padded_block_hash
0x00000000000019c6573a179d385f6b15f4b0645764c4960ee02b33a3c7117d1eL
>>> padded_hex_target
0x0000000000012dcd000000000000000000000000000000000000000000000000L
>>> assert padded_block_hash < padded_hex_target
>>>
```

在这种情况下，您将看到十六进制target大于块的hash。比较哪个字符串更大就像<或>一样简单。如果您想知道Python如何确定`'f' > '1'`之类的字符，请参见下面的问题。

另一种确认方法是对两个值都使用整数。但是你会看到，很难，如果不是不可能的话，观察哪个值更大，一个值比另一个大多少。

```
>>> block_hash_int = int(block_hash, 16)
>>> block_hash_int
41418456048005639864974238890271849696605172030151526454492446L
>>> target
484975157177710342494716926626447514974484083994735770500857856L
>>> assert(block_hash_int < target)
>>> target - block_hash_int
443556701129704702629742687736175665277878911964584244046365410L
```
这些整数是如此之大，以至于单独看一个整数几乎不可能知道对比它是容易的还是困难的。但是由于block_hash_int比target整数少一个字符，您可以看到这种情况。当你减去这两个大数时，你得到另一个大数。

这两个方法都是有效的，但是能够查看十六进制字符串比查看整数要好。但实际上，当代码运行时，选择其中一个并没有真正的好处。

# 计算难度
The most important part of `difficulty` is to remember from above that `difficulty` is simply a human representation of the `target`. I talked about how integer values of `targets` are pointless since it’s hard to humanly compare two values that are giant integer. Likewise, it’s tough to compare two targets to each other using the hex strings. The `difficulty` values in a block are to make that easier.

Below, `difficulty_one_target` is defined as the easiest allowed target value. To calculate the difficulty of another block, we simply divide the `difficulty_one_target` by the `target` we’ve calculated using `bits`. You’ll see that `difficulty_one_target` will be larger than any other `target` number since `targets` go lower with more difficulty. When you divide the larger numerator by a smaller denominator, you’ll have a greater than one value.

```
>>> difficulty_one_target = 0x00ffff * 2**(8*(0x1d - 3))
>>> difficulty_one_target
26959535291011309493156476344723991336010898738574164086137773096960L #calculated using 
>>> pad_leading_zeros(hex(difficulty_one_target)) #pad_leading_zeros is function defined below
'0x00000000ffff0000000000000000000000000000000000000000000000000000'
>>> calculated_difficulty = difficulty_one_target / float(target) #float() to make it decimal devision
55589.518126868665 #this is the same as the difficulty on the block.
>>> allowed_error = 0.01
>>> assert abs(float(block_difficulty) - calculated_difficulty) <= allowed_error
>>>
```

Feel free to go back and read this over slowly. It took me a long time to figure out the relationship between `bits`, `difficulty`, and `target`. Whether we need ints, hex values, or strings to verify. There are a bunch of ways to do this, and hopefully I’ve listed them all here.

# How and when block difficulty is updated
The final big part of Bitcoin’s proof of work is to show how and when the difficulty is changed. Pretty much every post on this topic will say that the chain will recalculate the required new difficulty every 2016 blocks. But they don’t dive too deep at all into the topic. That’s what I’m going to do here.

The first part I have to start with is how to start with a `target`, and go back to `bits`. The `get_bits_from_target` function takes a target, calculates the the highest ranking bit that’s set and uses that to calculate the size. Knowing the size, we shift the target all the way down, removing the zeros we don’t need anymore, and finally put the value of size at the front of the bits so we keep it stored.

```
>>> bits = '403088579'
>>> hex(int(bits))
'0x1806a4c3'
>>> target = get_target_from_bits(int(bits))
>>> hex(target)
'0x6a4c3000000000000000000000000000000000000000000L'
>>> def get_bits_from_target(target):
...   bitlength = target.bit_length() + 1 #look on bitcoin cpp for info
...   size = (bitlength + 7) / 8
...   value = target >> 8 * (size - 3)
...   value |= size << 24 #shift size 24 bits to the left, and taks those on the front of compact
...   return value
...
>>> bits = get_bits_from_target(target)
>>> bits
403088579L
>>> hex(bits)
'0x1806a4c3L
```

They match, so we know the code correctly returns a `target` to `bits`.

Time to show how to calculate the next block’s bits for an actual set of blocks on the blockchain. For the example here, I’m going to say we’ve just received block #[405215](https://blockchain.info/block/0000000000000000034656c96781091b5fbc799c881ea85b41cba0b88128eff7) and are looking to mine block #[405216](https://blockchain.info/block/000000000000000006969473ee3a2126d9a953ad00eee6443b4d2d2e0881fc07). Looking back 2016 blocks, [403200](https://blockchain.info/block/000000000000000000c4272a5c68b4f55e5af734e88ceab09abf73e9ac3b6d01) will be considered the starting block.

We have a starting `target` calculated from that block’s `bits`. We use that difficulty for the next 2016 blocks. When we come to the 2017th block after the first, we calculate the time in seconds it took for us to go from the original block to the 2016th. We divide that by the 2016 block in seconds, multiply the current target by that value, and then calculate the new bits from that target.

```
TARGET_TIMESPAN = 1209600

def change_target(prev_bits, starting_time_secs, prev_time_secs):
  old_target = get_target_from_bits(int(prev_bits))
  time_span = prev_time_secs - starting_time_secs
  time_span_seconds = int(time_span.total_seconds())
  new_target = old_target
  new_target *= time_span_seconds
  new_target /= TARGET_TIMESPAN
  return new_target

#starting_block is the initial block on this 2016 block runtime #403200
starting_block_timestamp = '2016-03-18 09:07:48' #timestamp on block
starting_block_time_seconds = datetime.datetime.strptime(starting_block_timestamp, '%Y-%m-%d %H:%M:%S')

#prev_block is the block right before the difficulty conversion #405215
prev_block_bits = '403088579' #bits on block
prev_block_timestamp = '2016-04-01 06:24:09' #timestamp on block
prev_block_time_seconds = datetime.datetime.strptime(prev_block_timestamp, '%Y-%m-%d %H:%M:%S')

calculated_new_target = change_target(prev_block_bits, starting_block_time_seconds, prev_block_time_seconds)
calculated_new_bits = get_bits_from_target(calculated_new_target)

#new_block is the first block of the next block #405216
new_bits = '403085044'
new_target = get_target_from_bits(new_bits)

print hex(calculated_new_target)
print hex(new_target)

print hex(calculated_new_bits)
print hex(int(new_bits))
```
Which spits out
```
Calculated new target: 0x696f4a7b94b94b94b94b94b94b94b94b94b94b94b94b94bL
New target from block: 0x696f4000000000000000000000000000000000000000000L
Calculated new bits: 0x180696f4L
New bits from block: 0x180696f4
```
The calculated new target is very specific. But we don’t really care about making sure a block’s hash is that specific. They only need to know the leading 23 bits. Ditch the rest of them, shove them down, and throw the shift on the front.

# Full Code
I was going to copy and paste the giant amount of code I wrote above so you could copy and paste it and run it yourself. But that would take up too much space. So [here’s a gist](https://gist.github.com/jackschultz/19cbb54b8637f854c3d99571b34bf4a7) where you’ll be able to see everything.

# Final Questions

ongrats if you’ve made it this far! Here’s a set of questions that I had when learning and writing this. If you have any more, get in contact and I’ll throw and answer on.

*Why are the hex values only strings?*

Because hex values are only representations of integers. If you’re looking to see the hex value of an integer, we need a string to look at the different base 16 values. Look at the [Python definition of hex()](https://docs.python.org/2/library/functions.html#hex) slightly more info.

*How can a string comparison know that 'f' > '9'?*

In memory, characters are stored as unicode values. So when you compare two characters, the one with the larger unicode value is shown as being the biggest. Note that ASCII also has the same values as unicode for 0-127. When comparing strings, they go through each character one by one and whichever has the largest value first wins the race.

```
>>> ord('f')
102
>>> ord('a')
97
>>> ord('b')
98
>>> ord('9')
57
>>> ord('1')
49
>>> ord('2')
50
>>> ord('a')
97
>>> ord('b')
98
>>> ord('f')
102
>>> 'f' > 'a'
True
>>> 'a' > '1'
True
```

*How has Bitcoin difficulty changed over time?*

There are a bunch of sites that have interactive graphs showing the change over time; quick google will find one. But, here’s the pic from [Bitcoin’s actual Wikipedia page](https://en.wikipedia.org/wiki/Bitcoin_network) that shows how that difficulty number has changed over time.

![image](https://bigishdata.files.wordpress.com/2017/11/history-bitcoin-difficulty.png)
Decently linear, with a flat spot in the middle. Someone should do a history on that.

*What’s the deal with mid 2011 to beginning of 2013 difficulty being flat?*

Look at this screenshot from [Bitcoin.com](https://charts.bitcoin.com/chart/price) for the time period of 2011 and 2013.

![image](https://bigishdata.files.wordpress.com/2017/11/bitcoin-price-chart-2011-2013.png)

Hmmm

A giant spike in price all the way up to an astonishing $29 a coin that crashed until the beginning of 2013 where it starts flying up again. I’ll go ahead and say there’s a nice correlation there where miners saw the crash and said screw it, stopped their mining, and left fewer nodes trying to mine a new block. Fewer nodes means the longer it takes to find a valid hash so we had to keep difficulty down.

*Does a larger bit value imply a larger difficulty?*

Nope. Remember, bit values are stored so we can split it into two numbers used to calculate the target. When looked at as an integer, it’s definitely not comparable.

If you want examples, look at the couple blocks listed in the code above. Comparing blocks 0 and 32256, you’ll see opposite difference in bits and difficulty.

How is the header hash calculated?

By smashing values together. [Take a look here](https://en.bitcoin.it/wiki/Block_hashing_algorithm) and you’ll see more about it.

I see people mentioning we only calculate new bits from 2015 blocks and not 2016. What’s that mean?

You’ll see that a lot when looking at other posts. What this means is if we included all 2016 blocks in a set, we’d start with the timestamp before the start of a new 2016 set of blocks since that’s says when nodes started trying to find the next block.

This doesn’t work out when you think of calculating the new bits for actual block 2017. If we took into account how long it took to mine the first 2016, we wouldn’t know that since we didn’t know when we started running the first miners.

Why didn’t you talk about how much time it takes to find a valid hash?

That’s a different topic. There’s [some material](https://en.bitcoin.it/wiki/Difficulty#What_network_hash_rate_results_in_a_given_difficulty.3F) on calculating that if there’s a single node doing the mining, but that’s tough to figure out unless you have a bigger amount of nodes. When I write the part for jbc’s difficulty, I’ll have tests to show time.

What’s your Twitter handle so I can follow you??

[@jack_schultz](https://twitter.com/jack_schultz)

What if I only want to ask you a question?

[Contact](https://bigishdata.com/contact/) is a great way to do this. Just make sure you type your email correctly; I’ve gotten some emails before that I couldn’t reply to since that email address didn’t exist. Also feel free to comment on this post as well.

What’s next?

Part 4.2 about the Ethereum proof of work difficulty! It’s somewhat similar, but the target is calculated in a different way. Also, the time when difficulties change is different as well. [Check out the chart](https://www.coinwarz.com/difficulty-charts/ethereum-difficulty-chart) that show difficulty over time to see what I mean.

After that, part 4.3 will be about how I want to calculate difficulty for jbc.