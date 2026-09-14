[上一篇](05-handshake模块函数级解析.md) · [总目录](README.md) · [下一篇](07-配置与UAPI函数级解析.md)

# 第四层：router 模块函数级解析

> **版本**：0.1.4
> **源码基准**：当前源码
> **范围**：`src/wireguard/router/` 下 `device.rs` / `peer.rs` / `receive.rs` / `send.rs` / `route.rs` / `worker.rs` / `queue.rs` / `anti_replay.rs` / `ip.rs` / `messages.rs`

## 1. 模块职责

「cryptokey 路由器 / 报文保护器」。负责传输报文（type=4）的加解密、按目标 IP 的 cryptokey 路由、按 receiver id 的解密状态定位、抗重放、以及顺序/并行两段式作业调度（router/mod.rs 头注释）。本模块消费握手产出的 `KeyPair`。

---

## 2. device.rs —— 路由设备

### 2.1 类型

**Struct：`DeviceInner<E,C,T,B>`**（device.rs 结构：`DeviceInner`）：
- `inbound: T`（TUN 写入器）、`outbound: RwLock<(bool, Option<B>)>`（启用标志 + UDP 写入器）。
- `recv: RwLock<HashMap<u32, Arc<DecryptionState>>>` —— receiver id → 解密状态（入站定位）。
- `table: RoutingTable<Peer>` —— cryptokey 路由（出站定位）。
- `work: ParallelQueue<JobUnion>` —— worker 任务队列。

**Struct：`DecryptionState`**（device.rs 结构：`DecryptionState`）：`keypair:Arc<KeyPair>`、`confirmed:AtomicBool`、`protector:Mutex<AntiReplay>`、`peer:Peer`。
**Struct：`EncryptionState`**（device.rs 结构：`EncryptionState`）：`keypair:Arc<KeyPair>`、`nonce:u64`。
**Struct：`DeviceHandle`**（device.rs 结构：`DeviceHandle`）：持有 `Device` + worker 线程句柄；`Drop` 时关闭队列并 join（device.rs 偏移：`+87`）。

### 2.2 函数：`DeviceHandle::new`

**源码**：device.rs 函数：`new`（偏移：`+106 ~ +135`）
**职责**：构造路由设备并启动 `num_workers` 个 worker 线程。
**参数**：`num_workers:usize`、`tun:T`。
**被调用**：`ParallelQueue::new`、`thread::spawn(worker)`。
**状态变化**：`work` 通道容量 `PARALLEL_QUEUE_SIZE=4*MAX_QUEUED_PACKETS`（router/constants.rs Const）。
**并发**：每个 worker `loop { recv }`，见 `worker` 节。

### 2.3 函数：`send`（出站入口）

**源码**：device.rs 函数：`send`（偏移：`+181 ~ +201`）
**职责**：把一个明文 IP 包做 cryptokey 路由后交给对应 peer 加密发送。
**参数**：`msg:Vec<u8>`（已预留 `SIZE_MESSAGE_PREFIX` 头部空间）。
**前置条件**：`msg.len() > SIZE_MESSAGE_PREFIX`（debug_assert）。
**执行流程**：
1. `packet = &msg[SIZE_MESSAGE_PREFIX..]`（跳过预留头）偏移：`+189`。
2. `table.get_route(packet)` 按目标 IP 找 peer；无路由 → `Err(NoCryptoKeyRoute)` 偏移：`+192 ~ +196`。
3. `peer.send(msg, true)` 入站加密（stage=true，无密钥时缓存）偏移：`+199`。
**错误传播**：`NoCryptoKeyRoute` 上抛。
**设计意图**：`msg` 预留头部使 `SendJob` 可就地构造 `TransportHeader`，避免二次分配。

### 2.4 函数：`recv`（入站入口）

**源码**：device.rs 函数：`recv`（偏移：`+211 ~ +250`）
**职责**：处理一条入站传输报文。
**参数**：`src:E`、`msg:Vec<u8>`。
**执行流程**：
1. `LayoutVerified::new_from_prefix` 解析 `TransportHeader`；失败 → `MalformedTransportMessage` 偏移：`+215 ~ +220`。
2. `recv.read().get(f_receiver)` 定位 `DecryptionState`；缺失 → `UnknownReceiverId` 偏移：`+236 ~ +239`。
3. `ReceiveJob::new(msg, dec.clone(), src)` 偏移：`+242`。
4. `dec.peer.inbound.push(job)`（顺序入队，满则丢）→ `work.send(JobUnion::Inbound(job))` 偏移：`+246 ~ +248`。
**数据流**：UDP 密文 → `TransportHeader`（读 receiver/counter）→ `ReceiveJob` → worker 解密 → TUN 写。

### 2.5 函数：`send_raw`（DeviceHandle 版）

**源码**：device.rs 函数：`send_raw`（偏移：`+137 ~ +145`）
**职责**：经当前 outbound writer 把裸报文发往 `dst` 端点。
**关键分支**：`outbound.0==false`（设备 down）则空操作；`writer` 为 `None` 则空操作（未 bind）。
**返回值**：`Result<(), B::Error>`。
**调用方**：`handshake_worker`（回送握手响应）、`Peer::send_raw`。

### 2.6 `down` / `up` / `clear_sending_keys` / `set_outbound_writer`

- `down`（偏移：`+150`）：`outbound.write().0=false`，禁止出站。
- `up`（偏移：`+156`）：置 `true`，允许出站。
- `clear_sending_keys`（偏移：`+162`）：当前源码为 `log::debug!` + TODO（未实现按 peer 清除发送密钥）。
- `set_outbound_writer`（偏移：`+253`）：`outbound.write().1 = Some(new)`，注入 UDP writer。

---

## 3. peer.rs —— 路由侧 peer

### 3.1 类型

**Struct：`KeyWheel`**（peer.rs 结构：`KeyWheel`）：`next/ current/ previous: Option<Arc<KeyPair>>` + `retired:Vec<u32>`（三槽密钥轮）。
**Struct：`PeerInner`**：`device`、`opaque:C::Opaque`、`outbound/inbound: Queue<_>`、`staged_packets`、`keys:Mutex<KeyWheel>`、`enc_key:Mutex<Option<EncryptionState>>`、`endpoint`。
**Struct：`PeerHandle`**：`peer:Peer`；`Drop` 时从 `table` 与 `recv` 移除并清零密钥（peer.rs 偏移：`+144`）。

### 3.2 函数：`Peer::send`（出站加密调度）

**源码**：peer.rs 函数：`send`（偏移：`+252 ~ +298`）
**参数**：`msg:Vec<u8>`、`stage:bool`。
**执行流程**：
1. 加锁 `enc_key`：
   - `None` 且无密钥 → 若 `stage` 缓存到 `staged_packets`，置 `need_key=true` 偏移：`+257 ~ +263`。
   - `nonce >= REJECT_AFTER_MESSAGES-1` → 密钥过期，`enc_key=None`，缓存，`need_key=true` 偏移：`+266 ~ +272`。
   - 否则构造 `SendJob::new(msg, nonce, keypair, peer)`，入 `outbound` 顺序队成功则 `nonce+=1` 偏移：`+275 ~ +279`。
2. `need_key` → `C::need_key(&opaque)` 触发握手 偏移：`+288 ~ +292`。
3. `Some(job)` → `device.work.send(JobUnion::Outbound(job))` 偏移：`+294 ~ +297`。
**并发**：`enc_key` 短锁保护 nonce 分配；实际加密在 worker 并行段（见 send.rs）。

### 3.3 函数：`add_keypair`（第二层主链路落地）

**源码**：peer.rs 函数：`add_keypair`（偏移：`+436 ~ +498`）
**参数**：`new:KeyPair` → 返回 `Vec<u32>`（待释放 receiver id）。
**关键分支**：
- `new.initiator==true`：置 `enc_key=EncryptionState::new(new)`，`previous=current`、`current=Some(new)`（发起方立即用于加密）偏移：`+446 ~ +452`。
- `new.initiator==false`（第二层场景）：`previous=next`、`next=Some(new)`（响应方待确认）偏移：`+453 ~ +457`。
**外部依赖**：写 `device.recv`：`recv.insert(new.recv.id, DecryptionState::new(peer,&new))`；purge 旧 `previous.id` 并加入 `release` 偏移：`+460 ~ +476`。
**调度确认**：若 `initiator` → `send_staged()` 或 `send_keepalive()` 触发对端确认（偏移：`+481 ~ +491`）。
**状态变化**：`KeyWheel` 旋转；`recv` 映射新增解密状态；`retired` 累积旧 id。

### 3.4 函数：`confirm_key`

**源码**：peer.rs 函数：`confirm_key`（偏移：`+316 ~ +349`）
**职责**：首个传输报文确认密钥后，把 `keys.next` 转正为 `current` 并启动堆积报文发送。
**执行流程**：
1. `keys.next` 必须 `Arc::ptr_eq` 传入 `keypair`，否则 return 偏移：`+321 ~ +329`。
2. `ekey=EncryptionState::new(next)`；swap `next→current→previous` 偏移：`+332 ~ +338`。
3. `C::key_confirmed(&opaque)` 回调（timers）偏移：`+341`。
4. `*enc_key = ekey`；`send_staged()` 发送堆积包 偏移：`+344 ~ +348`。

### 3.5 `send_raw` / `send_keepalive` / `add_allowed_ip` / `set_endpoint`

- `PeerInner::send_raw`（偏移：`+225`）：有 `endpoint` 且 `outbound` 启用则 `writer.write(msg, endpoint)`，否则 `NoEndpoint`。
- `send_keepalive`（偏移：`+500`）：`peer.send(vec![0; SIZE_MESSAGE_PREFIX], false)`（空负载确认包）。
- `add_allowed_ip`（偏移：`+519`）：`table.insert(ip, masklen, peer)`。
- `set_endpoint`（偏移：`+363`）：写 `endpoint` 锁。
- `Drop for PeerHandle`（偏移：`+144`）：`table.remove(peer)`、`recv` 移除 `next/current/previous` 的 id、清零 `keys`/`enc_key`/`endpoint`。

---

## 4. send.rs —— 出站加密作业

### 4.1 类型

**Struct：`SendJob<E,C,T,B>`**（send.rs 结构：`SendJob`）：包裹 `Inner{ ready:AtomicBool, buffer:Mutex<Vec<u8>>, counter:u64, keypair, peer }`。

### 4.2 函数：`parallel_work`（并行加密）

**源码**：send.rs 函数：`parallel_work`（偏移：`+59 ~ +109`）
**职责**：在 worker 线程并行段对报文体做 ChaCha20Poly1305 加密。
**执行流程**：
1. `msg.extend(0; SIZE_TAG)` 为 tag 留位 偏移：`+72`。
2. `LayoutVerified::new_from_prefix` 取 `TransportHeader` + 体 偏移：`+75`。
3. 填头：`f_type=TYPE_TRANSPORT`、`f_receiver=keypair.send.id`、`f_counter=counter` 偏移：`+84 ~ +86`。
4. nonce = counter 拼到 12 字节（高 4 字节为 0）偏移：`+89 ~ +92`。
5. `LessSafeKey::new(UnboundKey::new(CHACHA20_POLY1305, send.key))` → `seal_in_place_separate_tag` 加密体 偏移：`+96 ~ +101`。
6. 追加 tag，`ready.store(true)` 偏移：`+103 ~ +108`。
**外部依赖**：`ring::aead`（传输层 AEAD，与控制面的 `chacha20poly1305` crate 不同）。
**错误处理**：`unwrap()`（密钥/nonce 已前置校验，失败即 panic，当前源码未做优雅处理）。

### 4.3 函数：`sequential_work`（顺序写出）

**源码**：send.rs 函数：`sequential_work`（偏移：`+119 ~ +135`）
**职责**：在 peer 的 `outbound` 顺序队列消费时执行（保证同 peer 顺序）。
**执行流程**：`peer.send_raw(&msg)` 写 UDP；`C::send(&opaque, size, xmit, keypair, counter)` 回调（更新统计/计时器）偏移：`+131 ~ +134`。
**设计意图**：加密可并行（不同包独立），但写出与计数器推进需按 peer 顺序——故拆两段：并行 `parallel_work` + 顺序队列 `consume`（`worker.rs`）。

---

## 5. receive.rs —— 入站解密作业

### 5.1 类型

**Struct：`ReceiveJob`**（receive.rs 结构：`ReceiveJob`）：`Inner{ ready:AtomicBool, buffer:Mutex<(Option<E>, Vec<u8>)>, state:Arc<DecryptionState> }`。

### 5.2 函数：`parallel_work`（并行解密）

**源码**：receive.rs 函数：`parallel_work`（偏移：`+66 ~ +124`）
**职责**：在 worker 并行段解密 + cryptokey 路由校验。
**执行流程**：
1. `LayoutVerified::new_from_prefix` 取 `TransportHeader` + 体 偏移：`+84`。
2. nonce 由 `f_counter` 构造（高 4 字节 0）偏移：`+91 ~ +94`。
3. `key.open_in_place(nonce, Aad::empty(), packet)` 原地验证解密；失败返回 false 偏移：`+101 ~ +104`。
4. `f_counter >= REJECT_AFTER_MESSAGES` → false（拒绝过旧 counter）偏移：`+107`。
5. `packet.len()==SIZE_TAG || peer.device.table.check_route(&peer, &packet)`（空包=keepalive 免校验；否则源 IP 必须路由到该 peer，防伪装）偏移：`+112`。
6. 失败则 `msg.truncate(0)` 防误用未认证数据；`ready=true` 偏移：`+117 ~ +123`。
**关键**：抗重放不能在此并行段做（会因调度乱序丢包），留到顺序段。

### 5.3 函数：`sequential_work`（顺序抗重放 + 写 TUN）

**源码**：receive.rs 函数：`sequential_work`（偏移：`+134 ~ +184`）
**执行流程**：
1. 重解析 header 取 counter 偏移：`+148`。
2. `protector.lock().update(counter)` 抗重放；失败 return 偏移：`+158 ~ +161`。
3. `confirmed.swap(true)` 若为 false → `peer.confirm_key(&keypair)`（首个数据报文确认密钥）偏移：`+164 ~ +167`。
4. `*peer.endpoint = endpoint`（sticky socket 更新）偏移：`+170`。
5. `inner_length(packet)` 解析内层 IP 长；`inbound.write(&packet[..inner])` 写 TUN 偏移：`+174 ~ +180`。
6. `C::recv(&opaque, size, true, keypair)` 回调 偏移：`+183`。
**错误处理**：解析/AAD 失败静默 return（不写 TUN，不回调）。

---

## 6. route.rs —— cryptokey 路由表

**Struct：`RoutingTable<T>`**（route.rs 结构：`RoutingTable`）：`ipv4/ipv6: RwLock<IpLookupTable<_,T>>`（`treebitmap` 最长前缀匹配）。
**函数：`insert`**（偏移：`+40`）：按 `ip.mask(cidr)` 插入，`v4/v6` 分流。
**函数：`get_route`**（偏移：`+75`）：读 IP 版本（首字节高 4 位）→ 解析 `IPv4Header`/`IPv6Header` 取目标地址 → `longest_match` 返回 peer 克隆。**参数**：`packet:&[u8]`。
**函数：`check_route`**（偏移：`+117`）：取**源**地址 `longest_match`，返回 `peer == 该 peer`（防 IP 伪装）。
**函数：`remove`/`list`**（偏移：`+62`/`+47`）：按值收集并删除/列举所有子网。
**并发**：`ipv4`/`ipv6` 各自 `RwLock`；插入写锁、查询读锁。

---

## 7. worker.rs —— worker 线程体

**Enum：`JobUnion`**（worker.rs 枚举：`JobUnion`）：`Outbound(SendJob)` / `Inbound(ReceiveJob)`。
**函数：`worker`**（worker.rs 函数：`worker` 偏移：`+15 ~ +35`）：
1. `receiver.recv()` 取 `JobUnion`；通道关闭 → break 偏移：`+20`。
2. `Inbound`：`job.parallel_work()` 后 `job.queue().consume()`；`Outbound` 同理 偏移：`+25 ~ +32`。
**设计意图**：`parallel_work` 在任意 worker 线程跑（并行），`consume` 在 job 所属 peer 的顺序队列上跑（顺序），实现「跨 peer 并行、同 peer 顺序」。

---

## 8. queue.rs —— 顺序+并行队列

**Trait：`SequentialJob`**（queue.rs 偏移：`+9`）：`is_ready()` + `sequential_work()`。
**Trait：`ParallelJob`**（偏移：`+15`）：`queue()` + `parallel_work()`。
**Struct：`Queue<J>`**（偏移：`+21`）：`contenders:AtomicUsize` + `queue:Mutex<ArrayDeque>`。
**函数：`push`**（偏移：`+40`）：`queue.lock().push_back(job)`。
**函数：`consume`**（偏移：`+44 ~ +91`）：
1. `pos = contenders.fetch_add(1)`；若 `pos>0` 说明已有竞争者，直接 return（避免重入）偏移：`+46 ~ +50`。
2. 否则进入临界区：循环取队首，若 `is_ready()` 则 `pop_front` 并 `sequential_work()`，直到空或不就绪 偏移：`+54 ~ +83`。
3. `contenders.fetch_sub(contenders)` 退出 偏移：`+89`。
**并发模型**：多生产者单消费者语义——任意线程可 push/consume，但同一时刻仅一个线程真正 drain 队列（contenders 计数保证互斥），实现无锁化的顺序处理。

---

## 9. anti_replay.rs —— RFC 6479 滑动窗口

**Struct：`AntiReplay`**（anti_replay.rs 结构：`AntiReplay`）：`bitmap:[Word; BITMAP_LEN]` + `last:u64`（64 位平台 `BITMAP_BITLEN=2048`）。
**函数：`check`**（偏移：`+50`）：
- `seq > last` → 通过（新）；
- `last - seq > WINDOW_SIZE` → 失败（太旧）；
- 否则查 bitmap 位是否被标记 偏移：`+52 ~ +63`。
**函数：`update_store`**（偏移：`+67`）：`seq>last` 时前移窗口并清零过期区间；置位 `seq` 对应 bit。
**函数：`update`**（偏移：`+103`）：`check` 通过才 `update_store`，返回 bool。
**设计意图**：允许乱序到达（窗口内），拒绝重放与超出窗口的旧包；零 counter 也允许（与 RFC 6479 不同，WireGuard 语义）。

---

## 10. ip.rs / messages.rs —— 辅助

**messages.rs**：`TYPE_TRANSPORT=4`（偏移：`+5`）；`TransportHeader{ f_type:U32, f_receiver:U32, f_counter:U64 }`（`#[repr(packed)]`，偏移：`+9`）。
**ip.rs**：`VERSION_IP4=4`/`VERSION_IP6=6`（偏移：`+8`）；`IPv4Header`/`IPv6Header` 结构（仅取源/目标与长度字段）；`inner_length(packet)`（偏移：`+32`）按版本解析内层 IP 总长，供 `receive.rs` 决定写多少字节到 TUN。

---

## 11. 出站/入站数据流小结

```text
出站：tun_worker → Device::send → table.get_route → Peer::send → SendJob(并行加密) → 顺序 consume → send_raw → UDP
入站：udp_worker → Device::recv → recv[receiver_id] → ReceiveJob(并行解密+check_route) → 顺序 consume(抗重放+confirm+写TUN)
```

> 配置面与密钥如何注入 router 见 `07`；密钥轮旋转与计时器见 `09`。

[上一篇](05-handshake模块函数级解析.md) · [总目录](README.md) · [下一篇](07-配置与UAPI函数级解析.md)
