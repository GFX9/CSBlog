---
title: 'MIT6.5840(6.824) Lec06笔记: raft论文解读2: 恢复、持久化和快照'
date: 2024-01-12 09:19:15
category: 
- 'CS课程笔记'
- 'MIT6.5840(6.824) 2023'
- 课程笔记
tags:
- '分布式系统'
- 'Raft'
---

课程主页: https://pdos.csail.mit.edu/6.824/schedule.html

本节课是介绍`Raft`共识算法的第一部分, 建议阅读[论文](https://pdos.csail.mit.edu/6.824/papers/gfs.pdf), 如果要做`Lab`的话, 论文是一定要看的, 尤其是要吃透论文中的图2。

本节课介绍的内容包括: 日志恢复、选举限制、持久化、快照和一致性。

# 1 日志恢复
## 1.1 日志恢复的案例
为了举例说明`Raft`是如何进行日志恢复的, 我们假设有如下表格的情形:

|节点\log索引|10|11|12|13|
|:---|---:|---:|---:|---:|
|S1|3||||
|S2|3|3|4||
|S3|3|3|5||

表格中存在3个节点`S1,S2,S3`, 存放的值是每个日志槽位的`Term`。 假设其在日志槽位`0-9`都保持完全一致。以下说明了为什么某一时刻会出现上表的状态：

1. 到槽位`10`的日志为止， 所有的节点网络正常且工作正常, 此时最高`Term`为3, `Leader`为`S3`
2. 此后`S1`故障, 因此槽位`11`的`log`只被复制在了`S2`和`S3`
3. 之后`S2,S3`之间的网络也出现了故障`S2`进行选举
4. 恰好选举时`S1`恢复了, `S1,S2`都为`S2`投票, `S2`因为获得了超过半数投票而被选为新`Leader`, 新的`Term`为4
5. 新`Leader S2`向自身槽位12处追加一个`Term`为4的`log`, 然后故障了, 因此槽位12处`Term`为4的`log`只存在于`S2`
6. 此时`S3`触发选举超时, 并获取了`S1`和`S3`的选票成为新`Leader`, 新的`Term`为5
7.  新`Leader S3`向自身槽位12处追加一个`Term`为5的`log`, 然后准备开始广播, 并且广播前`S2`也恢复了

以上就是为什么会出现表中所示的内容的原因, 而`Raft`的日志恢复的目的是, 将`Leader`的日志强行复制到其他节点, 介绍这个机制前, 需要引入`Leader`维护的几个变量和`AppendEntries RPC`中的参数:

**`Leader`维护的变量**
- `nextIndex[]`: `Leader`认为下一个追加的日志在每个节点的索引
- `matchIndex[]`: `Leader`认为每个节点中已经复制的日志项的最高索引

**`AppendEntries RPC`中的参数**
- `term`: `Leader`的任期
- `leaderId`: `Leader`的`id`
- `prevLogIndex`: 追加的新日志前的日志索引
- `prevLogTerm`: 追加的新日志前的日志的`Term`
- `entries[]`: 日志项切片
- `leaderCommit`: `Leader`记录的已经`commit`的日志项的最高索引

因此, 按照论文[Figure 2](../../images/raft-figure2.png)中的描述, 新的`Leader`将把`nextIndex[]`初始化为自身日志数组长度, 发送时的`PrevLogIndex`就是`nextIndex[i] - 1`, 因此`Leader S3`向`S1`和`S2`发送的`AppendEntries RPC`为:

```go
args := &AppendEntriesArgs{
    Term:         5,
    LeaderId:     3,
    PrevLogIndex: 11,
    PrevLogTerm:  3,
    LeaderCommit: 1,
    Entrie:       ...,
}
```

因此, `Follower`收到`AppendEntries RPC`会根据`PrevLogIndex`, `PrevLogTerm`进行检查:
- `S1`发现自己没有`槽位11`的`log`, 返回`false`
- `S2`发现自己有`槽位11`的`log`, 其`Term`为3也与`AppendEntriesArgs`匹配, 因此其使用`AppendEntriesArgs`中的覆盖原来槽位12处的`log`, 返回`true`


`Leader S3`收到`S1`和`S2`回复的`AppendEntries RPC`后, 会做如下处理:
1. 发现`S2`回复了`true`, 因此将`S2`的`matchIndex[S2]`设置为`PrevLogIndex+len(Entries)`, 将`nextIndex[S2]`设置为`matchIndex[S2]+1`
2. 发现`S1`回复了`false`, 于是将其`nextIndex[S1]`自减, 再次发送的`AppendEntries RPC`为:

```go
args := &AppendEntriesArgs{
    Term:         5,
    LeaderId:     3,
    PrevLogIndex: 10,
    PrevLogTerm:  3,
    LeaderCommit: 1,
    Entrie:       ...,
}
```

这时`S1`发现自己有`槽位10`的`log`, 其`Term`也与`AppendEntriesArgs`匹配, 因此进行追加并返回`true`, `Leader S3`按照相同的逻辑处理`nextIndex[S1]`和`matchIndex[S1]`

## 1.2 日志恢复的逻辑
从上述的日志恢复的机制我们可以看出, `Raft`强制将`Leader`的日志条目覆盖到`Follower`上, 这一机制的根本前提是: **`Leader`的日志是最新和完整的**, 这一前提的实现就是接下来介绍了**选举约束**

# 2 选举约束
## 2.1 机制描述
`Raft`中选举约束的机制是:
1. 如果`Term`更小, 直接拒绝投票
2. `Candidate`的最后一条`Log`的`Term`大于本地最后一条`Log`的`Term`, 投票
3. 否则, `Candidate`的最后一条`Log`的`Term`等于本地最后一条`Log`的`Term`, 且`Candidate`的`Log数组`长度更长, 投票
4. 否则, 拒绝投票

在之前的`AppendEntries RPC`中的参数中, 包含了`Term`, 其表示`Candidate`的`Term`, 为什么不使用`Candidate`的`Term`进行比较而实用最后一条`Log`的`Term`进行比较呢? 因为使用`Candidate`的`Term`进行比较会出现很多问题, 例如孤立节点:

1. 某一时刻一个`server`网络出现了问题(称其为`S`), 其自增`currentTerm`(即记录自身的`Term`的字段)后发出选举， 经过多次选举超时后其`currentTerm`已经远大于离开集群时的`currentTerm`
2. 后来网络恢复了正常, 这时其他的服务器收到了`S`的选举请求, 这个选举请求有更新的term, 因此都同意向它投票, `S`成为了最新的`leader`
3. 由于`S`离开集群时集群其他的服务器已经提交了多个`log`, 这些提交在`S`中不存在, 而`S`信任自己的`log`, 并将自己的`log`复制到所有的`follower`上, 这将覆盖已经提交了多个`log`, 导致了错误

## 2.2 案例
此处还是举之前的那一个例子:

|节点\log索引|10|11|12|13|
|:---|---:|---:|---:|---:|
|S1|3||||
|S2|3|3|4||
|S3|3|3|5||

但我们假设这个到达这个状态(`S3`是`Leader`, `Term`为5, 且向自身追加`log`前已经发送了心跳, 即同步了`Term`)后, `S3`又故障了, 然后其马上又恢复, 此时没有`Leader`, 将触发选举超时, 假设`S1`先触发选举超时, 其广播投票的`RequestVote RPC`, 其论文[Figure2](../../images/raft-figure2.png)描述的请求参数为:

- `term`: `candidate` 的`Term`
- `candidateId`: `candidate` 的`id`
- `lastLogIndex`: `candidate` 最后一个日志项的索引
- `lastLogTerm`: `candidate` 最后一个日志项的`Term`

因此, `S1`发出的`RequestVote RPC`参数为:

```go
args := &RequestVoteArgs{
    Term:         5,
    candidateId:  1,
    lastLogIndex: 10,
    lastLogTerm:  3,
}
```

其余节点的反应为:
- `S2`和`S3`发现其`RequestVoteArgs`的`Term`为5, 进行下一步判断, 但发现`lastLogTerm`比自己的`4`和`5`更小, 因此拒绝投票

此后, `S2`发起了选举, `RequestVote RPC`参数为:

```go
args := &RequestVoteArgs{
    Term:         5,
    candidateId:  2,
    lastLogIndex: 12,
    lastLogTerm:  4,
}
```

其余节点的反应为:
- `S1`发现其`RequestVoteArgs`的`Term`为5, 进行下一步判断, 发现`lastLogTerm`为4, 比自己的3更大, 投票
- `S3`发现其`RequestVoteArgs`的`Term`为5, 进行下一步判断, 发现`lastLogTerm`为4, 比自己的5更小, 拒绝投票

`S2`收获了自身和`S1`两张选票, 满足过半的要求, 成为新的`Leader`, 并在稍后将通过心跳将自己的`Term`为4的那个`log`覆盖掉`S3`中相同位置的`log`

# 3 快速恢复
## 3.1 快速恢复的需求
在之前**日志恢复**的介绍中, 如果有`Follower`的日志不匹配, 每次`RPC`中, `Leader`会将其`nextIndex`自减1来重试, 但其在某些情况下会导致效率很低(说的就是`Lab2`的测例), 其情况为:
1. 某一时刻, 发生了网络分区, 旧的`leader`正好在数量较少的那一个分区, 且这个分区无法满足`commit`过半的要求
2. 另一个大的分区节点数量更多, 能满足投票过半和`commit`过半的要求, 因此选出了`Leader`并追加并`commit`了很多新的`log`
3. 于此同时, 旧的`leader`也在向其分区内的节点追加很多新的`log`, 只是其永远也无法`commit`
4. 某一时刻, 网络恢复正常, 旧的`Leader`被转化为`Follower`, 其需要进行新的`Leader`的日志恢复, 由于其`log数组`差异巨大, 因此将`nextIndex`自减1来重试将耗费大量的时间

因此, 在上述情况下, 需要进行**快速恢复**的优化

## 3.1 快速恢复的机制
论文中描述如下:
> If desired, the protocol can be optimized to reduce the number of rejected AppendEntries RPCs. For example, when rejecting an AppendEntries request, the follower can include the term of the conflicting entry and the first index it stores for that term. With this information, the leader can decrement nextIndex to bypass all of the conflicting entries in that term; one AppendEntries RPC will be required for each term with conflicting entries, rather than one RPC per entry. In practice, we doubt this optimization is necessary, since failures happen infrequently and it is unlikely that there will be many inconsistent entries.

论文中的描述过于简略, 教授在课堂上进行了进一步的解释, 其思想在于:**`Follower`返回更多信息给`Leader`，使其可以以`Term`为单位来回退**

具体而言, 需要在`AppendEntriesReplay`中增加下面几个字段:
- `XTerm`: `Follower`中与`Leader`冲突的`Log`对应的`Term`, 如果`Follower`在对应位置没有`Log`将其设置为-1
- `XIndex`: `Follower`中，对应`Term`为`XTerm`的第一条`Log`条目的索引
- `XLen`: 空白的`Log`槽位数, 如果`Follower`在对应位置没有`Log`，那么`XTerm`设置为-1

当`Follower`收到回复后, 按如下规则做出反应:
1. 如果`XTerm != -1`, 表示`PrevLogIndex`这个位置发生了冲突, `Follower`检查自身是否有`Term`为`XTerm`的日志项
   1. 如果有, 则将`nextIndex[i]`设置为自己`Term`为`XTerm`的最后一个日志项的下一位, 这样的情况出现在`Follower`有着更多旧`Term`的日志项(`Leader`也有这样`Term`的日志项), 这种回退会一次性覆盖掉多余的旧`Term`的日志项
   2. 如果没有, 则将`nextIndex[i]`设置为`XIndex`, 这样的情况出现在`Follower`有着`Leader`所没有的`Term`的旧日志项, 这种回退会一次性覆盖掉没有出现在`Leader`中的`Term`的日志项
2. 如果`XTerm == -1`, 表示`Follower`中的日志不存在`PrevLogIndex`处的日志项, 这样的情况出现在`Follower`的`log数组长度`更短的情况下, 此时将`nextIndex[i]`减去`XLen`

## 3.2 案例说明
如下所示为各个节点的状态, 此时`Leader`为`S3`, 其将要广播`Term`为6的`AppendEntries RPC`给`Follower`:
    
|节点\log索引|10|11|12|13|
|:---|---:|---:|---:|---:|
|S0|4||||
|S1|4|5|5||
|S2|4|4|4||
|S3|4|6|6|6|

其请求的`AppendEntriesArgs`为:
```go
args := &AppendEntriesArgs{
    Term:         6,
    LeaderId:     3,
    PrevLogIndex: 12,
    PrevLogTerm:  6,
    LeaderCommit: 1,
    Entrie:       ...,
}
```

1. 情况1: `S0`:
   - `S0`在`PrevLogIndex`位置不存在`log`, 其返回`XTerm=-1 && XLen=2`
   - `Follower`收到回复后, 将`nextIndex[S0]`减去`XLen=2`, 下次发送时`PrevLogIndex=10`. 将进行正常的追加日志
2. 情况2: `S1`:
   - `S1`在`PrevLogIndex`位置的`Term`发生了冲突, 其返回`XTerm=5 && XIndex=11`
   - `Follower`收到回复后, 发现自己没有`Term =5`的日志项, 将`nextIndex[S1]`设置为`XIndex=11`, 下次发送时`PrevLogIndex=10`. 将进行正常的追加日志并覆盖掉`Term=5`的部分
3. 情况3: `S2`:
   - `S2`在`PrevLogIndex`位置的`Term`发生了冲突, 其返回`XTerm=4 && XIndex=10`
    - `Follower`收到回复后, 发现自己也有`Term =4`的日志项, 将`nextIndex[S1]`设置为`Term=4`的最后一个`log`的下一位, 即11, 下次发送时`PrevLogIndex=10`. 将进行正常的追加日志并覆盖掉多余的`Term=4`的部分

# 4 持久化
## 4.1 持久化的内容
持久化存储的目的是为了在服务器重启时利用持久化存储的数据恢复节点上一个工作时刻的状态。并且，持久化的内容仅仅是`Raft`层, 其应用层不做要求。

论文中提到需要持久花的数据包括:
1. `votedFor`:
   `votedFor`记录了一个节点在某个`Term`内的投票记录, 因此如果不将这个数据持久化, 可能会导致如下情况:
   1. 在一个`Term`内某个节点向某个`Candidate`投票, 随后故障
   2. 故障重启后, 又收到了另一个`RequestVote RPC`, 由于其没有将`votedFor`持久化, 因此其不知道自己已经投过票, 结果是再次投票, 这将导致同一个`Term`可能出现2个`Leader`
2. `currentTerm`:
   `currentTerm`的作用也是实现一个任期内最多只有一个`Leader`, 因为如果一个几点重启后不知道现在的`Term`时多少, 其无法再进行投票时将`currentTerm`递增到正确的值, 也可能导致有多个`Leader`在同一个`Term`中出现
3. `Log`:
   这个很好理解, 需要用`Log`来恢复自身的状态

这里值得思考的是：**为什么只需要持久化`votedFor`, `currentTerm`, `Log`？**

原因是其他的数据， 包括 `commitIndex`、`lastApplied`、`nextIndex`、`matchIndex`都可以通过心跳的发送和回复逐步被重建, `Leader`会根据回复信息判断出哪些`Log`被`commit`了。

## 4.2 什么时候持久化
由于将任何数据持久化到硬盘上都是巨大的开销, 其开销远大于`RPC`, 因此需要仔细考虑什么时候将数据持久化。

如果每次修改三个需要持久化的数据: `votedFor`, `currentTerm`, `Log`时, 都进行持久化, 其持久化的开销将会很大， 很容易想到的解决方案是进行批量化操作， 例如只在回复一个`RPC`或者发送一个`RPC`时，才进行持久化操作。

# 5 快照
## 5.1 为什么需要快照？
`Log`实际上是描述了某个应用的操作, 以一个`K/V数据库`为例, `Log`就是`Put`或者`Get`, 当这个应用运行了相当长的时间后, 其积累的`Log`将变得很长, 但`K/V数据库`实际上键值对并不多, 因为`Log`包含了大量的对同一个键的赋值或取值操作。

因此， 应当设计一个阈值，例如1M， 将应用程序的状态做一个快照，然后丢弃这个快照之前的`Log`。

这里有两大关键点：
1. 快照是`Raft`要求上层的应用程序做的, 因为`Raft`本身并不理解应用程序的状态和各种命令
2. `Raft`需要选取一个`Log`作为快照的分界点, 在这个分界点要求应用程序做快照, 并删除这个分界点之前的`Log`
3. 在持久化快照的同时也持久化这个分界点之后的`Log`

引入快照后, `Raft`启动时需要检查是否有之前创建的快照, 并迫使应用程序应用这个快照。

## 5.2 快照造成的`Follower`日志缺失问题
假设有一个`Follower`的日志数组长度很短, 短于`Leader`做出快照的分界点, 那么这中间缺失的`Log`将无法通过心跳`AppendEntries RPC`发给`Follower`, 因此这个确实的`Log`将永久无法被补上。

- 解决方案1：
如果`Leader`发现有`Follower`的`Log`落后作快照的分界点，那么`Leader`就不丢弃快照之前的`Log`。

这个方案的缺陷在于如果一个`Follower`落后太多(例如关机了一周), 这个`Follower`的`Log`长度将使`Leader`无法通过快照来减少内存消耗。

- 解决方案2：
这也是`Raft`采用的方案。`Leader`可以丢弃`Follower`落后作快照的分界点的`Log`。通过一个新的`InstallSnapshot RPC`来补全丢失的`Log`, 具体来说过程如下:
  1. `Follower`通过`AppendEntries`发现自己的`Log`更短, 强制`Leader`回退自己的`Log`
  2. 回退到在某个点时，`Leader`不能再回退，因为它已经到了自己`Log`的起点, 更早的`Log`已经由于快照而被丢弃
  3. `Leader`将自己的快照发给`Follower`
  4. `Leader`稍后通过`AppendEntries`发送快照后的`Log`

