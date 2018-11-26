---
layout: post
title: "如何构建自己的区块链 第一部分 - 创建，存储，同步，显示，挖矿和工作量证明"
date: 2018-11-12 23:56:00 +0800
categories: 开发与技术
tag: Blockchain
---
独立翻译自:[How to Build Your Own Blockchain Part 1 — Creating, Storing, Syncing, Displaying, Mining, and Proving Work](https://bigishdata.com/2017/10/17/write-your-own-blockchain-part-1-creating-storing-syncing-displaying-mining-and-proving-work/)
<!-- more -->
* content
{:toc}

实际上我能够通过登录到我的Coinbase账户查询到账户有多久了，查看比特比钱包的历史记录，以及看看2012年注册后Coinbase上的交易。比特比是以约6.5美元每个交易的。如果我仍然持有那0.1BTC，那么此刻它价值超过500美元。有人会想知道，当比特币价值2000美元时我卖掉了它。所以我仅获得了200美元而不是现在的550美元。应继续持有的。

![image](https://bigishdata.files.wordpress.com/2017/10/screen-shot-2017-10-14-at-9-47-47-pm.png?w=696&h=754)

感谢脑子 胳膊壮(Someone named Brain Armstrong, haha). 

尽管知道比特币的存在，但我从未介入其中。我看到了$/BTC汇率的涨涨跌跌。我看到人们谈论它未来值多少，并且看了一些关于BTC是多么无意义的文章。我从来没有对此发表意见，只是持续跟进。

同样，我很少跟进区块链。最近，我父亲多次强调早上CNBC和Bloomberg电台经常提到区块链，以及他完全不懂它是什么。

突然间，我意识到我应该尝试学习区块链而不是我所仅知道的前沿新闻。我开始做了很多“研究”，这意味着我会在互联网上搜索，试图找到解释区块链的其他文章。有些很好，有些不好，有些讲的太细了，有些则是太宏观了。

阅读只能浅尝辄止，我就知道一件事，那就是**阅读学习并不能让你获得到从编程中学到的知识**。所以我意识到我应该通过尝试编写自己的基本本地区块链。

这里重点要说的是我在这里描述的基本区块链和“专业”区块链存在差异。本区块链不会创建加密货币。**区块链不需要生产可以交易和兑换实物货币的硬币**。区块链用于存储和验证信息。 货币用于鼓励节点参与验证，但不必存在。

我写这篇文章的原因是1）阅读本文的人因此可以更多地了解区块链本身，2）我可以因此尝试通过讲解代码来学习更多，而不仅仅是编写代码。

在本文中，我将展示我想要存储区块链数据并生成创世块的方式，节点如何与本地区块链数据同步，如何显示区块链（将来将用于与其他节点同步），然后如何通过和挖掘并创建有效的新块。 对于第一篇文章，没有其他节点。没有钱包，没有其他节点，没有重要数据。相关信息后面再说。

## 本文其他系列文
- [Part 2 — Syncing Chains From Different Nodes](https://bigishdata.com/2017/10/27/build-your-own-blockchain-part-2-syncing-chains-from-different-nodes/)
- [Part 3 — Nodes that Mine](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
- [Part 4.1 — Bitcoin Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
- [Part 4.2 — Ethereum Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/21/how-to-build-your-own-blockchain-part-4-2-ethereum-proof-of-work-difficulty-explained/)

## 长话短说
如果你不想涉入细节和阅读代码，或者如果你在搜索区块链理解的文章时遇到过这篇文章，我将尝试撰写有关区块链如何工作的摘要。

在上层角度来看，区块链是一个参与区块链的每个人都能够存储，查看，确认和永不删除数据的数据库。

从底层角度来看，块中的数据可以是任何数据只要这个区块链允许。例如，比特别区块链中的数据只是比特别账户间的交易信息。以太坊区块链允许Ether的类似事务，但也允许用于运行代码的事务。

继续往后说，在创建块并链接到区块链之前，它由大多数处理区块链的人（称为节点）进行验证。 真正的区块链是包含被大量节点正确验证的最大块数的链。这意味着如果一个节点尝试更改前一个块中的数据，则较新的块将无效，并且节点将不信任来自不正确块的数据。

别担心这困惑的一切。我花了一段时间自己才明白并且花更长的时间以我姐姐（没有任何区块链背景）能理解的方式来写这篇文章。

如果你想看代码，检出[the part 1 branch on Github](https://github.com/jackschultz/jbc/tree/part-1).。任何人有问题，留言，纠错或打赏（如果你乐意就太好了）保持联络, 或在 [twitter](https://twitter.com/jack_schultz)告诉我。

## Step 1 — 类和文件

对我来说，第1步是编写一个在节点运行时处理块的类。我将这个类称为“Block”。坦率地说，这个类没什么要做的。在`__init__`函数中，我们将默认所有必需的信息都在字典中提供了。如果我正在编写一个生产区块链，这就不明智了，但对于只有我是编写所有代码的范例来说还算可以了。我还想编写一个方法，将重要的块信息输出到dict中，然后如果我要是将一个块信息打印到终端，就会有好的方式来显示块信息。

```
class Block(object):
  def __init__(self, dictionary):
  '''
    We're looking for index, timestamp, data, prev_hash, nonce
  '''
  for k, v in dictionary.items():
    setattr(self, k, v)
  if not hasattr(self, 'hash'): #in creating the first block, needs to be removed in future
    self.hash = self.create_self_hash()

  def __dict__(self):
    info = {}
    info['index'] = str(self.index)
    info['timestamp'] = str(self.timestamp)
    info['prev_hash'] = str(self.prev_hash)
    info['hash'] = str(self.hash)
    info['data'] = str(self.data)
    return info

  def __str__(self):
    return "Block<prev_hash: %s,hash: %s>" % (self.prev_hash, self.hash)
```

当我们想要创建第一个块时，我们可以运行简单的代码。

```
def create_first_block():
  # index zero and arbitrary previous hash
  block_data = {}
  block_data['index'] = 0
  block_data['timestamp'] = date.datetime.now()
  block_data['data'] = 'First block data'
  block_data['prev_hash'] = None
  block = Block(block_data)
  return block
```
漂亮。本节的最后一个问题是将数据存储在文件系统中的位置。我们需要这样做，因为当我们关闭节点，我们不会丢失本地块数据。

为了尝试模仿以太坊Mist文件夹机制，我将使用'chaindata'命名该数据文件夹。每个块都将被允许根据其索引命名自己的文件。我们需要确保文件名以大量前导零开头，使得块按数字顺序排列。

以上这些代码，就是我创建第一个块所需要的

```
#check if chaindata folder exists.
chaindata_dir = 'chaindata'
if not os.path.exists(chaindata_dir):
  #make chaindata dir
  os.mkdir(chaindata_dir)
  #check if dir is empty from just creation, or empty before
if os.listdir(chaindata_dir) == []:
  #create first block
  first_block = create_first_block()
  first_block.self_save()
```
## Step 2 — 本地同步区块链
当你启动节点时，在能够开始挖矿，解释数据或发送/创建链的新数据之前，你需要同步节点。 由于没有其他节点，我只谈从本地文件中读取块。后面从文件读取将是同步的一部分，但也会与其他节点交流以收集在你未运行自己的节点时生成的块。

```
def sync():
  node_blocks = []
  #We're assuming that the folder and at least initial block exists
  chaindata_dir = 'chaindata'
  if os.path.exists(chaindata_dir):
    for filename in os.listdir(chaindata_dir):
      if filename.endswith('.json'): #.DS_Store sometimes screws things up
        filepath = '%s/%s' % (chaindata_dir, filename)
        with open(filepath, 'r') as block_file:
          block_info = json.load(block_file)
          block_object = Block(block_info) #since we can init a Block object with just a dict
          node_blocks.append(block_object)
return node_blocks
```
现在还比较简单。从文件夹中读取字符串并将其加载到数据结构中不需要太复杂的代码。 现在这些可以正常工作。但是在以后的文章中，当我编写不同节点进行通信的能力时，这个同步功能将变得更加复杂。

## Step 3 — 展示区块连
现在我们在内存中有Blockchain了，我想开始能够在浏览器中显示链。现在这样做的两个原因。 首先是在浏览器中验证事情已经发生了变化。然后我还想在将来使用浏览器来查看和操作区块链。比如发起交易或管理钱包。

我在这里使用Flask，因为它非常容易开始，而且因为我掌控着它。

这是显示区块链json的代码。 我会忽略这里节省空间的导入要求。

```
node = Flask(__name__)

node_blocks = sync.sync() #inital blocks that are synced

@node.route('/blockchain.json', methods=['GET'])
def blockchain():
  '''
  Shoots back the blockchain, which in our case, is a json list of hashes
  with the block information which is:
  index
  timestamp
  data
  hash
  prev_hash
  '''
  node_blocks = sync.sync() #regrab the nodes if they've changed
  # Convert our blocks into dictionaries
  # so we can send them as json objects later
  python_blocks = []
  for block in node_blocks:
    python_blocks.append(block.__dict__())
  json_blocks = json.dumps(python_blocks)
  return json_blocks

if __name__ == '__main__':
  node.run()
```
运行这段代码，访问`localhost:3000/blockchain.json`, 你将看到当前块信息输出。

## Part 4 — “挖矿”, 也被认为是创建区块


我们只有一个创世块，如果我们想要存储和分发更多数据，我们需要一种方法将其包含在一个新块中。问题是如何在链接到前一个区块时创建一个新块。

在比特币白皮书中，Satoshi将其描述如下。请注意，'timestamp server'被称为'node'：

---
> 本解决方案首先提出一个“时间戳服务器”。时间戳服务器通过对以区块(block)形式存在的一组数据实施随机散列而加上时间戳，并将该随机散列进行广播，就像在新闻或世界性新闻组网络（Usenet）的发帖一样。显然，该时间戳能够证实特定数据必然于某特定时间是的确存在的，因为只有在该时刻存在了才能获取相应的随机散列值。每个时间戳应当将前一个时间戳纳入其随机散列值中，每一个随后的时间戳都对之前的一个时间戳进行增强(reinforcing)，这样就形成了一个链条（Chain）。

> The solution we propose begins with a timestamp server. A timestamp server works by taking a hash of a block of items to be timestamped and widely publishing the hash... The timestamp proves that the data must have existed at the time, obviously, in order to get into the hash. Each timestamp includes the previous timestamp in its hash, forming a chain, with each additional timestamp reinforcing the ones before it.

这时描述下面的截图。

![image](https://bigishdata.files.wordpress.com/2017/10/screen-shot-2017-10-16-at-11-41-52-am.png)


该部分的摘要是，为了将块链接在一起，我们创建新块信息的哈希值，其包括块创建的时间，前一块的哈希值以及块中的信息。我将这组信息称为块的“头部”。通过这种方式，我们可以通过遍历块之前的所有哈希并验证序列来验证块的真实性。

对于本例中的头部，我正在创建的头部是将字符串值一起添加到一个巨大的字符串中。 我包括的数据是：

1. 索引, 指这将是第几个块
2. 前块的哈希值
3. 数据,在这种情况下只是随机字符串。对于比特币，这被称为Merkle根，它是关于交易的信息
4. 我们挖掘块时的时间戳

```
def generate_header(index, prev_hash, data, timestamp):
  return str(index) + prev_hash + data + str(timestamp)
```

在感到困惑之前，不需要一起添加信息字符串来创建头部。 **要求是每个人都知道如何生成块的头，并且在头部内是前一个块的哈希。 这样每个人都可以确认新块的正确哈希值，并验证两个块之间的链接。**

[比特币头部](https://en.bitcoin.it/wiki/Block_hashing_algorithm) 比组合字符串复杂多了。它使用数据的哈希值，时间以及关联字节如何存储在计算机内存中。 但是现在，添加字符串就足够了。

一旦我们有了头部，我们想要通过并计算经过验证的哈希值，并计算哈希值。 在我的哈希计算中，我将做一些与比特币方法略有不同的事情，但我仍然通过sha256函数计算块头。

```
def calculate_hash(index, prev_hash, data, timestamp, nonce):
  header_string = generate_header(index, prev_hash, data, timestamp, nonce)
  sha = hashlib.sha256()
  sha.update(header_string)
  return sha.hexdigest()
```

最后，为了挖掘块，我们使用上面的函数来获取新块的哈希值，将哈希值存储在新块中，然后将该块保存到chaindata目录中。

```
node_blocks = sync.sync()

def mine(last_block):
  index = int(last_block.index) + 1
  timestamp = date.datetime.now()
  data = "I block #%s" % (int(last_block.index) + 1) #random string for now, not transactions
  prev_hash = last_block.hash
  block_hash = calculate_hash(index, prev_hash, data, timestamp)

  block_data = {}
  block_data['index'] = int(last_block.index) + 1
  block_data['timestamp'] = date.datetime.now()
  block_data['data'] = "I block #%s" % last_block.index
  block_data['prev_hash'] = last_block.hash
  block_data['hash'] = block_hash
  return Block(block_data)

def save_block(block):
  chaindata_dir = 'chaindata'
  filename = '%s/%s.json' % (chaindata_dir, block.index)
  with open(filename, 'w') as block_file:
    print new_block.__dict__()
    json.dump(block.__dict__(), block_file)

if __name__ == '__main__':
  last_block = node_blocks[-1]
  new_block = mine(last_block)
  save_block(new_block)
```
嘿哈！虽然使用这种类型的块创建，但拥有最快CPU的人能够创建一个其他节点承认真实的最长链。我们需要一些方法来减缓块创建并在移向下一个块之前相互确认。

## Part 5 — 工作量证明

为了减速，我们研究一下比特币的工作量证明做法。[股权证明](https://en.wikipedia.org/wiki/Proof-of-stake)是你看到区块链用来达成共识的另一种方式，但这个我还要研究一下。

做这个的方法是调整块的哈希值具有某些属性的要求。就像比特币一样，我要确保哈希值以一定数量的零开始，然后才能继续下一个。 这样做的方法是在头部中添加一条信息 - nonce。

```
def generate_header(index, prev_hash, data, timestamp, nonce):
  return str(index) + prev_hash + data + str(timestamp) + str(nonce)
```
现在调整挖掘函数以创建哈希，但是如果块的哈希没有带足够的零，我们递增nonce值，创建新头部，计算新哈希值并检查是否带有足够的零。

```
NUM_ZEROS = 4

def mine(last_block):
  index = int(last_block.index) + 1
  timestamp = date.datetime.now()
  data = "I block #%s" % (int(last_block.index) + 1) #random string for now, not transactions
  prev_hash = last_block.hash
  nonce = 0

  block_hash = calculate_hash(index, prev_hash, data, timestamp, nonce)
  while str(block_hash[0:NUM_ZEROS]) != '0' * NUM_ZEROS:
    nonce += 1
    block_hash = calculate_hash(index, prev_hash, data, timestamp, nonce)
  block_data = {}
  block_data['index'] = int(last_block.index) + 1
  block_data['timestamp'] = date.datetime.now()
  block_data['data'] = "I block #%s" % last_block.index
  block_data['prev_hash'] = last_block.hash
  block_data['hash'] = block_hash
  block_data['nonce'] = nonce
  return Block(block_data)
```
精彩。这个新块包含有效的nonce值，因此其他节点可以验证哈希值。 我们可以生成，保存和分发这个新块到其他部分。

## 总结
就是这样！目前。关于区块链的大量问题和功能我还没有引入。 

例如，其他节点如何参与？节点如何传输他们想要包含在块中的数据？我们如何将信息存储在块中而不仅仅是一个巨大的字符串？是否有更好的头部类型而不用包含那个巨大的数据字符串？

这个系列的更多部分将会出现，我将继续解决这些问题。所以，如果你有关于你想看到哪些部分的建议，直接告诉我[twitter](https://twitter.com/jack_schultz), 在下面留言, 或者 [get in contact](https://bigishdata.com/contact/)!

感谢我的姐姐Sara阅读这篇文章进行编辑，并询问有关区块链的问题，所以我不得不重写以澄清。
