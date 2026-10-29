# Lab 1：MapReduce（MIT 6.5840 Spring 2026）

> 本页依据[你指定的 MIT 官方 Lab 1 页面](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)整理。该链接目前显示的是 **Spring 2026 的 6.5840**，因此下列文件名、命令和测试要求均以该版本为准；本目录名称 `6.824-2020` 不代表本页采用 2020 版要求。

## 实验目标

实现一个分布式 MapReduce 系统：一个 Coordinator 通过 RPC 向一个或多个 Worker 分配任务；Worker 调用应用提供的 Map、Reduce 函数并负责读写文件。所有进程在一台机器上运行，共享文件系统。Worker 应持续请求任务；Coordinator 需要在任务分配出去 **10 秒**仍未完成时，将任务重新分配给其他 Worker。

## 官方提供的代码

- `src/main/mrsequential.go`：单进程串行实现，可参考其中读取输入、排序中间键值对和写出结果的代码。
- `src/mrapps/wc.go` 与 `src/mrapps/indexer.go`：Word Count 和文本索引应用，提供应用侧的 Map、Reduce 函数；运行时通过 Go 插件（`.so`）加载。
- `src/main/mrcoordinator.go` 与 `src/main/mrworker.go`：Coordinator 和 Worker 的启动程序，不应修改。
- `src/mr/coordinator.go`、`src/mr/worker.go`、`src/mr/rpc.go`：供你补全的起始代码；`worker.go` 中的 `ihash(key)` 可用于选择 Reduce 分区。
- `src/mr/mr_test.go`：官方验收测试，连同所需输入文件一起提供。

## 需要实现的行为

1. 每个输入文件对应一个 Map 任务。Worker 向 Coordinator 请求任务、读取文件、调用应用的 Map 函数、写出中间结果，然后继续请求任务。
2. Map 输出按 `ihash(key)` 分到 `nReduce` 个分区；`nReduce` 由 `main/mrcoordinator.go` 传给 `MakeCoordinator()`。**每个 Map 任务都应创建 `nReduce` 个中间文件**，供相应 Reduce 任务读取。中间文件保存在当前目录；`mr-X-Y` 是官方建议的命名方式，其中 `X` 是 Map 任务编号、`Y` 是 Reduce 任务编号。
3. Reduce 任务读取其分区中来自所有 Map 任务的键值对，按键组织数据，调用应用的 Reduce 函数。Reduce 应等待 Map 阶段全部完成后再执行。
4. 编号为 `X` 的 Reduce 任务必须写出 `mr-out-X`。每个 Reduce 结果占一行，使用 Go 的 `"%v %v"` 格式输出键和值；可参照 `mrsequential.go` 中标注正确格式的代码。
5. Coordinator 应处理多个并发 RPC，保护共享状态。任务超时 10 秒后可重新分配；即使 Worker 在执行中崩溃，整个作业也要能完成。不要在 Worker 正常执行时额外派发备份任务。
6. `mr/coordinator.go` 必须提供 `Done()` 方法，在整个作业完成时返回 `true`，以便启动程序退出。作业结束后 Worker 也应退出；可以在无法联系已退出的 Coordinator 时结束，或使用退出任务等协议。
7. Worker 写文件时应避免让其他进程读到不完整结果。官方建议先写临时文件，写完后用 `os.Rename` 原子替换目标文件。

RPC 传输的结构体字段以及其嵌套结构体字段需要以大写字母开头。调用 `call()` 时，传入的 reply 结构体应保持零值。

## 修改范围

正式实现放在 `src/mr/worker.go`、`src/mr/coordinator.go`、`src/mr/rpc.go`。可以临时修改其他文件来调试，但提交的实现必须在官方原版启动程序和测试代码下工作。官方也允许在新文件中添加自己的函数。

## 运行与验收

在官方项目的 `src/main` 目录中编译应用插件：

```sh
go build -buildmode=plugin ../mrapps/wc.go
```

然后分别启动 Coordinator 和一个或多个 Worker。`sock123` 是双方使用的 RPC socket；`pg-*.txt` 是 Map 输入文件：

```sh
go run mrcoordinator.go sock123 pg-*.txt
go run mrworker.go wc.so sock123
```

可以用串行程序的结果对照 `mr-out-*`。修改 `mr/` 中的代码后，手动运行时可能需要重新编译插件。在官方项目的 `src` 目录运行完整测试，`make mr` 会自动编译插件：

```sh
make mr
```

单独运行匹配 `Wc` 的测试可使用 `make RUN="-run Wc" mr`。官方测试覆盖 Word Count、Indexer、Map 并行、Reduce 并行、任务数量、提前退出以及 Worker 崩溃恢复。验收结果应与串行版本一致，并能在并行与故障测试中正确完成。

## 官方建议与选做内容

Worker 无任务时可以间隔一段时间再次请求；Coordinator 的 RPC handler 也可等待有任务可分配。中间键值对可用 `encoding/json` 写入和读取。共享文件系统是本实验的前提，无须实现分布式文件系统。

官方另列出不计分的挑战：实现自己的 MapReduce 应用；或让 Coordinator 和 Worker 分布在不同机器上运行，并相应改用 TCP/IP RPC 与共享存储。

## 官方说明

[MIT 6.5840 Spring 2026 Lab 1: MapReduce](https://pdos.csail.mit.edu/6.824/labs/lab-mr.html)
