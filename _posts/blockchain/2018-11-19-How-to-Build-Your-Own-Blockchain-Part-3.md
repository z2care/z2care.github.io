---
layout: post
title: "如何构建自己的区块链 第三部分 - 编写挖矿和沟通的节点"
date: 2018-11-13 23:56:00 +0800
categories: 开发与技术
tag: Blockchain
---

独立翻译自:[How to Build Your Own Blockchain Part 3 — Writing Nodes that Mine and Talk](https://bigishdata.com/2017/11/02/build-your-own-blockchain-part-3-writing-nodes-that-mine/)
<!-- more -->
* content
{:toc}

大家好欢迎来到构建区块链第三部分，快速回顾[第一部分](https://bigishdata.com/2017/10/17/write-your-own-blockchain-part-1-creating-storing-syncing-displaying-mining-and-proving-work/)， 我编写了代码，遍历了顶级的数学和单个节点挖掘自己的区块链的需求;我创建具有有效信息的新块，将它们保存到文件夹中，然后再开始挖掘新块。[第二部分](https://bigishdata.com/2017/10/27/build-your-own-blockchain-part-2-syncing-chains-from-different-nodes/)介绍了多个节点以及它们具有同步功能。如果节点1自己进行挖掘，而节点2希望获取节点1的区块链，那么现在可以这样做了。

在第3部分中，阅读下面的“长话短说”，看看我们将能得到什么。然后阅读文章的其余部分，以了解(希望如此)这一切是如何发生的。

## 本系列其他文章
- [Part 1 — Creating, Storing, Syncing, Displaying, Mining, and Proving Work](https://bigishdata.com/2017/10/17/write-your-own-blockchain-part-1-creating-storing-syncing-displaying-mining-and-proving-work/)
- [Part 2 — Syncing Chains From Different Nodes](https://bigishdata.com/2017/10/27/build-your-own-blockchain-part-2-syncing-chains-from-different-nodes/)
- [Part 4.1 — Bitcoin Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/13/how-to-build-a-blockchain-part-4-1-bitcoin-proof-of-work-difficulty-explained/)
- [Part 4.2 — Ethereum Proof of Work Difficulty Explained](https://bigishdata.com/2017/11/21/how-to-build-your-own-blockchain-part-4-2-ethereum-proof-of-work-difficulty-explained/)

# 长话短说
节点将竞争看谁因挖掘一个块而获得荣誉。这是一个比赛!为了做到这一点，我们调整`mine.py`来检查是否有有效的块，方法是只检查一部分nonce值，而不是在匹配之前检查所有的nonces值。然后APScheduler将处理使用不同nonce范围运行的挖掘作业。如果我们想要让`node.py`像个Flask的web服务一样挖矿，我们就将挖掘功能转移到后台。最后，我们有了不同的节点竞争第一次挖掘并广播他们挖掘到的块!

在我们开始之前, 要是你想检查所有的事就[参考github上代码](https://github.com/jackschultz/jbc).这里有一些代码片段来说明我做了什么，但是如果你想看完整的代码，请到github上看。这段代码对我很有用，但我也在清理所有东西，并编写一个可用的README，以便人们可以自己克隆和运行它。如果像联系我，[Twitter](https://twitter.com/jack_schultz),和[联系方式](https://bigishdata.com/contact/) 。

# 用APScheduler挖掘和再次挖掘
这里的第一步是调整挖掘，以便在另一个节点找到了它正在处理的索引块时能够停止挖掘。
在第1部分中，挖掘是一个while循环，只有在找到有效的nonce时才会中断。当我们被告知另一个节点的成功，我们需要停止挖掘的能力。

我不会在这里忽悠人，我花了一段时间才找到最好的方法。阅读下面的列表，了解我的各种想法，以及它们为什么不奏效。

1. 在APScheduler上运行，当我们的Flask应用程序遇到来自不同节点的块时，停止对该索引的挖掘作业，并开始对索引+ 1进行挖掘。
2. 这似乎就是要做的事情，但我发现APScheduler的`remove_job()`函数在任务已经从队列中取出并运行时无法移除。所以我就无法停止挖掘。
3. 用Celery? Celery肯定可以工作，我研究过它，但是坦率地说，从一个基本用例开始需要大量的代码和配置。参考[介绍页面](http://docs.celeryproject.org/en/latest/getting-started/first-steps-with-celery.html)你们可以看到我在说什么。Celery肯定是一个选择，我不是说它不值得，但在这种情况下，它开销太大。
4. Gevent?[我以前用过](https://bigishdata.com/2017/05/11/general-tips-for-web-scraping-with-python/)，用它来一次抓取一堆页面，而不必等待请求返回。使收集页面更快。在这种情况下，它可以工作，但主要用于队列而不是单个作业。我不会一次挖掘一堆块，所以不需要创建多个Gevent会运行的块。我只需要有一个。
5. [rq](http://python-rq.org/)?不行，在停止运行中的作业上有同样的问题。
6. 在做了一些其他实现的调研后,比如在github上查看[以太坊Python库](https://github.com/ethereum/pyethapp/blob/05d8037cd7ebb085b24af41b185eaef3385f88e8/pyethapp/pow_service.py),我发现它们挖掘时用到了术语`rounds`和`starting_nonce`.我们不会使用一个完整的while循环来遍历nonces直到找到正确的循环为止的方式，我们只会在`[starting_nonce:starting_nonce+round]`的值中检查nonces。如果我们成功了，我们就移动到下一个块。否则，我们就用不同的`starting_nonce`值再次尝试。
7. 当调用挖掘作业时，它一次只检查特定数量的nonces。如果成功，它将返回有效的nonce和hash。如果没有，则返回None。把这个想法和APScheduler结合起来，我们似乎达实现了功能。
8. 然后我意识到，既然我们有了这些功能，我就可以使用上面列出的任何库来进行挖掘。如果它允许你对作业进行排队，那么当您将挖掘分解为不同的部分时，它就好用。实际上，一个好的文章应该是使用所有工作的库来实现挖掘。如果你需要，请告诉我。
下面是`mine.py`重写的代码。重点如下。 

- 当你运行`mine.py`时，你只想要去挖掘，我们将要使用的APSchedule是‘BlockingScheduler’。稍后你讲看到我们使用`BackgroundScheduler`。
- 下面有多个函数用于具有不同初始级别的挖掘。你可以对链上的下一个块进行挖掘，对一个块后的块进行挖掘，或者尝试为您刚刚生成的块找到有效的nonce而挖掘。
- 除了用于挖掘块的预定任务之外，我们还有一个侦听器用来检查返回值，查看是否继续使用上面第4（原文有误，应是6）节中描述的范围内的nonces来挖掘当前块，或者转到下一个块。

```
#mine.py

import apscheduler
from apscheduler.schedulers.blocking import BlockingScheduler

#if we're running mine.py, we don't want it in the background
#because the script would return after starting. So we want the
#BlockingScheduler to run the code.
sched = BlockingScheduler(standalone=True)

import logging
import sys
logging.basicConfig(stream=sys.stdout, level=logging.DEBUG)

STANDARD_ROUNDS = 100000

def mine_for_block(chain=None, rounds=STANDARD_ROUNDS, start_nonce=0):
  if not chain:
    chain = sync.sync_local() #gather last node

  prev_block = chain.most_recent_block()
  return mine_from_prev_block(prev_block, rounds=rounds, start_nonce=start_nonce)

def mine_from_prev_block(prev_block, rounds=STANDARD_ROUNDS, start_nonce=0):
  #create new block with correct
  new_block = utils.create_new_block_from_prev(prev_block=prev_block)
  return mine_block(new_block, rounds=rounds, start_nonce=start_nonce)

def mine_block(new_block, rounds=STANDARD_ROUNDS, start_nonce=0):
  #Attempting to find a valid nonce to match the required difficulty
  #of leading zeros. We're only going to try 1000
  nonce_range = [i+start_nonce for i in range(rounds)]
  for nonce in nonce_range:
    new_block.nonce = nonce
    new_block.update_self_hash()
    if str(new_block.hash[0:NUM_ZEROS]) == '0' * NUM_ZEROS:
      print "block %s mined. Nonce: %s" % (new_block.index, new_block.nonce)
      assert new_block.is_valid()
      return new_block, rounds, start_nonce

  #couldn't find a hash to work with, return rounds and start_nonce
  #as well so we can know what we tried
  return None, rounds, start_nonce

def mine_for_block_listener(event):
  new_block, rounds, start_nonce = event.retval
  #if didn't mine, new_block is None
  #we'd use rounds and start_nonce to know what the next
  #mining task should use
  if new_block:
    print "Mined a new block"
    new_block.self_save()
    sched.add_job(mine_from_prev_block, args=[new_block], kwargs={'rounds':STANDARD_ROUNDS, 'start_nonce':0}, id='mine_for_block') #add the block again
  else:
    print "No dice mining a new block. Restarting with different nonce range"
    sched.add_job(mine_for_block, kwargs={'rounds':rounds, 'start_nonce':start_nonce+rounds}, id='mine_for_block') #add the block again
sched.print_jobs()

if __name__ == '__main__':

  sched.add_job(mine_for_block, kwargs={'rounds':STANDARD_ROUNDS, 'start_nonce':0}, id='mine_for_block') #add the block again
  sched.add_listener(mine_for_block_listener, apscheduler.events.EVENT_JOB_EXECUTED)#, args=sched)
  sched.start()
  ```
当我们运行这个代码时，节点将会成功地挖掘，但是是在不同的作业中而不是只有一个作业。这是伟大的开端。

# 节点挖掘
我要添加的下一部分是`node.py`的功能，Flask节点也可以运行挖掘。就像我上面说的，运行`mine.py`将只进行挖掘，但我们需要在Flask节点下面的后台运行挖掘任务。为此，我们加载`BackgroundScheduler`，告诉导入的挖掘任务我们使用的是BackgroundScheduler调度器，而不是那个文件中的调度器，然后像以前一样添加作业和侦听器。

当我们运行这个代码时，我们会看到输出是一样的，我们有一个关于正在运行的作业的日志记录，并且能打开`blockchain.json`，刷新它，就能看到新的节点出来了。

```
#node.py
.....
import mine
.....
from apscheduler.schedulers.background import BackgroundScheduler
sched = BackgroundScheduler(standalone=True)
.....

if __name__ == '__main__':

.....

  mine.sched = sched #to override the BlockingScheduler in this case, sched is the BackgroundSchedule
  sched.add_job(mine.mine_for_block, kwargs={'rounds':STANDARD_ROUNDS, 'start_nonce':0}, id='mine_for_block') #add the block again
  sched.add_listener(mine.mine_for_block_listener, apscheduler.events.EVENT_JOB_EXECUTED)
  sched.start()

  node.run(host='127.0.0.1', port=port)
```
# 命令行解析
中场休息时间!以前，为了查看我们希望节点运行的端口，我只检查是否传递了一个参数，如果传递了，那就是端口。

```
if __name__ == '__main__':
  if len(sys.argv) >= 2:
    port = sys.argv[1]
  else:
    port = 5000
```
很简单，但太简单了。现在，由于我让节点也可以进行挖掘，所以我希望能够指定节点是否应该进行挖掘。

打开[argparse](https://docs.python.org/2.7/library/argparse.html)。它非常好，只需要几行就可以定义我要寻找的参数。我们希望能够指出默认运行到5000的端口，并且也能够指明是否需要挖掘。

```
if __name__ == '__main__':
  #args!
  parser = argparse.ArgumentParser(description='JBC Node')
  parser.add_argument('--port', '-p', default='5000',
                 help='what port we will run the node on')
  parser.add_argument('--mine', '-m', dest='mine', action='store_true')
  args = parser.parse_args()

  #only mine if we want to
  if args.mine:
    mine.sched = sched #to override the BlockingScheduler in the
    #in this case, sched is the background sched
    sched.add_job(mine.mine_for_block, kwargs={'rounds':STANDARD_ROUNDS, 'start_nonce':0}, id='mining') #add the block again
    sched.add_listener(mine.mine_for_block_listener, apscheduler.events.EVENT_JOB_EXECUTED)#, args=sched)
  
  sched.start() #want this to start so we can validate on the schedule and not rely on Flask

  #now we know what port to use
  node.run(host='127.0.0.1', port=args.port)
```

例如,`python node.py -m` 将会在5000端口上运行节点和挖掘。`python node.py -p 5001` 将会再5001端口上运行节点和挖掘。`python --port=5002 -m`将会在5000端口上运行节点和挖掘。你懂了吧。

好的，回到挖掘。

# 监听节点
为了广播一个节点，我们需要一个Flask后端来接受block的dicts。我们不希望用节点运行验证工作——我们希望将此工作放入计划中。

首先，验证将检查它是否是一个有效的块，如果是，则保存它。

```
#node.py
@node.route('/mined', methods=['POST'])
def mined():
  possible_block_dict = request.get_json()
  sched.add_job(mine.validate_possible_block, args=[possible_block_dict], id='validate_possible_block') #add the block again
  return jsonify(received=True)

#mine.py

def validate_possible_block(possible_block_dict):
  possible_block = Block(possible_block_dict)
  if possible_block.is_valid():
    possible_block.self_save()
    return True
return False
```

为了进行测试，我们在一个端口上运行一个节点来进行挖掘，另一个节点在那里等待广播。我们观察非挖掘节点的chaindir，并将看到由其对等节点挖掘的节点出现。

一方面，与不同的区块链相比，我们使用Flask和http进行通信的方式非常简单。看看[以太坊如何让节点互相通信](https://github.com/ethereum/wiki/wiki/JSON-RPC)。我想提一下，这样读者就不会认为所有区块链都会在http上来回传递简单的json信息。还有更好的方法。

# 竞争挖掘节点
到最后了!这篇文章的重点是拥有多个节点，它们都在挖掘，第一个节点得到一个有效的块广播给其他节点，它们都需要接收块，并开始挖掘下一个节点。

还记得上面我说过我们想要`validate_possible_block`作为一个作业吗?这是因为我们希望在没有挖掘作业运行时检查验证。如果块是有效的，我们希望删除调度队列中的挖掘作业。由于侦听器将nonce范围递增的下一个挖掘作业插入到队列中，因此我们删除该作业，然后在新的有效块之后插入一个挖掘区块作业，然后从那里开始。

```
def validate_possible_block(possible_block_dict):
  possible_block = Block(possible_block_dict)
  if possible_block.is_valid():
    #this means someone else won
    possible_block.self_save()
    #we want to kill and restart the mining block so it knows it lost
    try:
      sched.remove_job('mining') 
      print "removed running mine job in validating possible block"
    except apscheduler.jobstores.base.JobLookupError:
      print "mining job didn't exist when validating possible block"
    print "readding mine for block validating_possible_block"
    sched.add_job(mine_for_block, kwargs={'rounds':STANDARD_ROUNDS, 'start_nonce':0}, id='mining') #add the block again

    return True
  return False
```

当运行代码并查看节点时，要查看哪个节点获得了该块的挖掘并不简单。本文不讨论块中的数据，但是我们需要一个简单的方法来告诉世界哪个节点赢了。

在`node.py`中，我们要写一个`data.txt`文件，其中包含由该端口上的节点挖掘的块的数据。然后在`utils.py`中，我们创建一个未挖掘的、无效的块，我们读取数据文件并将其输入到头文件中。

```
#node.py

if __name__ == '__main__':
  .....
  filename = '%sdata.txt' % (CHAINDATA_DIR)
  with open(filename, 'w') as data_file:
    data_file.write("Mined by node on port %s" % args.port)
  .....

#utils.py

def create_new_block_from_prev(prev_block=None):
  if not prev_block:
  #index zero and arbitrary previous hash
    index = 0
    prev_hash = ''
  else:
    index = int(prev_block.index) + 1
    prev_hash = prev_block.hash

  filename = '%sdata.txt' % (CHAINDATA_DIR)
  with open(filename, 'r') as data_file:
    data = data_file.read()
  nonce = 0
  timestamp = datetime.datetime.utcnow().strftime('%Y%m%d%H%M%S%f')
  block_info_dict = dict_from_block_attributes(index=index, timestamp=timestamp, data=data, prev_hash=prev_hash, nonce=nonce)
  print block_info_dict
  new_block = block.Block(block_info_dict)
  return new_block
```
另外，当我们试图在区块链上保存实际数据时，拥有一个包含数据信息的文件似乎是一件好事。嗯…。

当你启动这两个节点时，`$python node.py -m`以及在5001硬链接目录中执行`$python node.py -p 5001 -m`。我们可以观察所有的日志记录，等待节点被写到每个`chaindata`目录，然后看到新的节点同时出现，以及不同的节点获胜。

用四个节点竞争的例子运行这个程序，坐在那里观看chaindir并查看哪个节点挖掘了出现的下一个块是非常棒的事。观看你编写的代码运行起来是编程中最好的感觉之一。记住这一点。

下面是我在端口5000和5001上使用节点运行挖掘时的几个屏幕截图。5000端口赢得了9个块，5001端口赢得了7个。

![image](https://bigishdata.files.wordpress.com/2017/11/screen-shot-2017-11-01-at-9-14-48-pm.png)

5000端口

![image](https://bigishdata.files.wordpress.com/2017/11/screen-shot-2017-11-01-at-9-14-35-pm.png)

5001端口

# 结论
看看这个，我们有能力让一组分布式节点相互竞争，尝试挖掘块，为挖掘获得酬劳，并能够在其他块想要运行和挖掘区块链时与它们同步。这条线很长。但还有更多的工作要做。

每个块中唯一的数据是哪个节点赢得了挖掘。区块链的概念是分布式数据存储(如果您听说过术语“账本”)。我们需要更多的数据。区块链只会继续使用广播的第一个节点。如果两个节点同时找到一个有效的节点会怎样?我们如何确定要使用哪个块?沿着这些思路，我们如何确保链的起始块永远不变?

我认为本系列的下一个主要部分是处理block header。现在它把字符串连在一起然后计算哈希值。如果你看看比特币和Ethereum区块链，它们有更复杂的头信息，这样矿工们就能准确地知道哪些区块是有效的，无论它们在哪里计算。另一个问题是存储散列所需的难度。现在它在配置中。

这将是下一个。

如果你还在阅读，请随时与我联系，向我提问或问好。我喜欢和人们交流。
 [Twitter](https://twitter.com/jack_schultz), [contact](https://bigishdata.com/contact/), [Github](https://github.com/jackschultz/jbc). 下次再见。