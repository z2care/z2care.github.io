---
layout: post
title: "强化学习框架PARL总结"
date: 2020-06-24 16:05:00 +0800
categories: 学习与总结
tag: MachineLearning
---

* content
{:toc}


这门课算是普及知识的课程，并不能算作算法研究和提高编程技能的课程，而且更多的是理解什么是RL，PARL如何使用，做fine-tuning和dnn修改的工作。其实如果能理解PL和熟悉PARL使用，已经足够这门课的价值了。

<!-- more -->

在人工智能火热发展的时代，机器学习作为有效的实现方式，其分支也在百家争鸣百花齐放。曾经的BAT在经过www、web2.0、mobile的洗礼后，来到新的战场——AI。

Baidu作为曾经的B，在www、web2.0时代曾拔得头筹，但在mobile时代一蹶不振，未能突破AT的包围，被ByteDance替代。在AI如火如荼的今天，Baidu顺势发力，希望能借AI的东风重新回高光时刻。

因此Baidu开发了一系列机器学习的项目，包括核心库PaddlePaddle（飞桨）、学习系统AIStudio、以及数据集和模型库等等。当然，官方文档和培训课程也是必不可少的，99.9%的（只有一个企业级培训是收费的）课程是免费的，还有丰富的物质奖励。

偶然的机会（我也忘了是怎么知道的）获得PARL强化学习的免费学习机会，于是开始了知道我不知道PARL的历程。

## 一、课程预习中的知识点

### 1. 新手入门第一课-什么是深度学习？

![image](https://ai-studio-static-online.cdn.bcebos.com/60ba91a3d2c4427d82b933b25e490275e993d3e75f3149269a6d07efd3ff2067)

- 人工智能是计算机科学的一个分支，研究计算机中智能行为的仿真。
- 机器学习是实现人工智能的一种手段，也是目前被认为比较有效的实现人工智能的手段
- 深度学习是一种机器学习方法 ， 它允许我们训练人工智能来预测输出，给定一组输入(指传入或传出计算机的信息)

1. 神经网络结构
2. 梯度下降法
3. 神经网络的小例子

### 2.新手入门第二课-必备数学知识

理论是武器，没有武器怎么打仗

### 3.新手入门第三课-Python快速入门

Python是门非常容易学的语言，但不是最佳编程入门语言，却是实现机器学习的最佳语言

### 4.新手入门第四课-PaddlePaddle快速入门

PaddlePaddle是Baidu的深度学习框架，核心框架，类似Tensorflow/Pytorch

### 5. AI Studio基本操作-Notebook篇

AI Studio Notebook蛮强大的，后端有服务器支持，免费的GPU可以薅

## 二、PARL强化学习公开课Lesson1

强化学习（英语：Reinforcement learning，简称RL）是机器学习中的一个领域，强调如何基于环境而行动，以取得最大化的预期利益。

核心思想：智能体agent在环境environment中学习，根据环境的状态state（或观测到的observation），执行动作action，并根据环境的反馈 reward（奖励）来指导更好的动作。

下面这个图是RL的精髓，最后有个小例子，真的很有意思。

![image](https://ai-studio-static-online.cdn.bcebos.com/992844ad9a8e478b8a4490e7eb6d3a6398b6e2b8fd5c40998a02eef5fceb9fc3)

强化学习与监督学习的区别：
- 强化学习、监督学习、非监督学习是机器学习里的三个不同的领域，都跟深度学习有交集。
- 监督学习寻找输入到输出之间的映射，比如分类和回归问题。
- 非监督学习主要寻找数据之间的隐藏关系，比如聚类问题。
- 强化学习则需要在与环境的交互中学习和寻找最佳决策方案。
- 监督学习处理认知问题，强化学习处理决策问题。

## 三、基于表格型方法求解RL

PARL强化学习公开课 Lesson2_Sarsa

> Sarsa全称是state-action-reward-state'-action'，目的是学习特定的state下，特定action的价值Q，最终建立和优化一个Q表格，以state为行，action为列，根据与环境交互得到的reward来更新Q表格，更新公式为：

![image](https://ai-studio-static-online.cdn.bcebos.com/776b473b7f994702a3e05c5eac1156a7ce03b9e6bdb5453085fa9cbb86979715)

Sarsa在训练中为了更好的探索环境，采用ε-greedy方式来训练，有一定概率随机选择动作输出。

PARL强化学习公开课Lesson2_Q_learning

1. Q-learning也是采用Q表格的方式存储Q值（状态动作价值），决策部分与Sarsa是一样的，采用ε-greedy方式增加探索。
2. Q-learning跟Sarsa不一样的地方是更新Q表格的方式。
3. Sarsa是on-policy的更新方式，先做出动作再更新。
4. Q-learning是off-policy的更新方式，更新learn()时无需获取下一步实际做出的动作next_action，并假设下一步动作是取最大Q值的动作。

Q-learning的更新公式为：
![image](https://ai-studio-static-online.cdn.bcebos.com/38158582039041edad0a5a704ba792d0e464f2eb8a394577bf88051cc52d6b66)

## 四、基于神经网络方法求解RL

PARL强化学习公开课Lesson3_DQN

上节课介绍的表格型方法存储的状态数量有限，当面对围棋或机器人控制这类有数不清的状态的环境时，表格型方法在存储和查找效率上都受局限，DQN的提出解决了这一局限，使用神经网络来近似替代Q表格。

>本质上DQN还是一个Q-learning算法，更新方式一致。为了更好的探索环境，同样的也采用ε-greedy方法训练。

在Q-learning的基础上，DQN提出了两个技巧使得Q网络的更新迭代更稳定。

- 经验回放 Experience Replay：主要解决样本关联性和利用效率的问题。使用一个经验池存储多条经验s,a,r,s'，再从中随机抽取一批数据送去训练。
- 固定Q目标 Fixed-Q-Target：主要解决算法训练不稳定的问题。复制一个和原来Q网络结构一样的Target Q网络，用于计算Q目标值。

## 五、基于策略梯度求解RL

策略梯度方法求解RL——Policy Gradient

### 1. Policy Gradient简介

在强化学习中，有两大类方法，一种基于值（Value-based），一种基于策略（Policy-based）
- Value-based的算法的典型代表为Q-learning和SARSA，将Q函数优化到最优，再根据Q函数取最优策略。
- Policy-based的算法的典型代表为Policy Gradient，直接优化策略函数。

采用神经网络拟合策略函数，需计算策略梯度用于优化策略网络。

优化的目标是在策略π(s,a)的期望回报：所有的轨迹获得的回报R与对应的轨迹发生概率p的加权和，当N足够大时，可通过采样N个Episode求平均的方式近似表达。

![image](https://ai-studio-static-online.cdn.bcebos.com/eb184ddf8dcc4dc3b528a105f8d8e3ea6487d4905bc04cdebd7725f2d6a2752f)

优化目标对参数θ求导后得到策略梯度：

![image](https://ai-studio-static-online.cdn.bcebos.com/326d8abe040347cea25e4c0be3e09015e85cb818a02c445483381540ab1d238c)

### 2. Policy Gradient实践——REINFORCE算法

使用REINFORCE解决 连续控制版本的CartPole问题，向小车提供推力使得车上的摆杆倒立起来。

## 六、连续动作空间上求解RL

PARL强化学习公开课Lesson5_DDPG

### 1. DDPG简介

- DDPG的提出动机其实是为了让DQN可以扩展到连续的动作空间。
- DDPG借鉴了DQN的两个技巧：经验回放 和 固定Q网络。
- DDPG使用策略网络直接输出确定性动作。
- DDPG使用了Actor-Critic的架构。

### 2. DDPG实践

使用DDPG解决连续控制版本的CartPole问题，给小车一个力（连续量）使得车上的摆杆倒立起来。
