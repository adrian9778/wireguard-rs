[总目录](README.md) · [下一篇](01-简单框架-系统骨架.md)

# 文档规范与源码定位约定

> **版本**：wireguard-rs 0.1.4（Cargo.toml 声明）
> **源码基准**：当前仓库源码（edition = "2018"）

本库所有文档遵循同一套源码定位规则，禁止用绝对行号作为主要依据。

## 1. 定位优先级

1. **函数内部**：`文件路径 + 函数名 + 函数内相对偏移（+0 = 函数定义行）`
   - 例：`src/wireguard/handshake/device.rs` 函数：`process` 偏移：`+330 ~ +385`
2. **非函数代码**：稳定符号
   - `Struct：WireGuard`（文件：`src/wireguard/wireguard.rs`）
   - `Trait：Tun`（文件：`src/platform/tun.rs`）
   - `Const：REKEY_TIMEOUT`（文件：`src/wireguard/constants.rs`）
   - `Module：handshake`（文件：`src/wireguard/handshake/mod.rs`）
3. **调用关系**：同时标注调用方与被调用方
   - `src/wireguard/workers.rs` 函数：`handshake_worker` 调用偏移：`+183`
     ↓
     `src/wireguard/handshake/device.rs` 函数：`process`

## 2. 四层结构

```text
┌─────────────────────────────────────────┐
│ 第四层 源码补齐：模块→类型→impl→函数→内部逻辑 │  (05~10)
├─────────────────────────────────────────┤
│ 第三层 详细逐步：主链路逐跳拆解            │  (03)
├─────────────────────────────────────────┤
│ 第二层 简单例子：一个真实最小可运行成功路径  │  (02)
├─────────────────────────────────────────┤
│ 第一层 简单框架：系统骨架、模块、主数据流    │  (01)
└─────────────────────────────────────────┘
```

- **第一层 简单框架**：系统骨架、模块职责、依赖、主数据流。不展开函数。
- **第二层 简单例子**：一个真实、最小、可运行的成功场景，从入口到结果的全路径。
- **第三层 详细逐步**：基于第二层逐跳拆解（谁调谁、参数如何传、状态如何变、错误如何传播）。
- **第四层 源码补齐**：模块 → 类型 → impl → 函数/方法 → 函数内部关键逻辑，达到函数级。

## 3. 内容红线

- 禁止空泛描述、伪造函数、臆测不存在的逻辑；无法从源码证实的结论显式写「当前源码无法确定」。
- 禁止「同上 / 略 / 以此类推」替代核心内容。
- 每篇至少含 1 个 ASCII（流程）或 Mermaid（关系/时序）图。
- 每篇带「上一篇 / 总目录 / 下一篇」导航。
- 采用紧凑 Markdown：删除不必要空行，标题与正文只保留必要空行。

## 4. 术语

- **WireGuard 设备**：一个虚拟网络接口实例（`WireGuard<T, B>`）。
- **Peer**：对端。`PeerInner`（握手侧状态）与 `router::PeerHandle`（路由侧状态）一一对应。
- **KeyPair**：一次握手协商出的收发密钥对（`src/wireguard/types.rs` Struct：`KeyPair`）。
- **cryptokey route**：把目标 IP 子网映射到某个 peer 的路由表（`router::RoutingTable`）。
- **UAPI**：Userspace API，通过 Unix 域套接字与 `wg` 工具通信的纯文本配置协议。
- **Platform 抽象**：`Tun` / `UDP` / `UAPI` 三个 trait，使核心逻辑与具体 IO 解耦（`src/platform/`）。

## 5. 依赖标注

第三方 crate 注明名称与用途（版本见 Cargo.toml）：
- `x25519-dalek`（X25519 密钥交换）、`chacha20poly1305`（AEAD）、`blake2`（哈希/MAC）、`ring`（传输层 AEAD）、`hmac`（HMAC）、`zerocopy`（零拷贝字节布局）、`hjul`（定时器轮）、`dashmap`（并发 map）、`spin` / `parking_lot`（锁）、`crossbeam-channel`（并行队列）、`treebitmap`（路由表）、`subtle`（常量时间比较）、`clear_on_drop`（敏感数据清零）。

[总目录](README.md) · [下一篇](01-简单框架-系统骨架.md)
