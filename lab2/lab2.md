# Lab 2

## Lab 2 A
Implement Raft by adding code to raft/raft.go. In that file you'll find skeleton code, plus examples of how to send and receive RPCs.


## Lab 2 B

## Lab 2 C

## Lecture

## RAFT
1. how leader election works
    why do we need a leader?
    - the part of answer is you do not need a leader to build a system like this, it is possibile to build an agreement system by which a cluster of servers agrees, the sequence of entries in a log with having any kind of designed leader. Indeed the original pack so system which the paper refers to original paxos did not have a leader.
    - more efficient system if you have a leader. You can basically get agreement on request with one round message.

2. how leader deals with the different replicas logs particularly after failure.

3. Leader failure, it requires to elect new leader. 
    - 对于worker来说，leader超时选举的时间应该超过2次head beat的长度既，是最小值应该为两倍的heat beat的长度，这样才能尽量避免网络拥堵导致leader发送的head beat没有被worker即时收到导致在老leader存活的情况下重新选举新的leader
    - 对于worker来说，选举超时时间的最大值不应该太长，避免程序等待太久去选举新的leader
    - 对于worker来说，每次的选举超时时间应该需要每次生成一次，避免和其他worker同时选举超时时间超时，导致无法凑齐足够多的worker去选举出新的leader

4. 新的leader和旧的leader同时存在为什么不影响结果
    - 旧的leader收到了客户端发来的请求，但是旧leader无法得到半数以上的server的回应，包括自己，就不会做出提交客户端的请求，所以客户端不会认为旧的lead做出任何修改
    - 

5. 经过一连串的carsh后，log长啥样
a. 最简单的情况，就是S1死了
S1: 1
S2: 1   1
S3: 1   1
b. S1先挂掉，然后S2选举成为新的leader，然后s2没有等s3写入，只有两个server，自己回复就成，，然后s2挂掉，s1回复，然后s3被选举成新的leader，然后s3写入
S1: 3
S2: 3   3   4
S3: 3   3   5   
c. server如何处理这种情况
leader上面会保留nextindex信息,记录不同的server的log index情况,当log index
S1: 3
S2: 3   3   4
S3: 3   3   5   6
先发送
    appendEntry = 6 3
    prelogindex = 2
    prelogterm  = 5
对于s1来说, preloginindex的位置上面没有东西, 回复拒绝
对于s2来说, preloginindex的位置上面的值不一致, 回复拒绝
然后再发送
    appendEntry = 5 2, 6 3
    prelogindex = 1
    prelogterm = 3
对于s1来说, preloginindex的位置上面没有东西, 回复拒绝
对于s2来说, preloginindex的位置上面的值一致, 回复成功,并且修改该位置上面的值
S1: 3
S2: 3   3   5   6
S3: 3   3   5   6
然后再发送
    appendEntry = 3, 15 2, 6 3
    prelogindex = 0
    prelogterm = 3
对于s1来说, preloginindex的位置上面的值一致, 回复成功,并且修改该位置上面的值
S1: 3   3   5   6
S2: 3   3   5   6
S3: 3   3   5   6
log就成功同步了

6. why not longest log as leader?
    S1: 5   6   7
    S2: 5   8
    S3: 5   8

### PERSISTENT
if a server crashes and restarts, what must Raft remember?
  
  Figure 2 lists "persistent state":
    log[], currentTerm, votedFor

  a Raft server can only re-join after restart if these are intact
  
  thus it must save them to non-volatile storage:
    non-volatile = disk, SSD, battery-backed RAM, &c
    save after each change -- many points in code
    or before sending any RPC or RPC reply

- log: it is the only record for the application state
- current term: to ensure terms only increase, so each term has at most one leader
    to detect RPCs from stale leaders and candidates
- vote for: to prevent a client from voting for one candidate, then reboot, then vote for a different candidate in the same (or older!) term could lead to two leaders for the same term.

some Raft state is volatile, like commitIndex, lastApplied, next/matchIndex[], why is it OK not to save these?

persistence is often the bottleneck for performance
  a hard disk write takes 10 ms, SSD write takes 0.1 ms
  so persistence limits us to 100 to 10,000 ops/second
  (the other potential bottleneck is RPC, which takes << 1 ms on a LAN)
  lots of tricks to cope with slowness of persistence:
    batch many new log entries per disk write
    persist to battery-backed RAM, not disk

### Linearizability
we need a definition of correct for lab 3 and 4

how should client expect put and get to behave? helps us reason about how to handle complex situations correctly e.g. concurrency, replicas, failures, RPC retransmission,leader changes, optimizations

"linearizability" is the most common and intuitive definition formalizes behavior expected of a single server ("strong" consistency)
  an execution history is linearizable if one can find a total order of all operations,that matches real-time (for non-overlapping ops), and in which each read sees the value from them write preceding it in the order.
example history 1:
  |-Wx1-| |-Wx2-|
    |---Rx2---|
      |-Rx1-|
|-Wx1-| : first | mean client request, second | mean client get the reply

## Raft 算法
[raft算法详解](https://zhuanlan.zhihu.com/p/32052223)

### In Search of an Understandable Consensus Algorithm (Raft Extended Version)
https://www.jianshu.com/p/4d442a726c92
https://zhuanlan.zhihu.com/p/445546225
https://blog.csdn.net/qq_40832456/article/details/104843098

1. GFS 和HDFS
- [GFS一些总结性得内容](https://mp.weixin.qq.com/s/ut8Q7vXa5Lm0auNaN2_Emg)
- [HDFS总结](https://zhuanlan.zhihu.com/p/341711981)

2. Paxos问题
  1. 难以理解
  2. 无法通过论文搭建一个实用得实现


### The design of a practical system for fault-tolerant virtual machines (Fault-Tolerant Virtual Machines)
