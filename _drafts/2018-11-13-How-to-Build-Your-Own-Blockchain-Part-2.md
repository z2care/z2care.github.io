---
layout: post
title: "如何构建自己的区块链 第二部分 - 从不同节点同步区块链"
date: 2018-11-13 23:56:00 +0800
categories: 开发技术
tag: Blockchain
---
* content
{:toc}

起初我的目标是写关于节点间同步和彼此通信，还有挖矿和广播它们成功区块给其他节点。最终我意识到这些大量的代码和阐释在一篇文章中完成太多了。基于此因，我决定第二部分只谈节点作为后续话题的开始。

<!-- more -->

欢迎来进入JackBlockChain第二部分，这里我将介绍不同节点间通信的能力。

起初我的目标是写关于节点间同步和彼此通信，还有挖矿和广播它们成功区块给其他节点。最终我意识到这些大量的代码和阐释在一篇文章中完成太多了。基于此因，我决定第二部分只谈节点作为后续话题的开始。

通过阅读本文有将看到我做了什么和怎么做的。但是你不会看到所有代码。相关代码太多，如果你要看最终实现，请看[GitHub上的part-2分支](https://github.com/jackschultz/jbc/tree/part-2)。

同所有编程一样，我不会按顺序写代码。我有不同的想法，尝试不同的策略，对代码修修改改并最终完成它们。

看起来不错！我想提一下我的流程让读这篇文章的人并不要总认为编写编程的人会按照代码的顺序进行编写。如果进展顺利，我乐意写我修复过的问题，以及写困住我的部分。

很难解释所有的流程而且我猜大多数读者并不想知道如何编程，而是想看看代码如何实现的。记住，编程很少按顺序进行。

[Twitter](https://twitter.com/jack_schultz), [contact](https://bigishdata.com/contact/), 随时通过下面的留言告诉我错误，或者告诉我哪里帮到你了，很乐意听你的反馈。

![image](https://bigishdata.files.wordpress.com/2017/10/screen-shot-2017-10-27-at-11-04-01-am.png)

## 本系列其他文章
- [Part 1 — Creating, Storing, Syncing, Displaying, Mining, and Proving Work](https://bigishdata.com/2017/10/17/write-your-own-blockchain-part-1-creating-storing-syncing-displaying-mining-and-proving-work/)
- [Part 3 — Nodes that Mine](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
- [Part 4.1 — Bitcoin Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
- [Part 4.2 — Ethereum Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/21/how-to-build-your-own-blockchain-part-4-2-ethereum-proof-of-work-difficulty-explained/)

## 长话短说

如果你想知道挖矿如何工作，那么在这里学不到。现在，去读第一部分文章做的介绍，一直等到后面更多的涉及高级挖矿。

最后，我将展示创建节点的方法，即在运行中询问其他节点它们正在用什么样的链，以及应能够存储到本地用于开始挖矿。就这么多。为什么本文这么长？因为为了使程序将来运行更简易而涉及很多内容。
在最后，我会展示如何在开始挖矿前与其它节点通信，获取区块链信息并存储在本地。为什么文章会这么长呢？这是因为考虑到如何能够将来在这个应用上工作更容易，所涉及到的东西是非常多的。

话虽这么说，这里的同步方法并不是超级先进的。我继续改进链中涉及的类，测试新功能，简洁地创建其他节点，最后是节点在开始运行时进行同步的方法。

本节中，我将讨论和展示代码，准备开始。

## 扩展Block类和增加Chain类

这个项目是`OOP`优势的大好例子。本节中，我将开始讨论`Block`类的变动，然后去添加`Chain`类。

Blocks区块的关键点是：

1. 加入一种方式把块头中的键值从输入的值转换成我们要的类型。这样就能轻而易举的从文件写入json字典以及把`index,nonce,second timestamp`转换成`ints`。这可以更改后续的新变量。例如如果我想转换datetime成为datetimes对象，这个方法会很有用。
2. 通过检查hash是否以必要数量的零开头来验证块。
3. 块可以将自己保存在`chaindata`文件夹中。允许这样做而不需要工具函数来处理它。
4. 重载一些操作符函数
- `__repr__` 令debug时更易获取块的信息。打印时它重载了`__str__`方法。
- `__eq__` 允许您用==来比较两个块是否相等。否则Python将比较块的内存地址。我们想要的是数据对比。
- `__ne__`, 即非 `__eq__`
- 后续，我们可能要一些大于`__gt__`操作符，我们就能简单的用`>`来看那个链更好。但此刻，我们还没有好的方式来做。

```
############config.py

CHAINDATA_DIR = 'chaindata/'
BROADCASTED_BLOCK_DIR = CHAINDATA_DIR + 'bblocs/'

NUM_ZEROS = 5 #difficulty, currently.

BLOCK_VAR_CONVERSIONS = {'index': int, 'nonce': int, 'hash': str, 'prev_hash': str, 'timestamp': int}

###########block.py

from config import *

class Block(object):
  def __init__(self, *args, **kwargs):
    '''
      We're looking for index, timestamp, data, prev_hash, nonce
    '''
    for key, value in dictionary.items():
      if key in BLOCK_VAR_CONVERSIONS:
        setattr(self, key, BLOCK_VAR_CONVERSIONS[key](value))
      else:
        setattr(self, key, value)

    if not hasattr(self, 'hash'): #in creating the first block, needs to be removed in future
      self.hash = self.create_self_hash()
    if not hasattr(self, 'nonce'):
    #we're throwin this in for generation
    self.nonce = 'None'

.....

  def self_save(self):
    '''
      Want the ability to easily save
    '''
    index_string = str(self.index).zfill(6) #front of zeros so they stay in numerical order
    filename = '%s%s.json' % (CHAINDATA_DIR, index_string)
    with open(filename, 'w') as block_file:
      json.dump(self.to_dict(), block_file)

  def is_valid(self):
    '''
      Current validity is only that the hash begins with at least NUM_ZEROS
    '''
    self.update_self_hash()
    if str(self.hash[0:NUM_ZEROS]) == '0' * NUM_ZEROS:
      return True
    else:
      return False

  def __repr__(self): #used for debugging without print, __repr__ > __str__
    return "Block<index: %s>, <hash: %s>" % (self.index, self.hash)

  def __eq__(self, other):
    return (self.index == other.index and
            self.timestamp == other.timestamp and
            self.prev_hash == other.prev_hash and
            self.hash == other.hash and
            self.data == other.data and
            self.nonce == other.nonce)

  def __ne__(self, other):
    return not self.__eq__(other)
```
同样，这些操作将来很容易改变。

该讲链了。

1. 通过传入块列表来初始化链。如果你要创建空链，用`Chain([block_zero，block_one])`或`Chain([])`。
2. 有效性由索引递增1确定，prev_hash实际上是前一个块的哈希值，而哈希值具有有效零。
3. 将块保存到`chaindata`的功能
4. 能够通过索引或哈希在链中查找特定块
5. 可以通过`len(chain_obj)`调用得到链的长度即是`self.blocks`的长度
6. 链的相等性要求相等长度的块列表全部相等。
7. 大于，小于仅由链的长度决定
8. 是否能够将一个块添加到链中，目前是通过将其放到块变量的末尾，而不是检查有效性
9. 返回字典对象列表给链中所有的块。

```
from block import Block

class Chain(object):

  def __init__(self, blocks):
    self.blocks = blocks

  def is_valid(self):
  '''
    Is a valid blockchain if
    1) Each block is indexed one after the other
    2) Each block's prev hash is the hash of the prev block
    3) The block's hash is valid for the number of zeros
  '''
    for index, cur_block in enumerate(self.blocks[1:]):
      prev_block = self.blocks[index]
      if prev_block.index+1 != cur_block.index:
        return False
      if not cur_block.is_valid(): #checks the hash
        return False
      if prev_block.hash != cur_block.prev_hash:
        return False
    return True

  def self_save(self):
    '''
      We want to save this in the file system as we do.
    '''
    for b in self.blocks:
      b.self_save()
    return True

  def find_block_by_index(self, index):
    if len(self) <= index:
      return self.blocks[index]
    else:
      return False

  def find_block_by_hash(self, hash):
    for b in self.blocks:
      if b.hash == hash:
        return b
    return False

  def __len__(self):
    return len(self.blocks)

  def __eq__(self, other):
    if len(self) != len(other):
      return False
    for self_block, other_block in zip(self.blocks, other.blocks):
      if self_block != other_block:
        return False
    return True

  def __gt__(self, other):
    return len(self.blocks) > len(other.blocks)

.....

  def __ge__(self, other):
    return self.__eq__(other) or self.__gt__(other)

  def max_index(self):
    '''
      We're assuming a valid chain. Might change later
    '''
    return self.blocks[-1].index

  def add_block(self, new_block):
    '''
      Put the new block into the index that the block is asking.
      That is, if the index is of one that currently exists, the new block
      would take it's place. Then we want to see if that block is valid.
      If it isn't, then we ditch the new block and return False.
    '''
    self.blocks.append(new_block)
    return True

  def block_list_dict(self):
    return [b.to_dict() for b in self.blocks]
```
哦。很好，我在这里列出了很多Chain类的功能。后面，这些功能可能会发生变化，尤其是`__gt__`和`__lt__`比较。现在，它处理了我们正需的一些功能。

## 测试
我将在这里放上关于测试类的说明。可以用这些新类挖掘新区块。 一种测试方法是更改类，运行挖掘，观察错误，尝试修复，然后再次运行挖掘。测试代码的所有部分需要很久，特别是这时候，这是相当浪费时间的。

虽然有很多测试库可用，但它们却涉及一堆格式要求。我现在不想用它们，所以有一个`test.py`足够工作了。

为了在开始时列出块序列，我简单运行几次挖掘算法以获得块序列的有效链。从那里，我编写了测试类中不同部分的方法。如果我改变或添加某类，几乎不费时间为它编写测试。当我运行`python test.py`时，文件运行速度非常快，并告诉我错误来自哪一行。

```
from block import Block
from chain import Chain

block_zero_dir = {"nonce": "631412", "index": "0", "hash": "000002f9c703dc80340c08462a0d6acdac9d0e10eb4190f6e57af6bb0850d03c", "timestamp": "1508895381", "prev_hash": "", "data": "First block data"}
block_one_dir = {"nonce": "1225518", "index": "1", "hash": "00000c575050241e0a4df1acd7e6fb90cc1f599e2cc2908ec8225e10915006cc", "timestamp": "1508895386", "prev_hash": "000002f9c703dc80340c08462a0d6acdac9d0e10eb4190f6e57af6bb0850d03c", "data": "I block #1"}
block_two_dir = {"nonce": "1315081", "index": "2", "hash": "000003cf81f6b17e60ef1e3d8d24793450aecaf65cbe95086a29c1e48a5043b1", "timestamp": "1508895393", "prev_hash": "00000c575050241e0a4df1acd7e6fb90cc1f599e2cc2908ec8225e10915006cc", "data": "I block #2"}
block_three_dir = {"nonce": "24959", "index": "3", "hash": "00000221653e89d7b04704d4690abcf83fdb144106bb0453683c8183253fabad", "timestamp": "1508895777", "prev_hash": "000003cf81f6b17e60ef1e3d8d24793450aecaf65cbe95086a29c1e48a5043b1", "data": "I block #3"}
block_three_later_in_time_dir = {"nonce": "46053", "index": "3", "hash": "000000257df186344486c2c3c1ebaa159e812ca1c5c29947651672e2588efe1e", "timestamp": "1508961173", "prev_hash": "000003cf81f6b17e60ef1e3d8d24793450aecaf65cbe95086a29c1e48a5043b1", "data": "I block #3"}

###########################
#
# Block time
#
###########################

block_zero = Block(block_zero_dir)
another_block_zero = Block(block_zero_dir)
assert block_zero.is_valid()
assert block_zero == another_block_zero
assert not block_zero != another_block_zero

....
#####################################
#
# Bringing Chains into play
#
#####################################
blockchain = Chain([block_zero, block_one, block_two])
assert blockchain.is_valid()
assert len(blockchain) == 3

empty_chain = Chain([])
assert len(empty_chain) == 0
empty_chain.add_block(block_zero)
assert len(empty_chain) == 1
empty_chain = Chain([])
assert len(empty_chain) == 0

.....
assert blockchain == another_blockchain
assert not blockchain != another_blockchain
assert blockchain <= another_blockchain
assert blockchain >= another_blockchain
assert not blockchain > another_blockchain
assert not another_blockchain < blockchain

blockchain.add_block(block_three)
assert blockchain.is_valid()
assert len(blockchain) == 4
assert not blockchain == another_blockchain
assert blockchain != another_blockchain
assert not blockchain <= another_blockchain
assert blockchain >= another_blockchain
assert blockchain > another_blockchain
assert another_blockchain < blockchain
```
将来添加到test.py的功能包括所有独立运行的测试库。这会让你体会所有测试失败和而不是一次只能处理它们中的一个。另一种方法是将测试块dicts放在文件中而不是脚本中。例如，在测试目录中添加chaindata目录，以便我可以测试块的创建和保存。

## 对等点, 以及硬链接
The whole point of this part of the jbc is to be able to create different nodes that can run their own mining, own nodes on different ports, store their own chains to their own chaindata folder, and have the ability to give other peers their blockchain.

To do this, I want to share the files quickly between the folders so any change to one will be represented in the other blockchain nodes. Enter hard links.

At first I tried rsync, where I would run the bash script and have it copy the main files into a different local folder. The problem with this is that every change to a file and restart of the node would require me to ship the files over. I don’t want to have to do that all the time.

Hard links, on the other hand, will make the OS point the files in the different folder exactly to the files in my main jbc folder. Any change and save for the main files will be represented in the other nodes.

Here’s the bash script I created that will link the folders.

```
#!/bin/bash

port=$1

if [ -z "$port" ] #if port isn't assigned
then
  echo Need to specify port number
  exit 1
fi

FILES=(block.py chain.py config.py mine.py node.py sync.py utils.py)

mkdir jbc$port
for file in "${FILES[@]}"
do
  echo Syncing $file
  ln jbc/$file jbc$port/$file
done

echo Synced new jbc folder for port $port

exit 1
```
To run, $./linknodes 5001 to create a folder called jbc5001 with the correct files.

Since Flask runs initially on port 5000, I’m gong to use 5001, 5002, and 5003 as my initial peer nodes. Define them in config.py and use them when config is imported. As with many parts of the code, this will definitely change to make sure peers aren’t hardcoded. We want to be able to ask for new ones.

```
#config.py

#possible peers to start with
PEERS = [
          'http://localhost:5000/',
          'http://localhost:5001/',
          'http://localhost:5002/',
          'http://localhost:5003/',
        ]

#node.py

if __name__ == '__main__':

  if len(sys.argv) >= 2:
    port = sys.argv[1]
  else:
    port = 5000

  node.run(host='127.0.0.1', port=port)
```
Cool. In part 1, the Flask node already has an endpoint for sharing it’s blockchain so we’re good on that front, but we still need to write the code to ask our peer friends about what they’re working with.

Quick side, we’re using Flask and http. This works in our case, but a blockchain like Ethereum has their own protocol for broadcasting, the [Ethereum Wire Protocol](https://github.com/ethereum/wiki/wiki/Ethereum-Wire-Protocol).

## Syncing
When a node or mine starts, the first requirement is to sync up to get a valid blockchain to work from. There are two parts to this – first, getting that node’s local blockchain it has in its (fix the it’s to be its) chaindata dtr, and second, the ability to ask your peers what they have.

There’s a big part of syncing that I’m ignoring for now — determining which chain is the best to work with. Remember back in the __gt__ method in the Chain class? All I’m doing is seeing which chain is longer than the other. That’s not going to cut it when we’re dealing with information being locked in and unchangeable.

Imagine if a node starts mining on its own, goes off in a different direction and doesn’t ask peers what they’re working on, gets lucky to create new valid blocks with its own data, and ends up longer than the others but wayy different blocks. Should they be considered to have the most valid and correct block? Right now, I’m going to stick with length. This’ll change in a future part of this project.

Another part of the code below to keep in mind is how few error checks I have. I’m assuming the files and block dicts are valid without any attempted changes. Important for production and wider distribution, but not for now.

Foreward: sync_local() is pretty much the same as the initial sync_local() from part 1 so I’m not going to include the difference here. The new sync_overall.py asks peers what they have for the blockchains and determines the best by chain length.

```
from block import Block
from chain import Chain
from config import *
from utils import is_valid_chain

.....

def sync_overall(save=False):
  best_chain = sync_local()
  for peer in PEERS:
    #try to connect to peer
    peer_blockchain_url = peer + 'blockchain.json'
    try:
      r = requests.get(peer_blockchain_url)
      peer_blockchain_dict = r.json()
      peer_blocks = [Block(bdict) for bdict in peer_blockchain_dict]
      peer_chain = Chain(peer_blocks)

      if peer_chain.is_valid() and peer_chain > best_chain:
        best_chain = peer_chain

    except requests.exceptions.ConnectionError:
      print "Peer at %s not running. Continuing to next peer." % peer
  print "Longest blockchain is %s blocks" % len(best_chain)
  #for now, save the new blockchain over whatever was there
  if save:
    best_chain.self_save()
  return best_chain

def sync(save=False):
  return sync_overall(save=save)
```
Now when we run node.py we want to sync the blockchain we’re going to use. This involves initially asking peers what they’re woking with and saving the best. Then when we get asked about our own blockchain we’ll use the local-saved chain. node.py requires a couple changes. Also see how much simpler the blockchain function is than from part 1? That’s why classes are great to use.

```
#node.py
.....
sync.sync(save=True) #want to sync and save the overall "best" blockchain from peers

@node.route('/blockchain.json', methods=['GET'])
def blockchain():
  local_chain = sync.sync_local() #update if they've changed
  # Convert our blocks into dictionaries
  # so we can send them as json objects later
  json_blocks = json.dumps(local_chain.block_list_dict())
  return json_blocks

.....
```
Testing time! We want to do the following to show that our nodes are syncing up.

1. Create an initial generation and main node where we mine something like 6 blocks.
2. Hard link the files to create a new folder with a node for a different port.
3. Run node.py for the secondary node and view the blockchain.json endpoint to see that we don’t have any blocks for now.
4. Run the main node.
5. Shutdown and restart the secondary node.
6. Check to see if the secondary node’s chaindata dir has the block json files, and see that its /blockchain.json enpoint is now showing the same chain as the main node.
I suppose I could share screenshots of this, or create a video that depicts me going through the process, but you can believe me or take the git repo and try this yourself. Did I mention that programming > only reading?

## That’s it, for now
As you read above and as you read from reading this post all the way to the bottom, I didn’t talk about syncing with other nodes after mining. That’ll be Part 3. And then after Part 3 there’s still a bunch to go. Things like creating a Header class that advances the information and structure of the header rather than only having the Block object.

How about the bigger and more correct logic behind determining what the correct blockchain is when you sync up? Huge part I haven’t addressed yet.

Another big part would be syncing data transactions. Right now all we have are dumb little sentences for data that tell people what index the block is. We need to be able to distribute data and tell other nodes what to include in the block.

Tons to go, but we’re on our way. Again, [twitter](https://twitter.com/jack_schultz), [contact](https://bigishdata.com/contact/), [github repo](https://github.com/jackschultz/jbc/tree/part-2) for Part 2 branch. Catch y’all next time.