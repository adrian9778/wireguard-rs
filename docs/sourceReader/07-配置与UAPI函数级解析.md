[上一篇](06-router模块函数级解析.md) · [总目录](README.md) · [下一篇](08-平台IO与启动并发模型.md)

# 第四层：配置与 UAPI 函数级解析

> **版本**：0.1.4
> **源码基准**：当前源码
> **范围**：`src/configuration/`（`mod.rs` / `config.rs` / `error.rs` / `uapi/mod.rs` / `uapi/set.rs` / `uapi/get.rs`）

## 1. 模块职责

控制面。把 `wg` 工具通过 Unix 套接字发来的纯文本 UAPI 协议（`set=1` / `get=1`）翻译为对 `WireGuard` 核心的调用，并隐藏底层 IO 泛型（`Configuration` trait 注释：`src/configuration/config.rs`）。

---

## 2. uapi/mod.rs —— 协议入口

### 2.1 常量

`MAX_LINE_LENGTH=256`（uapi/mod.rs 偏移：`+11`）。

### 2.2 函数：`handle`

**源码**：`src/configuration/uapi/mod.rs` 函数：`handle`（偏移：`+13 ~ +82`）
**职责**：为每个 UAPI 连接服务的顶层循环。
**参数**：`stream:&mut S`（Read+Write）、`config:&C`（实现 `Configuration`）。
**被调用**（内部闭包）：`readline`、`keypair`、`operation`。
**执行流程**：
1. `operation(stream, config)` 处理一个完整请求 偏移：`+14`。
2. 无论成功失败，写 `errno=<n>\n\n` 收尾（偏移：`+73 ~ +81`）；错误码由 `ConfigError::errno()` 给出（error.rs）。
**数据流**：文本行 → `operation` → `config.*` → 文本响应 + `errno`。

### 2.3 内部 `operation`（偏移：`+14 ~ +66`）

1. `readline` 读操作行：`get=1` → `serialize(stream, config)`；`set=1` → 循环 `readline` 逐行 `LineParser::parse_line`，遇空行 `flush` 后结束；其它 → `InvalidOperation` 偏移：`+46 ~ +65`。
2. `readline`（偏移：`+19`）：逐字节读至 `\n`，超 `MAX_LINE_LENGTH` → `LineTooLong`；EOF → `IOError`。
3. `keypair`（偏移：`+37`）：`splitn(2, '=')` 拆 `key=value`，不足两段 → `LineTooLong`。

---

## 3. uapi/set.rs —— 解析 set 事务

### 3.1 类型

**Enum：`ParserState`**（set.rs 枚举：`ParserState`）：`Peer(ParsedPeer)` / `Interface`。
**Struct：`ParsedPeer`**（set.rs 结构：`ParsedPeer`）：`public_key`、`update_only`、`allowed_ips`、`remove`、`preshared_key`、`replace_allowed_ips`、`persistent_keepalive_interval`、`protocol_version`、`endpoint`。
**Struct：`LineParser<'a, C>`**（set.rs 结构：`LineParser`）：`config:&'a C`、`state:ParserState`。

### 3.2 函数：`new` / `new_peer`

- `LineParser::new`（偏移：`+31`）：初始 `state=Interface`。
- `new_peer`（偏移：`+38`）：`<[u8;32]>::from_hex(value)` 解析公钥 → `ParserState::Peer(...)`；失败 `InvalidHexValue`。

### 3.3 函数：`parse_line`（核心）

**源码**：set.rs 函数：`parse_line`（偏移：`+55 ~ +258`）
**职责**：逐行解析 `set` 事务，累积 peer 状态，遇 `public_key` 切换或空行 `flush_peer`。
**关键分支**：
- `ParserState::Interface`：
  - `private_key`（偏移：`+111`）：hex → `StaticSecret`（全零视为 `None`），`config.set_private_key`。
  - `listen_port`（偏移：`+124`）：`config.set_listen_port`。
  - `fwmark`（偏移：`+133`）：0 视为 `None`。
  - `replace_peers`（偏移：`+143`）：对 `get_peers()` 逐个 `remove_peer`。
  - `public_key`（偏移：`+154`）：切到 `Peer` 状态。
  - `""`（偏移：`+160`）：事务结束，空操作。
- `ParserState::Peer`：
  - `public_key`（偏移：`+169`）：先 `flush_peer` 再 `new_peer`（批量多 peer）。
  - `remove`（偏移：`+176`）、`update_only`（偏移：`+182`）、`preshared_key`（偏移：`+188`）、`endpoint`（偏移：`+197`）、`persistent_keepalive_interval`（偏移：`+206`）、`replace_allowed_ips`（偏移：`+215`）、`allowed_ip`（偏移：`+222`，`ip/masklen` 拆分）、`protocol_version`（偏移：`+236`）。
  - `""`（偏移：`+248`）：`flush_peer` 收尾。

### 3.4 内部 `flush_peer`（偏移：`+64 ~ +104`）

**职责**：把一个完整 `ParsedPeer` 应用到配置。
**执行流程**：
1. `remove` → `config.remove_peer` 偏移：`+66`。
2. `!update_only` → `config.add_peer` 偏移：`+71 ~ +73`。
3. `allowed_ips` 逐个 `config.add_allowed_ip` 偏移：`+76 ~ +79`。
4. `preshared_key` → `set_preshared_key`；`persistent_keepalive_interval` → `set_persistent_keepalive_interval` 偏移：`+81 ~ +89`。
5. `protocol_version` 越界 → `UnsupportedProtocolVersion` 偏移：`+91 ~ +96`。
6. `endpoint` → `set_endpoint` 偏移：`+98 ~ +101`。
**返回值**：`Option<ConfigError>`（首个错误即终止）。

---

## 4. uapi/get.rs —— 序列化 get 响应

**函数：`serialize`**（get.rs 偏移：`+5 ~ +56`）
**职责**：把设备与所有 peer 状态写成 `key=value` 文本行。
**参数**：`writer:&mut W`、`config:&C`。
**输出**：`private_key` / `listen_port` / `fwmark`（接口级）；每 peer 输出 `public_key`/`preshared_key`/`rx_bytes`/`tx_bytes`/`persistent_keepalive_interval`/`last_handshake_time_*`/`endpoint`/`allowed_ip`（get.rs 偏移：`+17 ~ +53`）。
**设计意图**：与 `wg` 工具读取格式对齐；`ConfigError`/错误不在此产生，仅 `io::Result`。

---

## 5. config.rs —— Configuration 实现

### 5.1 类型

**Struct：`PeerState`**（config.rs 结构：`PeerState`）：peer 快照（统计/握手时间/公钥/allowed_ips/endpoint/keepalive/psk）。
**Struct：`WireGuardConfig<T,B>`**（config.rs 结构：`WireGuardConfig`）：`Arc<Mutex<Inner>>`；`Inner{ wireguard, port, bind:Option<B::Owner>, fwmark }`。
**Trait：`Configuration`**（config.rs Trait：`Configuration`）：对外配置接口（up/down/set_private_key/add_peer/.../get_peers 等）。

### 5.2 函数：`WireGuardConfig::new`

**源码**：config.rs 函数：`new`（偏移：`+47`）
**职责**：包装 `WireGuard` 为可克隆配置句柄（初始 `port=0`、`bind=None`、`fwmark=None`）。

### 5.3 函数：`start_listener`（绑定 UDP）

**源码**：config.rs 函数：`start_listener`（偏移：`+195 ~ +222`）
**职责**：按 `port` 调 `B::bind`，注入 UDP writer 与 readers，持有 `Owner`。
**执行流程**：`B::bind(cfg.port)` → `owner.set_fwmark(cfg.fwmark)` → `wireguard.set_writer(writer)` → 每个 reader `add_udp_reader` → `cfg.bind=Some(owner)` 偏移：`+201 ~ +221`。
**错误处理**：bind 失败 → `ConfigError::FailedToBind`。

### 5.4 重要 `Configuration` 方法（impl 偏移：`+224`）

| 方法 | 偏移 | 调用核心 | 说明 |
|---|---|---|---|
| `up(mtu)` | `+225` | `wireguard.up(mtu)` + `start_listener` | 设备 up 并监听 UDP |
| `down` | `+232` | `wireguard.down()` + `bind=None` | 设备 down |
| `set_private_key` | `+243` | `wireguard.set_key(sk)` | 替换私钥（触发 `handshake::Device::set_sk`） |
| `get_private_key` | `+248` | `wireguard.get_sk()` | 取回私钥 |
| `get_protocol_version` | `+252` | 常量 `1` | 协议版本 |
| `get_listen_port` | `+256` | `bind.get_port()` | 当前端口 |
| `set_listen_port` | `+262` | 更新 `port` + 必要时 `start_listener` | 重绑端口 |
| `set_fwmark` | `+281` | `bind.set_fwmark(mark)` | 设 fwmark |
| `replace_peers` | `+295` | `wireguard.clear_peers()` | 清空所有 peer |
| `remove_peer` | `+299` | `wireguard.remove_peer` | 删 peer |
| `add_peer` | `+303` | `wireguard.add_peer` | 加 peer |
| `set_preshared_key` | `+307` | `wireguard.set_psk` | 设 PSK |
| `set_endpoint` | `+311` | `peers.read().get(peer).set_endpoint(B::Endpoint::from_address)` | 设端点 |
| `set_persistent_keepalive_interval` | `+317` | `peer.opaque().set_persistent_keepalive_interval(secs)` | 设保活 |
| `replace_allowed_ips` | `+323` | `peer.remove_allowed_ips()` | 清空子网 |
| `add_allowed_ip` | `+329` | `peer.add_allowed_ip(ip, masklen)` | 加 cryptokey 子网 |
| `get_peers` | `+354` | 遍历 `peers` 构造 `PeerState` | 取全部 peer 快照 |

**并发**：所有方法经 `self.lock()`（互斥锁）串行化；`get_peers` 在锁内读 `walltime_last_handshake` 等并转换 `(secs, nsecs)`。

---

## 6. error.rs —— 配置错误

**Enum：`ConfigError`**（error.rs 枚举：`ConfigError`）：`FailedToBind`/`InvalidHexValue`/`InvalidPortNumber`/`InvalidFwmark`/`InvalidKey`/`InvalidSocketAddr`/`InvalidKeepaliveInterval`/`InvalidAllowedIp`/`InvalidOperation`/`LineTooLong`/`IOError`/`UnsupportedValue`/`UnsupportedProtocolVersion`。
**函数：`errno`**（偏移：`+42`）：Unix 下把错误映射到 `libc` errno（如 `FailedToBind→EPERM`、`Invalid*→EINVAL`、`LineTooLong→EPROTO`、`IOError→EIO`），供 UAPI 回写 `errno=`。

---

## 7. set 事务数据流

```text
UAPI 连接
  → uapi::handle → operation
  → readline("set=1")
  → loop readline:
       LineParser::parse_line(key, value)
         ├─ Interface 态：set_private_key / listen_port / fwmark / replace_peers / public_key(切 Peer 态)
         └─ Peer 态：累积 ParsedPeer（public_key 切换时 flush_peer）
       "" → flush_peer（末个 peer）
  → 每行 flush_peer → Configuration::add_peer / add_allowed_ip / set_endpoint / set_preshared_key ...
  → WireGuard::add_peer → handshake::Device::add + router::PeerHandle::add_allowed_ip
  → 写 "errno=0\n\n"
```

> 启动流程、platform IO 与 worker 并发见 `08`。

[上一篇](06-router模块函数级解析.md) · [总目录](README.md) · [下一篇](08-平台IO与启动并发模型.md)
