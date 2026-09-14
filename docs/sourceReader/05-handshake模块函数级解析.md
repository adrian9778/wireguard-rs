[上一篇](04-核心模块与类型关系图.md) · [总目录](README.md) · [下一篇](06-router模块函数级解析.md)

# 第四层：handshake 模块函数级解析

> **版本**：0.1.4
> **源码基准**：当前源码
> **范围**：`src/wireguard/handshake/` 下 `device.rs` / `noise.rs` / `peer.rs` / `messages.rs` / `macs.rs` / `ratelimiter.rs` / `timestamp.rs`

## 1. 模块职责

实现 Noise_IKpsk2_25519_ChaChaPoly_BLAKE2s 握手状态机（`handshake/mod.rs` 头注释）。输入不可信的握手报文，输出（1）待回送报文、（2）协商出的 `KeyPair`。所有密钥材料在敏感函数返回前用 `clear_stack_on_return_fnonce` 清栈（noise.rs 宏）。

### 1.1 握手报文流（3 类消息 + cookie 风暴）

```text
发起方 I                               响应方 R
─────────                            ─────────
begin(rng, pk_r)
  → create_initiation
  → Initiation ────────────────────▶  process：consume_initiation
                                         → create_response
  ◀────────────────────────────────  Response
process：consume_response
  → KeyPair(initiator=true)
                                      KeyPair(initiator=false)
  （cookie 风暴时 insert 额外 CookieReply 往返，
   不影响最终 send/recv 对称密钥对齐）
```

> 入站 Initiation 的端到端路径见 `02`/`03`；`KeyPair` 如何交回路由层见 `06`（`router::PeerHandle::add_keypair`）。

---

## 2. device.rs —— 握手设备

### 2.1 类型

**Struct：`KeyState`**（device.rs 结构：`KeyState`）
字段：`sk: StaticSecret`、`pk: PublicKey`、`macs: macs::Validator`。设备级长期密钥与 MAC 校验器。

**Struct：`Device<O>`**（device.rs 结构：`Device`）
字段：
- `keyst: Option<KeyState>` —— 本设备私钥（未配置为 None）。
- `id_map: DashMap<u32,[u8;32]>` —— receiver id → 对端公钥（并发安全）。
- `pk_map: HashMap<[u8;32], Peer<O>>` —— 公钥 → peer（配置期持有写锁）。
- `limiter: Mutex<RateLimiter>` —— 全局限速器（DoS 缓解）。

**Enum/Type：`Output<'a, O>`**（types.rs Type：`Output`）= `(Option<&'a O>, Option<Vec<u8>>, Option<KeyPair>)`。

**Const：`MAX_PEER_PER_DEVICE = 1<<20`**（device.rs 偏移：`+27`）。

### 2.2 函数：`new`

**源码**：`src/wireguard/handshake/device.rs` 函数：`new`（偏移：`+97`）
**职责**：构造空握手设备。
**返回值**：`Device<O>`（`keyst=None`，`id_map`/`pk_map` 空，`limiter=RateLimiter::new()`）。
**状态变化**：无副作用。

### 2.3 函数：`set_sk`

**源码**：device.rs 函数：`set_sk`（偏移：`+134 ~ +156`）
**职责**：设置/替换设备私钥，重算所有 peer 的预共享秘密（static-static DH）。
**参数**：`sk: Option<StaticSecret>`。
**调用方**：`WireGuard::set_key`（`src/wireguard/wireguard.rs` 函数：`set_key`）。
**被调用**：`macs::Validator::new`、`update_ss`、`release`。
**执行流程**：
1. 构造 `KeyState`（含 `macs = Validator::new(pk)`）偏移：`+136 ~ +140`。
2. `update_ss()` 重算每 peer 的 `ss`，并 `reset_state()` 收集待释放 id 偏移：`+143`。
3. 释放被中止握手的 id 偏移：`+146 ~ +148`。
4. 若新公钥与某 peer 公钥相同，移除该 peer 并返回其公钥（防止自连）偏移：`+152 ~ +155`。
**错误处理**：无；`None` 仅清空 `keyst`。

### 2.4 函数：`add`

**源码**：device.rs 函数：`add`（偏移：`+174 ~ +201`）
**职责**：按公钥登记 peer，预计算 static-static 共享秘密。
**参数**：`pk: PublicKey`、`opaque: O`。
**前置条件**：调用方持有 `&mut self`（配置期写锁）。
**关键分支**：超过 `MAX_PEER_PER_DEVICE` → `Err(ConfigError::Too many peers)`；公钥等于设备公钥 → `Err(...matches the device)` 偏移：`+176 ~ +185`。
**被调用**：`Peer::new`、`diffie_hellman`。
**返回值**：`Result<(), ConfigError>`。
**状态变化**：`pk_map.insert(pk, Peer::new(pk, ss, opaque))` 偏移：`+188 ~ +198`。

### 2.5 函数：`allocate`

**源码**：device.rs 函数：`allocate`（偏移：`+463 ~ +478`）
**职责**：拒绝采样分配一个未被占用的本地 receiver id。
**参数**：`rng: &mut R`、`pk: &PublicKey`（用于 id_map 反向索引）。
**关键循环**：`loop { id = rng.gen(); if contains_key(id) continue; if Entry::Vacant insert; return id }` 偏移：`+464 ~ +477`。
**状态变化**：`id_map.insert(id, pk.as_bytes())`。
**返回值**：`u32` 本地 id。

### 2.6 函数：`lookup_pk` / `lookup_id`

- `lookup_pk`（偏移：`+436`）：`pk_map.get(pk.as_bytes())`，缺失 → `Err(UnknownPublicKey)`。
- `lookup_id`（偏移：`+445`）：先 `id_map.get(id)` 得公钥，再 `pk_map.get` 得 peer；缺失分别 → `UnknownReceiverId` / `unreachable!()`。

### 2.7 函数：`process`（核心）

**源码**：device.rs 函数：`process`（偏移：`+308 ~ +431`）
**所属**：`impl<O> Device<O>`
**职责**：处理一条不可信握手报文，分派类型并返回 `Output`。
**调用方**：`src/wireguard/workers.rs` `handshake_worker` 偏移：`+183`。
**被调用**：`Initiation::parse` / `Response::parse` / `CookieReply::parse`、`macs::Validator::{check_mac1,check_mac2,create_cookie_reply}`、`noise::{consume_initiation,create_response,consume_response}`、`allocate`、`lookup_id`、`peer.macs.generate`、`release`。
**参数**：`rng:&mut R`、`msg:&[u8]`、`src:Option<SocketAddr>`。
**返回值**：`Result<Output<'a,O>, HandshakeError>`。

**执行流程**：
1. 长度校验 `msg.len() < 4` → `InvalidMessageFormat` 偏移：`+315`。
2. 取 `keyst`；`None` 返回 `(None,None,None)` 空操作（未配置私钥）偏移：`+321`。
3. `LittleEndian::read_u32(msg)` 分派 偏移：`+329`：
   - `TYPE_INITIATION` 偏移：`+330`：
     - `Initiation::parse` 偏移：`+332`。
     - `check_mac1` 偏移：`+335`。
     - `src` 存在时 `check_mac2`，失败则 `create_cookie_reply` 并提前返回 `(None, Some(cookie_reply), None)` 偏移：`+338 ~ +356`。
     - `limiter.allow(&src.ip())` 失败 → `RateLimited` 偏移：`+353`。
     - `noise::consume_initiation` 偏移：`+359`。
     - `allocate` 本地 id 偏移：`+362`。
     - `noise::create_response` 偏移：`+368`（失败 `release(local)`）。
     - `peer.macs.generate` 偏移：`+375`。
     - 返回 `(Some(&peer.opaque), Some(resp), Some(keys))` 偏移：`+380`。
   - `TYPE_RESPONSE` 偏移：`+386`：parse → check_mac1 → mac2/cookie/ratelimit → `noise::consume_response`。
   - `TYPE_COOKIE_REPLY` 偏移：`+416`：`lookup_id` → `peer.macs.process`（不产出密钥，不加密验证对端）偏移：`+420 ~ +427`。
   - 其它 → `InvalidMessageFormat` 偏移：`+429`。

**错误处理**：各 `parse`/`check_mac1`/`consume_*` 的错误向上传播；`create_response` 失败先 `release(local)` 再传播。
**状态变化**：`id_map` 分配本地 id；`peer.state` 在 `consume_initiation` 内复位；`peer.macs.last_mac1` 在 `generate` 内更新。

### 2.8 函数：`begin`（发起方路径，非第二层主链路但重要）

**源码**：device.rs 函数：`begin`（偏移：`+278 ~ +301`）
**职责**：本地主动发起握手，构造 Initiation 字节。
**参数**：`rng:&mut R`、`pk:&PublicKey`。
**被调用**：`allocate`、`noise::create_initiation`、`peer.macs.generate`。
**返回值**：`Result<Vec<u8>, HandshakeError>`（Initiation 报文）。
**调用方**：`handshake_worker` 的 `HandshakeJob::New` 分支（workers.rs 偏移：`+257`）。

---

## 3. noise.rs —— Noise 密码学

### 3.1 常量与宏

- `INITIAL_CK` / `INITIAL_HS`（noise.rs Const）：预计算的链密钥与哈希转录（`HASH!(CONSTRUCTION)` / `HASH!(INITIAL_CK, IDENTIFIER)`），有单测 `precomputed_chain_key` / `precomputed_hash` 校验。
- 宏：`HASH!`(blake2)、`HMAC!`、`KDF1/KDF2/KDF3!`(HKDF 风格)、`SEAL!`/`OPEN!`(ChaCha20Poly1305 零 nonce AEAD)。
- `TemporaryState = (u32, PublicKey, GenericArray<u8,U32>, GenericArray<u8,U32>)`（noise.rs Type：`TemporaryState`）—— (receiver_id, eph_r_pk, hs, ck)。

### 3.2 函数：`shared_secret`

**源码**：noise.rs 函数：`shared_secret`（偏移：`+221 ~ +228`）
**职责**：X25519 DH 并做零共享密钥检查（与内核行为一致，非 Noise 标准）。
**参数**：`sk:&StaticSecret`、`pk:&PublicKey`。
**返回值**：`Result<SharedSecret, HandshakeError>`；`ss == 0^32` → `InvalidSharedSecret`。
**副作用**：无。

### 3.3 函数：`create_initiation`（发起方）

**源码**：noise.rs 函数：`create_initiation`（偏移：`+230 ~ +317`）
**所属**：`pub(super) fn`
**职责**：按 Noise_IK 构造 Initiation 的 noise 部分。
**参数**：`rng, keyst:&KeyState, peer:&Peer<O>, pk:&PublicKey, local:u32, msg:&mut NoiseInitiation`。
**关键分支**：`peer.ss == 0` → `InvalidSharedSecret` 偏移：`+241`。
**内部逻辑**：
1. `ck=INITIAL_CK; hs=HASH!(hs, pk)` 偏移：`+248 ~ +250`。
2. 生成 `eph_sk/eph_pk`；`KDF1!(ck, eph_pk)`；`msg.f_ephemeral=eph_pk` 偏移：`+257 ~ +270`。
3. `KDF2!(ck, DH(eph, S_pub))` → `SEAL!` 加密 `keyst.pk` 到 `msg.f_static` 偏移：`+274 ~ +283`。
4. `KDF2!(ck, peer.ss)` → `SEAL!` 加密 `timestamp::now()` 到 `msg.f_timestamp` 偏移：`+291 ~ +304`。
5. 写 `peer.state = State::InitiationSent{hs,ck,eph_sk,local}` 偏移：`+308 ~ +313`。
**状态变化**：对端 peer 进入 `InitiationSent`（等待响应）。
**错误处理**：`shared_secret` 失败传播。

### 3.4 函数：`consume_initiation`（响应方，第二层主链路）

**源码**：noise.rs 函数：`consume_initiation`（偏移：`+319 ~ +404`）
**职责**：解密对端 Initiation，还原对端 static 公钥与时间戳，产出 `TemporaryState`。
**参数**：`device:&'a Device<O>, keyst:&KeyState, msg:&NoiseInitiation`。
**执行流程**：
1. `ck=INITIAL_CK; hs=HASH!(hs, keyst.pk)` 偏移：`+329 ~ +331`。
2. `KDF1!(ck, msg.f_ephemeral)` 吸收对端瞬态 偏移：`+335`。
3. `shared_secret(keyst.sk, eph_r_pk)` → `KDF2!` → `OPEN!` 解密 `msg.f_static` 得 `pk` 偏移：`+344 ~ +355`。
4. `device.lookup_pk(&pk)` 找本地 peer 偏移：`+357`。
5. `peer.ss == 0` 检查 偏移：`+361`。
6. `peer.state = State::Reset`（清旧状态）偏移：`+367`。
7. 解密 `msg.f_timestamp` 得 `ts`，`peer.check_replay_flood(device,&ts)` 偏移：`+381 ~ +390`。
8. 返回 `(peer, pk, (msg.f_sender, eph_r_pk, hs, ck))` 偏移：`+398 ~ +402`。
**错误处理**：`OPEN!` 失败 → `DecryptionFailure`；`check_replay_flood` 失败 → `OldTimestamp`/`InitiationFlood`。
**状态变化**：peer.state 复位；时间戳/洪泛窗口在 `check_replay_flood` 内更新。

### 3.5 函数：`create_response`（响应方，第二层主链路）

**源码**：noise.rs 函数：`create_response`（偏移：`+406 ~ +488`）
**职责**：基于 `TemporaryState` 构造 Response 并派生 `KeyPair`。
**参数**：`rng, peer, pk, local:u32, state:TemporaryState, msg:&mut NoiseResponse`。
**执行流程**：
1. 解包 `state → (receiver, eph_r_pk, hs, ck)` 偏移：`+418`。
2. `msg.f_type=2; f_sender=local; f_receiver=receiver` 偏移：`+420 ~ +422`。
3. 生成响应方 `eph_sk/eph_pk`；`KDF1!(ck, eph_pk)` 偏移：`+426 ~ +439`。
4. `KDF1!(ck, DH(eph, eph_r))`、`KDF1!(ck, DH(eph, pk))` 两次吸收 偏移：`+443 ~ +447`。
5. `KDF3!(ck, peer.psk) → (ck, tau, key)`；`SEAL!` 空明文到 `msg.f_empty`（PSK 绑定）偏移：`+451 ~ +464`。
6. `KDF2!(ck, &[]) → (key_recv, key_send)` 偏移：`+471`。
7. 返回 `KeyPair{ birth, initiator:false, send:{id:receiver,key:key_send}, recv:{id:local,key:key_recv} }` 偏移：`+475 ~ +486`。
**关键**：`send.id` 指向对端 sender id，`recv.id` 指向本端刚 `allocate` 的 id。
**状态变化**：无 peer 状态写（密钥经 `add_keypair` 在 router 侧落地）。

### 3.6 函数：`consume_response`（发起方确认）

**源码**：noise.rs 函数：`consume_response`（偏移：`+494 ~ +590`）
**职责**：发起方收到 Response，确认握手并产出已确认 `KeyPair`（`initiator=true`）。
**关键**：先取 `peer.state` 的 `InitiationSent`，跑完 Noise 后半程后释放 state 锁（防 DoS），再重新校验 state 未变，变则 `InvalidState`（抗重放响应）。
**返回值**：`Output` 中 `keypair.initiator=true`，`send.id=remote`、`recv.id=local`。

---

## 4. peer.rs —— 握手侧 peer 状态

### 4.1 类型

**Struct：`Peer<O>`**（peer.rs 结构：`Peer`）：`opaque:O`、`state:Mutex<State>`、`timestamp`、`last_initiation_consumption`、`macs:Mutex<macs::Generator>`、`ss:[u8;32]`、`psk:Psk`。

**Enum：`State`**（peer.rs 枚举：`State`）：`Reset` 或 `InitiationSent{ local, eph_sk, hs, ck }`。`Drop` 实现清零 `hs`/`ck`（peer.rs 偏移：`+51`）。

### 4.2 函数：`check_replay_flood`

**源码**：peer.rs 函数：`check_replay_flood`（偏移：`+87 ~ +120`）
**职责**：防时间戳重放与发起洪泛。
**参数**：`device:&Device<O>`、`timestamp_new:&TAI64N`。
**关键分支**：
- 旧时间戳 `!timestamp::compare(old, new)` → `OldTimestamp` 偏移：`+97 ~ +101`。
- 距上次 `last_initiation_consumption < TIME_BETWEEN_INITIATIONS`(20ms) → `InitiationFlood` 偏移：`+104 ~ +108`。
- 进入新状态前 `device.release(local)` 释放旧握手 id 偏移：`+111 ~ +113`。
**状态变化**：`state=Reset`、`timestamp=Some(new)`、`last_initiation_consumption=Now` 偏移：`+116 ~ +118`。
**错误处理**：返回 `Err` 即中止本次 `consume_initiation`。

---

## 5. messages.rs —— 报文线格式

**Const**：`TYPE_INITIATION=1`、`TYPE_RESPONSE=2`、`TYPE_COOKIE_REPLY=3`（messages.rs 偏移：`+22 ~ +24`）；`MAX_HANDSHAKE_MSG_SIZE`（取三报文最大，偏移：`+31`）。
**Struct**：`Initiation`/`Response` = `NoiseInitiation`/`NoiseResponse` + `MacsFooter`（`#[repr(packed)]` + `FromBytes,AsBytes`，零拷贝，`#[derive]` 见偏移：`+38 ~ +59`）。
**解析函数**：`Initiation::parse`/`Response::parse`/`CookieReply::parse`（偏移：`+92 ~ +129`）→ `LayoutVerified::new` + `f_type` 断言。
**设计意图**：`#[repr(packed)]` 使 `&[u8]` 可直接 zero-copy 解释为结构体，避免拷贝；`LayoutVerified` 在运行时校验对齐/长度。

---

## 6. macs.rs —— MAC1 / MAC2 / Cookie

### 6.1 常量

`LABEL_MAC1=b"mac1----"`、`LABEL_COOKIE=b"cookie--"`；`COOKIE_UPDATE_INTERVAL=120s`（macs.rs 偏移：`+22 ~ +30`）。

### 6.2 `Validator`（校验对端发来的 mac）

- `new`（偏移：`+194`）：由设备公钥派生 `mac1_key`/`cookie_key`。
- `check_mac1`（偏移：`+264`）：`MAC!(mac1_key, inner).ct_eq(mac1)` 常量时间比较，失败 `InvalidMac1`。
- `check_mac2`（偏移：`+273`）：取 `get_tau(src)`（`MAC!(secret, src)`，每 120s 轮换），`MAC!(tau, inner, mac1).ct_eq(mac2)`。
- `create_cookie_reply`（偏移：`+237`）：用 `cookie_key` 以 XChaCha20Poly1305 密封 `tau` 成 CookieReply。
- `get_tau`/`get_set_tau`（偏移：`+205`/`+214`）：惰性生成/缓存 cookie 密钥（双检查锁避免竞态）。

### 6.3 `Generator`（本端为某 peer 生成 mac）

- `new`（偏移：`+122`）：由该 peer 公钥派生 `mac1_key`/`cookie_key`。
- `generate`（偏移：`+165`）：`f_mac1=MAC!(mac1_key, inner)`；若有 cookie 则 `f_mac2=MAC!(cookie, inner, mac1)` 否则清零；记录 `last_mac1`（供 `process` 解 CookieReply 时比对）。
- `process`（偏移：`+141`）：用 `cookie_key` 开 CookieReply，得到 `self.cookie`，供后续 `generate` 填 mac2。
- `addr_to_mac_bytes`（偏移：`+95`）：把 `SocketAddr` 转成 `(ip 字节 ++ 端口 le)` 用作 mac2 地址域。

---

## 7. ratelimiter.rs —— 令牌桶限速

**Struct：`RateLimiter`（Arc 包裹 inner）**（ratelimiter.rs 结构：`RateLimiter`）：每 `IpAddr` 一个 `Entry{last_time, tokens}`。
**Const**：`PACKETS_PER_SECOND=20`、`PACKETS_BURSTABLE=5`、`PACKET_COST=1e9/20`、`MAX_TOKENS=PACKET_COST*5`、`GC_INTERVAL=1s`（偏移：`+8 ~ +13`）。

### 7.1 函数：`allow`

**源码**：ratelimiter.rs 函数：`allow`（偏移：`+48 ~ +108`）
**职责**：令牌桶判定是否放行某 IP 的握手报文（DoS 缓解）。
**参数**：`addr:&IpAddr`。
**执行流程**：
1. 已存在条目：按流逝时间补 token（`min(MAX, tokens+elapsed_ns)`），扣 `PACKET_COST`，`>0` 放行 偏移：`+52 ~ +68`。
2. 新条目：插入 `tokens=MAX-Cost`，放行 偏移：`+70 ~ +78`。
3. 若无 GC 线程在跑，启动一个（按 1s 周期清理过期条目）偏移：`+82 ~ +105`。
**返回值**：`bool`（true=放行）。
**并发**：`table` 用 `spin::RwLock`；GC 用 `AtomicBool` + `Condvar` 协作退出（`Drop` 唤醒，偏移：`+28`）。

---

## 8. timestamp.rs —— TAI64N 时间戳

**Type：`TAI64N = [u8;12]`**（timestamp.rs 偏移：`+3`）。
- `now`（偏移：`+9`）：UNIX 时间 + `TAI64_EPOCH` 转 8 字节秒 + 4 字节纳秒，大端序列化。
- `compare`（偏移：`+25`）：逐字节字典序比较，新 > 旧返回 true（用于防重放）。
- `ZERO`（偏移：`+7`）：零时间戳常量。

> 第二层主链路涉及的 `consume_initiation`/`check_replay_flood` 已在本篇与第三层说明。下一层解析 router 模块。

[上一篇](04-核心模块与类型关系图.md) · [总目录](README.md) · [下一篇](06-router模块函数级解析.md)
