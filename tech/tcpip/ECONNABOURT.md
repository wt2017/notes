# Linux 6.12 TCP socket 返回 ECONNABORTED 的代码分支分析

> 基于内核源码树 `d:\CODE\linux-6.12`(6.12.0,Makefile `VERSION=6 PATCHLEVEL=12`)的代码走读。
> 文中 `file:line` 均为该源码树中的位置。

## 结论速览

内核中把 **TCP socket 的 `sk_err` 直接写成 `ECONNABORTED` 的位置只有两处源头**:

1. `tcp_disconnect()` 的 `TCP_REPAIR` 分支 —— `net/ipv4/tcp.c:3285`
2. `tcp_abort(sk, ECONNABORTED)` → `tcp_done_with_error()` —— `net/ipv4/tcp_input.c:4512`

其余都是这两个源头的传播路径,外加两处"纯软件兜底"返回值:

3. `__inet_stream_connect()` 中 `connect()` 的 `sock_error() ?: -ECONNABORTED` 兜底 —— `net/ipv4/af_inet.c:735`
4. `accept4()` 中 `getname()` 失败的通用兜底 —— `net/socket.c:1959`

> 注意:`sk_err` 中存放的是**正数** errno;`sock_error()`(`include/net/sock.h:2390`)用 `xchg(&sk->sk_err, 0)` 取出并清零后返回**负数**。下文"设置 ECONNABORTED"指写入正数 103,用户态看到的是 `-ECONNABORTED`。

---

## 一、源头 A:`tcp_disconnect()` 的 TCP_REPAIR 分支

`net/ipv4/tcp.c:3270`:

```c
int tcp_disconnect(struct sock *sk, int flags)
{
	struct inet_sock *inet = inet_sk(sk);
	struct inet_connection_sock *icsk = inet_csk(sk);
	struct tcp_sock *tp = tcp_sk(sk);
	int old_state = sk->sk_state;
	...
	if (old_state != TCP_CLOSE)
		tcp_set_state(sk, TCP_CLOSE);

	/* ABORT function of RFC793 */
	if (old_state == TCP_LISTEN) {
		inet_csk_listen_stop(sk);                  /* 优先级最高,不设错误 */
	} else if (unlikely(tp->repair)) {
		WRITE_ONCE(sk->sk_err, ECONNABORTED);      /* tcp.c:3285  ★ */
	} else if (tcp_need_reset(old_state)) {
		tcp_send_active_reset(sk, gfp_any(), SK_RST_REASON_TCP_STATE);
		WRITE_ONCE(sk->sk_err, ECONNRESET);        /* 非 repair:RST 路径 */
	} else if (tp->snd_nxt != tp->write_seq &&
		   (1 << old_state) & (TCPF_CLOSING | TCPF_LAST_ACK)) {
		tcp_send_active_reset(sk, gfp_any(),
				      SK_RST_REASON_TCP_DISCONNECT_WITH_DATA);
		WRITE_ONCE(sk->sk_err, ECONNRESET);
	} else if (old_state == TCP_SYN_SENT)
		WRITE_ONCE(sk->sk_err, ECONNRESET);
	...
	sk_error_report(sk);                           /* tcp.c:3401 */
	return 0;
}
```

### 触发条件

`tp->repair == 1`,即用户先 `setsockopt(TCP_REPAIR, TCP_REPAIR_ON)`:

- `tcp_can_repair_sock()`(`net/ipv4/tcp.c:3406`)要求进程具备目标 netns user namespace 的 `CAP_NET_ADMIN`,且 socket 状态不是 `TCP_LISTEN`;
- 注意判断顺序:若 `old_state == TCP_LISTEN`,会先走 `listen_stop`,**不会**设置 `ECONNABORTED`。

### `tcp_disconnect()` 在 TCP 上的入口

`tcp_disconnect` 注册为 `sk->sk_prot->disconnect`(`tcp_ipv4.c:3349`、`tcp_ipv6.c:2322`),用户态可达路径:

| 入口 | 位置 | 说明 |
|---|---|---|
| `connect(fd, AF_UNSPEC)` | `net/ipv4/af_inet.c:646-651` | `uaddr->sa_family == AF_UNSPEC` 时直接调用 `sk->sk_prot->disconnect(sk, flags)` |
| `close()`(repair socket) | `net/ipv4/tcp.c:3086-3087`(`__tcp_close`) | `if (unlikely(tcp_sk(sk)->repair)) sk->sk_prot->disconnect(sk, 0);` |
| `shutdown()` | `net/ipv4/af_inet.c:916-937` | `TCP_LISTEN` + `SHUT_RD`,或 `TCP_SYN_SENT` 状态走 `disconnect` |

### 观察方式

- `connect(AF_UNSPEC)` 本身返回 0(disconnect 的返回值),但 `sk_err` 已置位:
  后续 `recv`/`connect`/`getsockopt(SO_ERROR)` 会取到 `-ECONNABORTED`;
- `close()` 分支本身不返回错误;若同一 socket 还有 `dup()` 出去的 fd、正阻塞在 `recv`/`poll` 上,该线程会被 `sk_error_report()` 唤醒并读到 `ECONNABORTED`;
- 普通(非 repair)socket 的 `SO_LINGER={1,0}` 或 `connect(AF_UNSPEC)` 走后面的分支,得到 `ECONNRESET`,**不是** `ECONNABORTED`。

---

## 二、源头 B:`tcp_abort()` → `tcp_done_with_error()`

### `tcp_abort()` —— `net/ipv4/tcp.c:4874`

```c
int tcp_abort(struct sock *sk, int err)
{
	int state = inet_sk_state_load(sk);

	if (state == TCP_NEW_SYN_RECV) {           /* 只丢弃 request sock,不设 err */
		struct request_sock *req = inet_reqsk(sk);

		local_bh_disable();
		inet_csk_reqsk_queue_drop(req->rsk_listener, req);
		local_bh_enable();
		return 0;
	}
	if (state == TCP_TIME_WAIT) {              /* 只销毁 timewait,不设 err */
		struct inet_timewait_sock *tw = inet_twsk(sk);

		refcount_inc(&tw->tw_refcnt);
		local_bh_disable();
		inet_twsk_deschedule_put(tw);
		local_bh_enable();
		return 0;
	}

	/* BPF context ensures sock locking. */
	if (!has_current_bpf_ctx())
		lock_sock(sk);

	/* Avoid closing the same socket twice. */
	if (sk->sk_state == TCP_CLOSE) {           /* 已关闭 → -ENOENT */
		...
		return -ENOENT;
	}

	if (sk->sk_state == TCP_LISTEN) {
		tcp_set_state(sk, TCP_CLOSE);
		inet_csk_listen_stop(sk);
	}
	...
	if (tcp_need_reset(sk->sk_state))
		tcp_send_active_reset(sk, GFP_ATOMIC,
				      SK_RST_REASON_TCP_STATE);
	tcp_done_with_error(sk, err);              /* tcp.c:4920 */
	...
}
```

### `tcp_done_with_error()` —— `net/ipv4/tcp_input.c:4509`

```c
void tcp_done_with_error(struct sock *sk, int err)
{
	/* This barrier is coupled with smp_rmb() in tcp_poll() */
	WRITE_ONCE(sk->sk_err, err);               /* tcp_input.c:4512  ★ 正数存入 */
	smp_wmb();

	tcp_write_queue_purge(sk);
	tcp_done(sk);                              /* 状态置 TCP_CLOSE */
	if (!sock_flag(sk, SOCK_DEAD))
		sk_error_report(sk);
}
```

### 以 `err = ECONNABORTED` 调用 `tcp_abort()` 的调用方

| # | 调用链 | 位置 | 触发方式 |
|---|---|---|---|
| 1 | `tcp_diag_destroy` → `sock_diag_destroy(sk, ECONNABORTED)` → `sk->sk_prot->diag_destroy` = `tcp_abort` | `net/ipv4/tcp_diag.c:202-221`(216);`net/core/sock_diag.c:315-324` | `NETLINK_SOCK_DIAG` 的 `SOCK_DESTROY`,即 `ss -K`;需 `CAP_NET_ADMIN` |
| 2 | BPF kfunc `bpf_sock_destroy()` | `net/core/filter.c:12146-12159` | cgroup BPF 程序;注释明确写明 "Destroy the given socket with ECONNABORTED error code",仅 TCP/UDP 有 `diag_destroy` |
| 3 | MPTCP subflow 覆写 `tcp_abort_override` → `tcp_abort(ssk, err)` | `net/mptcp/subflow.c:2096-2105`,注册于 2151/2187 | 对 MPTCP subflow 做 `SOCK_DESTROY` |
| 4 | SMC 内部 `tcp_abort(smc->clcsock->sk, ECONNABORTED)` | `net/smc/af_smc.c:327`、`net/smc/smc_close.c:134-136` | SMC 内部销毁底层 TCP clcsock(用户看到的是 SMC socket,不是裸 TCP fd) |

`diag_destroy` 的注册点:`net/ipv4/tcp_ipv4.c:3391`、`net/ipv6/tcp_ipv6.c:2365`。

### 可达状态范围

除以下情况外,`sk_err` 都会写成 `ECONNABORTED`:

- `TCP_NEW_SYN_RECV`:只丢弃 req,返回 0;
- `TCP_TIME_WAIT`:只销毁 timewait,返回 0;
- 已经是 `TCP_CLOSE`:返回 `-ENOENT`。

其余 `SYN_SENT / SYN_RECV / ESTABLISHED / FIN_WAIT* / CLOSE_WAIT / LAST_ACK / CLOSING / LISTEN` 均可达;`LISTEN` 还会先 `inet_csk_listen_stop()`。

---

## 三、纯软件兜底分支(未写入 `sk_err`)

### 1. `connect()` 失败兜底 —— `net/ipv4/af_inet.c:734-735`

```c
	/* Connection was closed by RST, timeout, ICMP error
	 * or another process disconnected us.
	 */
	if (sk->sk_state == TCP_CLOSE)
		goto sock_error;
	...
sock_error:
	err = sock_error(sk) ? : -ECONNABORTED;    /* af_inet.c:735  ★ */
	sock->state = SS_UNCONNECTED;
	sk->sk_disconnects++;
	if (sk->sk_prot->disconnect(sk, flags))
		sock->state = SS_DISCONNECTING;
	goto out;
```

- 阻塞 `connect()` 等到状态离开 `TCP_SYN_SENT/SYN_RECV` 后若为 `TCP_CLOSE` 且 `sk_err == 0`,返回 `-ECONNABORTED`(兜底);
- 若 `sk_err` 非 0(例如已被上文 `tcp_abort()` 或 repair-disconnect 写成 `ECONNABORTED`),`sock_error()` 原样返回 `-ECONNABORTED`;
- `sock_error()` 会**清除** `sk_err`。

> 复用同一分支的还有:IPv6(`inet6_stream_ops` 的 `.connect = inet_stream_connect`,`net/ipv6/af_inet6.c:692`)和 MPTCP(`net/mptcp/protocol.c:4079/4178` 也用 `inet_stream_connect`)。

### 2. `accept4()` 的 `getname` 失败 —— `net/socket.c:1956-1961`

```c
	if (upeer_sockaddr) {
		len = ops->getname(newsock, (struct sockaddr *)&address, 2);
		if (len < 0) {
			err = -ECONNABORTED;           /* net/socket.c:1959  ★ 通用 socket 层 */
			goto out_fd;
		}
```

对 TCP 而言,`inet_getname(..., peer=2)` 只在 `inet_dport == 0` 时返回 `-ENOTCONN`(`net/ipv4/af_inet.c:807-813`),正常 accept 出来的子 socket 都带 dport,所以这是通用兜底、TCP 实际几乎不可达。它**不是**"accept 队列中连接被 abort"的路径。

### 3. 不是 `ECONNABORTED` 的 accept 失败分支(易混淆)

`inet_csk_accept()`(`net/ipv4/inet_connection_sock.c:664`)本身**没有** `ECONNABORTED`:

- 监听 socket 状态不是 `TCP_LISTEN` → `-EINVAL`(677-679);
- 等待期间 listener 状态变化 → `-EINVAL`(647-649);
- 非阻塞无连接 → `-EAGAIN`;超时/信号 → `sock_intr_errno()`。

也就是说,listener 被 `tcp_abort()` 销毁后,`accept()` 得到的是 `-EINVAL`(状态已变 `TCP_CLOSE`),而不是 `ECONNABORTED`;但对该 listener 调 `getsockopt(SO_ERROR)` 会读到 `ECONNABORTED`。

---

## 四、`sk_err = ECONNABORTED` 之后,哪些系统调用能看到

`sk_err` 是 socket 级错误,由各操作通过 `sock_error()` 取走(取一次即清除):

| 操作 | 位置 | 返回 |
|---|---|---|
| `read` / `recv` / `recvmsg` | `net/ipv4/tcp.c:2662-2670`(`copied = sock_error(sk)`) | `-ECONNABORTED`(若已拷出数据则先返回数据,下次再报错) |
| `splice()`(`tcp_splice_read`) | `net/ipv4/tcp.c:824-826` | `-ECONNABORTED` |
| `recvmsg` + `TCP_ZEROCOPY_RECEIVE` | `net/ipv4/tcp.c:4609-4610`(`zc.err = sock_error(sk)`) | ioctl 返回 0,`zc.err = -ECONNABORTED` |
| `getsockopt(SO_ERROR)` | `include/net/sock.h:2390` | `-ECONNABORTED`(并清除) |
| `connect()` | `net/ipv4/af_inet.c:735` | `-ECONNABORTED`(兜底或透传) |
| `poll` / `epoll` | `net/ipv4/tcp.c:607-609` | 只置 `EPOLLERR`,errno 需 `SO_ERROR` 取 |
| `write` / `sendmsg` | `net/ipv4/tcp.c:1143-1145` | **固定 `-EPIPE`**,不是 `ECONNABORTED` |

注意:

- `tcp_zerocopy_receive` 的 `sock_error()` 会清掉 `sk_err`;
- `sendmsg` 开头只判断 `sk->sk_err` 非零就返回 `-EPIPE`,会掩盖具体错误码;
- `tcp_poll` 通过 `smp_rmb()` 与 `tcp_done_with_error()` 的 `smp_wmb()` 配对,保证 `EPOLLERR` 可见性。

---

## 五、形似但**不会**产生 `ECONNABORTED` 的同源分支

这些都会走到 `tcp_done_with_error()` 或设置 `sk_err`,但错误码不同:

- `tcp_reset()`(收到 RST):`ECONNREFUSED`(SYN_SENT)/ `EPIPE`(CLOSE_WAIT)/ `ECONNRESET`(其他)—— `net/ipv4/tcp_input.c:4537-4549`;
- ICMP 错误:`tcp_v4_err()` 中 `SYN_SENT/SYN_RECV` 直接 `tcp_done_with_error(sk, err)`(`net/ipv4/tcp_ipv4.c:625-627`,IPv6 为 `net/ipv6/tcp_ipv6.c:494-496`),已建立时写 `sk_err`(`net/ipv4/tcp_ipv4.c:649-650`);err 来自 `icmp_err_convert`,只会是 `ECONNREFUSED/EHOSTUNREACH/ENETUNREACH/EPROTO/EMSGSIZE` 等;
- 重传超时:`tcp_write_err()` → `sk_err_soft ?: ETIMEDOUT`(`net/ipv4/tcp_timer.c:77`);
- `TCP_REPAIR` 之外的 `tcp_disconnect()` 各分支:`ECONNRESET`;
- `close()` 带 `SO_LINGER={1,0}`:`tcp_disconnect()` → `tcp_need_reset(ESTABLISHED)` → `ECONNRESET`;
- `tcp_abort()` 作用于 `TCP_NEW_SYN_RECV` / `TCP_TIME_WAIT`:直接返回 0,不设错误。

---

## 六、汇总调用图

```
sk_err = ECONNABORTED  (正数存储,用户态看到 -ECONNABORTED)
│
├─ tcp_disconnect()  [tcp.c:3285]        ← 需 TCP_REPAIR + CAP_NET_ADMIN
│    ├─ connect(fd, AF_UNSPEC)           af_inet.c:646
│    ├─ close() / __tcp_close            tcp.c:3086-3087
│    └─ shutdown() LISTEN/SYN_SENT       af_inet.c:916-937
│
├─ tcp_done_with_error() [tcp_input.c:4512] ← tcp_abort(sk, ECONNABORTED)
│    ├─ inet_diag SOCK_DESTROY (ss -K)   tcp_diag.c:216
│    ├─ bpf_sock_destroy() kfunc         filter.c:12159
│    ├─ MPTCP subflow diag_destroy       mptcp/subflow.c:2104
│    └─ SMC 内部 clcsock                 smc/af_smc.c:327, smc/smc_close.c:136
│
└─ 兜底返回值(不写 sk_err):
     ├─ connect() 状态 TCP_CLOSE 且 sk_err==0   af_inet.c:735
     └─ accept4() getname 失败                  net/socket.c:1959
```

---

## 七、复现手段

如果目标是**稳定让一个 TCP fd 报 `ECONNABORTED`**,用户态可用的手段只有两类:

1. `CAP_NET_ADMIN` + `ss -K`(`SOCK_DESTROY`)销毁目标连接,之后该 socket 上的 `recv`/`SO_ERROR` 返回 `ECONNABORTED`;
2. `CAP_NET_ADMIN` + `TCP_REPAIR` 打开后,对该 fd 做
   - `connect(AF_UNSPEC)`,或
   - `shutdown(SHUT_RD)`(listener)/ `shutdown()`(SYN_SENT),或
   - `close()`(需有其他 fd 或阻塞线程观察)。

补充:BPF 侧可用 `bpf_sock_destroy()` kfunc 达到与 `SOCK_DESTROY` 相同的效果(同样以 `ECONNABORTED` 销毁)。

---

## 附:非 TCP 协议中的同名错误(便于区分)

`net/` 下其他协议也有 `ECONNABORTED`,但与本主题无关:

- SMC socket 自身:`net/smc/smc_core.c:1505`、`net/smc/smc_close.c:134`;
- TIPC:`net/tipc/socket.c:2835`;
- SCTP:`net/sctp/sm_statefuns.c`(多处 `SCTP_ERROR(ECONNABORTED)`);
- rxrpc、bluetooth、llc、9p 等。
