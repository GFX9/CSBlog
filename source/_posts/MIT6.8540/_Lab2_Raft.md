---
title: 'MIT6.8540(6.824) Lab2: Raft'
date: 2023-12-26 15:29:17
- 'CS课程笔记'
- 'MIT6.5840(6.824) 2023'
- 'Lab笔记'
tags:
- '分布式系统'
- 'Go'
---
Raft
1. Raft organizes client requests into a sequence, called the log, and ensures that all the replica servers see the same log
2. Each replica executes client requests in log order
3. If a server fails but later recovers, Raft takes care of bringing its log up to date. 

- log entries
  Your Raft interface will support an indefinite sequence of numbered commands, also called log entries. The entries are **numbered with index numbers**