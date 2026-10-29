# MIT 分布式系统实验文档索引

本目录沿用旧版 `6.824-2020` 路径。[MapReduce](lab1-mapreduce.md)、[Raft](lab2-raft.md) 和 [容错 KV](lab3-kvraft.md) 已对应 MIT 6.5840 Spring 2026 及本仓库的 `6.5840/src` 代码框架；分片 KV 文档仍是 6.824 Spring 2020 版笔记。文件名 `lab2-raft.md`、`lab3-kvraft.md` 保留旧编号，文档标题和内容采用当前课程的 Lab 3、Lab 4 编号。

## 实验路线

| 实验 | 要实现的系统 | 核心任务 | 后续依赖 |
| --- | --- | --- | --- |
| [6.5840 Lab 1：MapReduce](lab1-mapreduce.md) | 并行计算框架 | Coordinator 分配任务；Worker 执行 Map/Reduce；处理 Worker 失效 | 独立 |
| [6.5840 Lab 3：Raft](lab2-raft.md) | 一致性协议 | 3A 选举、3B 日志、3C 持久化、3D 快照 | 当前 `raft1` 代码框架 |
| [6.5840 Lab 4：容错 KV 服务](lab3-kvraft.md) | Raft 复制的键值服务 | 4A RSM、4B 带版本的 Put/Get、4C 快照 | 建立在当前 `raft1` 上 |
| [6.824 旧版 Lab 4：分片 KV 服务](lab4-shardkv.md) | 可动态重配置的分布式键值服务 | Shard Master 管理配置；Replica Group 提供服务；跨组迁移 Shard | 旧版要求，未对应当前代码框架 |

```text
MapReduce（独立）

Raft
  └── Raft KV（含快照）
        └── Sharded KV（多组 Raft KV + 配置管理 + Shard 迁移）
```

## 学习时要逐步回答的问题

1. **MapReduce：** 如何把一个大任务分派给多个 Worker，并在 Worker 失效时仍能产出正确结果？
2. **Raft：** 多个节点如何在网络延迟、丢包、分区和崩溃恢复下，对同一组日志达成一致？
3. **容错 KV：** 如何把客户端操作变成复制状态机中的有序操作，并让客户端看到线性一致的结果？
4. **分片 KV：** 如何将键空间分散到多个复制组，在配置变化时迁移数据，同时保持客户端操作正确？

## 建议推进方式

- 学习当前代码框架时，先完成 6.5840 Lab 3 的 `raft1`，再完成 Lab 4 的 `rsm1` 和 `kvraft1`；Lab 1 可单独学习。
- 每个 Part 单独运行对应测试，稳定通过后再进入下一阶段。
- Raft 及其上层服务尤其要检查并发安全和重复运行结果；网络测试会模拟延迟、丢包、断连及节点重启。
- 先对照实验说明和测试理解验收条件。实验文档只定义需要达到的行为，不替代论文和接口注释。

## 官方实验说明

- [6.5840 Lab 1：MapReduce（2026）](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)
- [6.5840 Lab 3：Raft（2026）](https://pdos.csail.mit.edu/6.5840/labs/lab-raft1.html)
- [6.5840 Lab 4：Fault-tolerant Key/Value Service（2026）](https://pdos.csail.mit.edu/6.5840/labs/lab-kvraft1.html)
- [6.824 Lab 4：Sharded Key/Value Service（2020）](http://nil.csail.mit.edu/6.824/2020/labs/lab-shard.html)
