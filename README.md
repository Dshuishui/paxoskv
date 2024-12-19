# paxoskv: a Naive and Basic paxos kv storage

![naive](https://github.com/openacid/paxoskv/workflows/test/badge.svg?branch=naive)
[![Build Status](https://travis-ci.com/openacid/paxoskv.svg?branch=naive)](https://travis-ci.com/openacid/paxoskv)

这个 repo 仅是用于学习的实例代码，来自 [XP 哥](https://github.com/openacid/paxoskv).

这是一个基于paxos, 只有200行代码的kv存储系统的简单实现, 以最简洁的形式展示paxos如何运行.
[200行代码实现基于paxos的kv存储][] 是对本代码的原始讲解.

## Paxos 核心实现要点
### 基本原则
1. **Proposal处理**
  - Phase 1 中发现任何已接受值时，Phase 2 必须提议该值.
  - 不能仅通过提高票号来修改已接受的值.

2. **读操作的特殊性**
  - 为什么可能需要执行完整的 Paxos 过程：
    - Phase 1 中看到的值可能只在少数 Acceptor 中写入就中断.
    - 必须通过 Phase 2 确保值已被多数派确认.
    - 防止并发写操作破坏一致性.
  - 读操作的两种情况：
    - 发现已投票值：执行 Phase 2 来"固化"该值.
    - 未发现任何值：直接返回 nil.

3. **票号(BallotNum)机制**
  - 比较规则：
    - 优先比较 N 值大小.
    - N 相同时比较 ProposerId.
  - 保证：
    - 提案的顺序性.
    - 并发冲突的处理.
    - 系统的活性.

## 系统实现

### 基本特性
- 写入操作通过一次2轮的 paxos 实现.
- 读取操作也通过一次1轮或2轮的 paxos 实现.
- 每个 key 支持多版本更新，每个版本对应独立的 paxos 实例.
- 仅支持指定 key-version 的写入.
- 不支持自动版本管理.
- 无 WAL 和 compaction 机制，直接把paxos instance对应到key的每个版本上.

### 目录结构

- `proto/paxoskv.proto`: 定义paxos相关的数据结构.

- `paxoskv/`:

    - `impl.go`: 206行代码实现的paxos协议:
        - 实现paxos Acceptor的`Prepare()`和`Accept()`这两个request handler;
        - 实现Proposer的功能: 执行`Phase1()`和`Phase2()`;
        - 以及完整运行一次paxos的`RunPaxos()`方法;
        - 实现一个kv纯内存的存储, 每个key有多个version, 每个version对应一个paxos instance;
        - 以及启动n个Acceptor的grpc服务函数.

    - `paxos_slides_case_test.go`: 按照 [可靠分布式系统-paxos的直观解释][] 给出的两个例子([slide-32][]和[slide-33][]), 调用paxos接口来模拟这2个场景中的paxos运行.

    - `example_set_get_test.go`: 使用paxos提供的接口实现指定key和ver的写入和读取.

## 使用

Requirements: `go >= 1.14`.

跑测试: `go test ./...`.

重新build proto文件(如果宁想要修改下玩玩的话): `make gen`.

数据结构使用protobuf 定义; RPC使用grpc实现.

如想了解最新的go grpc的环境部署,请看[go-grpc文档](https://grpc.io/docs/languages/go/quickstart/)

## 名词对照

在paxos相关的paper, [可靠分布式系统-paxos的直观解释][],
以及这个repo中代码涉及到的各种名词, 下面列出的都是等价的:

```
rnd == bal == BallotNum ~= Ballot
quorum == majority == 多数派
voted value == accepted value // by an acceptor
```


[issue]:                          https://github.com/openacid/paxoskv/issues/new/choose
[可靠分布式系统-paxos的直观解释]: https://blog.openacid.com/algo/paxos/
[200行代码实现基于paxos的kv存储]: https://blog.openacid.com/algo/paxoskv/
[slide-32]:                       https://blog.openacid.com/algo/paxos/#slide-32
[slide-33]:                       https://blog.openacid.com/algo/paxos/#slide-33
