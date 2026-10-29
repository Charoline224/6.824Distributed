# Lab 4：基于 Raft 的容错键值服务（对应本仓库的 6.5840 代码框架）

> 本文件沿用旧版 `lab3-kvraft.md` 文件名，便于已有链接继续使用。仓库中的 `6.5840/src/kvraft1` 对应 MIT 6.5840 Spring 2026 的 **Lab 4：Fault-tolerant Key/Value Service**，分为 **4A、4B、4C**。它依赖 [Lab 3：Raft](lab2-raft.md) 的实现。

## 目标与代码位置

通过 Raft 让多个 KV 副本按同一顺序执行操作。多数节点仍可通信时服务应继续工作；已完成的 `Put/Get` 必须满足线性一致性。当前接口使用 **带版本号的 `Put` 和 `Get`**，没有旧版实验中的 `Append`。

| 用途 | 本仓库路径 |
| --- | --- |
| 通用复制状态机（4A，4C） | `6.5840/src/kvraft1/rsm/rsm.go` |
| KV 客户端（4B） | `6.5840/src/kvraft1/client.go` |
| KV 服务端及快照（4B，4C） | `6.5840/src/kvraft1/server.go` |
| 请求、响应及错误类型 | `6.5840/src/kvsrv1/rpc/rpc.go` |
| RSM 测试 | `6.5840/src/kvraft1/rsm/rsm_test.go` |
| KV 测试 | `6.5840/src/kvraft1/kvraft_test.go` |
| 测试服务入口 | `6.5840/src/main/rsm1d.go`、`6.5840/src/main/kvraft1d.go` |

`rsm` 位于服务与 Raft 之间：服务调用 `rsm.Submit()` 提交操作；Raft 提交后通过 `applyCh` 通知各副本；`rsm` 调用服务的 `DoOp()`，再把执行结果交还给对应的 `Submit()`。`StateMachine` 接口还提供 `Snapshot()` 和 `Restore()`，供 4C 使用。

## Part 4A：通用复制状态机（RSM）

- 在 `rsm/rsm.go` 实现 `Submit()`，调用 `raft.Start()` 提交带唯一标识的 `Op`，并等待该操作提交、执行后的结果。
- 实现读取 `applyCh` 的后台处理：按提交顺序调用 `StateMachine.DoOp()`，将结果送回对应的等待者；所有副本都要执行提交的操作。
- 非 Leader 应返回 `rpc.ErrWrongLeader`。若提交后失去领导权，或同一日志索引最终提交了别的操作，也要识别原操作未成功，避免错误地返回结果。
- 测试覆盖基本复制、并发提交、Leader 故障或分区、重启重放及关闭行为。

## Part 4B：不使用快照的 KV 服务

- 在 `client.go` 实现 Clerk 的 `Get`、`Put`：找不到 Leader、RPC 失败或收到错误 Leader 响应时，尝试其他服务器并重试。
- 在 `server.go` 实现 RPC 处理函数和 `DoOp()`；`Put` 与 `Get` 都经由 `rsm.Submit()`，使读取也能避免落后少数派返回旧值。无需实现 Raft 论文第 8 节的只读优化。
- `Get` 返回键的值和版本；键不存在时返回 `ErrNoKey`。`Put(key, value, version)` 仅在请求版本与当前版本匹配时更新；新键以版本 0 创建，版本不匹配返回 `ErrVersion`。
- 已提交的 `Put` 最多生效一次。首次 RPC 明确收到 `ErrVersion` 时，Clerk 返回 `ErrVersion`；重试时收到 `ErrVersion`，此前的结果可能丢失，Clerk 应返回 `ErrMaybe`。服务器错误与网络失败不能让已完成的操作重复生效。
- 测试覆盖单客户端、并发客户端、不可靠网络、分区、多数派进度、重启及线性一致性。

## Part 4C：带快照的 KV 服务

- 在 `rsm` 中根据 `maxraftstate` 与 `rf.PersistBytes()` 判断 Raft 持久状态是否接近上限；达到阈值时调用服务的 `Snapshot()`，再交给 Raft 的 `Snapshot(index, data)`。`maxraftstate == -1` 时无需创建快照。
- 在 `server.go` 实现 `Snapshot()`、`Restore()`，保存并恢复 KV 数据及维持版本语义所需的状态。重启时，`rsm` 从 `persister.ReadSnapshot()` 取得已有快照并恢复服务状态。
- `rsm` 还要处理 `applyCh` 中的快照消息，让落后副本追上，并保证快照与后续日志的应用顺序正确。
- 测试覆盖日志大小限制、快照大小、落后副本安装快照、重启恢复、不可靠网络与并发分区。完成 4C 后仍须通过 4A、4B 和 Raft 的 3A–3D 测试。

## 运行与自查

在 `6.5840/src` 目录运行；Makefile 默认启用 `-race`：

```sh
make RUN="-run 4A" rsm1
make RUN="-run 4B" kvraft1
make RUN="-run 4C" rsm1
make RUN="-run 4C" kvraft1
make rsm1
make kvraft1
make raft1
```

- [ ] `Submit()` 只把对应已执行操作的结果交给等待的请求。
- [ ] `Put/Get` 的版本、错误和重试语义正确；少数派不返回过期的成功结果。
- [ ] 重启与快照安装后，所有副本仍按顺序执行，且持久 Raft 状态受控。
- [ ] 4A–4C 与 Raft 3A–3D 测试稳定通过，无数据竞争。

## 官方说明

[MIT 6.5840 Spring 2026 Lab 4: Fault-tolerant Key/Value Service](https://pdos.csail.mit.edu/6.5840/labs/lab-kvraft1.html)
