# MIT-6.824-LearningNotes
The paper and details notes after learning MIT 6.824 Distributed System.

**大部分笔记都记录在Raft论文里了。**

---

以下只是部分补充：

- **Leader Election领带人选举**：Leader宕机后选取新的Leader

- **Log Replication日志复制**

- **Safety安全性**：

  - 日志只能从领导人流向Follower
  - 只有**当前Leader任期**的日志条目才能通过计算数目来进行提交。为了提交之前任期日志条目，只能通过在当前任期新commit了一个当前任期的日志条目，才能把在此之前的日志条目一并提交

- **成员变更**：join consensus共同一致

- **日志压缩**：snapshot快照。

  - **何时发送InstallSnapshotRPC？**

    =>当Leader发送AppendEntryRPC是，发现目标Follower的nextIndex比自己第一个log的index还小时，就把自己的Snapshot发送给该Follower