5G SSL VPN 技术白皮书（终版）》【Markdown】

定位：对内评审 / 对外技术说明 / 投资或客户交流
特点：不吹概念，全是工程可落地设计

1. 引言

随着云化、移动化和高并发场景的发展，传统 SSL VPN 在吞吐、延迟和并发连接数方面逐渐暴露出瓶颈。
本文给出一套 基于 TUN 虚拟网卡 + TLS 的 5Gbps 级 SSL VPN 数据面设计方案，重点解决：

TLS record 碎片化

syscall / memcpy 放大

TCP 背压导致的吞吐不稳定

长连接高并发下的可扩展性问题

2. 总体架构

本方案采用分层解耦设计，分为控制面（Control Plane）和数据面（Data Plane）：

- **控制面**：负责认证、配置和管理，确保系统安全和灵活运维。
- **数据面**：专注于高性能数据转发，采用多队列 TUN 虚拟网卡与多线程 TLS Worker 绑定，实现高并发与高吞吐。

架构示意：

```
+----------------------+
| Control Plane        |
| 认证 / 配置 / 管理   |
+----------------------+

+----------------------+
| Data Plane           |
|                      |
|  TUN (multi-queue)   |
|        ↓             |
|  TLS Workers (N)     |
|        ↓             |
|   TCP / TLS          |
+----------------------+
```

**核心设计要点：**
- 控制面与数据面彻底解耦，互不影响。
- 每个 TUN 队列独立绑定一个 TLS Worker，充分利用多核资源。
- 数据面全事件驱动（epoll-only），无阻塞高效处理数据流。
```
+----------------------+
| Control Plane        |
| 认证 / 配置 / 管理   |
+----------------------+
           |
           v
+----------------------+
| Data Plane           |
|                      |
|  TUN (multi-queue)   |
|        ↓             |
|  TLS Workers (N)     |
|        ↓             |
|   TCP / TLS          |
+----------------------+
```
该架构确保了系统的高性能、可扩展性和易维护性，适用于大规模 5G VPN 场景。


核心思想：

控制面与数据面彻底解耦

每个 TUN 队列绑定一个 TLS Worker

数据面全事件驱动（epoll-only）

3. 数据路径设计

3.1 TUN → TLS（发送方向）

```
TUN packet
    ↓ readv()
packet batch
    ↓ append
byte stream buffer
    ↓ SSL_write()
TLS records
    ↓ TCP
```



关键点：

packet / stream 解耦

不保留 IP 包边界

批量 append 后统一加密

3.2 TLS → TUN（接收方向）

```
TCP
    ↓
TLS record
    ↓ SSL_read()
plaintext bytes
    ↓
TUN write
```

接收侧同样采用状态机设计，确保数据流的有序处理和高效转发：

- 监听 sock_fd 的 EPOLLIN 事件，触发 handle_tls_read。
- 循环调用 SSL_read，读取解密后的明文数据。
- 每次读取到数据后，立即写入 TUN 设备，实现数据下发。
- 若遇到 SSL_ERROR_WANT_READ，说明当前无更多可读数据，退出循环，等待下次事件触发。
- 正确处理连接关闭和异常，保证稳定性。

这种设计保证了 TLS 解密与 TUN 写入的高效协作，避免阻塞和乱序问题，适应高并发场景下的数据接收需求。

遇到 SSL_ERROR_WANT_READ 立即停止

4. TLS Worker 设计
4.1 epoll-only 事件模型

监听对象：

tun_fd（EPOLLIN）

tls_sock_fd（EPOLLIN | EPOLLOUT）

timerfd（flush 兜底）

4.2 发送状态机（TX）

状态定义：

IDLE：无待发送数据

READY：可写，未阻塞

BLOCKED：send buffer 背压

核心状态位：

bool want_write;


语义：

want_write = true
→ 上次 SSL_write 遭遇 send buffer 背压

want_write = false
→ 发送路径畅通，可继续 flush

4.3 flush 策略

条件 flush

out_buf ≥ 12–16KB

定时兜底 flush

100–300 µs

目标：

TLS record 尽量接近 MSS

低流量场景不引入额外延迟

5. 性能关键优化点

5.1 批量 syscall

- TUN RX：使用 `readv` 实现批量读取，减少系统调用次数。
- TLS TX：批量调用 `SSL_write`，提升加密和发送效率。

5.2 TLS record 控制

- 通过 `SSL_CTX_set_max_send_fragment(ctx, 16*1024);` 控制 TLS record 大小，接近 MSS，减少碎片化。
- 启用 `SSL_set_mode(ssl, SSL_MODE_RELEASE_BUFFERS);`，降低内存占用。

5.3 乱序处理

- 设计时充分考虑数据包乱序的可能性，确保数据面状态机能正确处理乱序到达的数据。
- 通过事件驱动和状态管理，避免因乱序导致的数据丢失或阻塞。


5.3 背压显式建模

不忙轮询

不隐式阻塞

EPOLLOUT 精确触发

6. 稳定性与可靠性

正确处理 SSL_ERROR_WANT_WRITE / WANT_READ

send buffer 满时不丢数据

支持 TLS Worker 热重建

长连接 10k+ 稳定运行
6.1 乱序处理

在高并发和复杂网络环境下，数据包乱序是常见现象。为保证数据面稳定高效，设计时需充分考虑乱序的影响：

- **事件驱动状态机**：所有数据收发均通过 epoll 事件驱动，避免因乱序导致阻塞或死锁。
- **解耦包与流**：TUN 层与 TLS 层解耦，不保留 IP 包边界，所有数据以流方式处理，天然适应乱序。
- **状态管理**：每个 TLS Worker 独立维护收发缓冲区和状态，确保即使乱序到达也能正确处理和转发。
- **无隐式阻塞**：所有 I/O 操作均为非阻塞，遇到 SSL_ERROR_WANT_READ/WRITE 时立即退出，等待下次事件触发，避免因乱序或背压导致的性能抖动。
- **数据完整性保障**：通过批量 syscall 和缓冲区管理，确保数据不会因乱序丢失或重复。

这种设计保证了即使在极端乱序场景下，系统依然能够高效、稳定地收发数据，满足 5G VPN 的高并发和高可靠性需求。

### 乱序处理核心实现

在高并发和复杂网络环境下，数据包乱序是常见现象。为保证数据面稳定高效，核心实现需充分考虑乱序的影响：

#### 1. 包与流解耦

- TUN 层与 TLS 层解耦，不保留 IP 包边界，所有数据以流方式处理，天然适应乱序。
- 通过批量 syscall（如 `readv`）和缓冲区管理，避免因乱序导致的数据丢失或重复。

#### 2. 事件驱动状态机

- 所有数据收发均通过 epoll 事件驱动，避免因乱序导致阻塞或死锁。
- 每个 TLS Worker 独立维护收发缓冲区和状态，确保即使乱序到达也能正确处理和转发。

#### 3. 非阻塞 I/O 与错误处理

- 所有 I/O 操作均为非阻塞，遇到 `SSL_ERROR_WANT_READ`/`WRITE` 时立即退出，等待下次事件触发，避免因乱序或背压导致的性能抖动。
- 明确建模背压状态（`want_write`），不忙轮询、不隐式阻塞，确保高并发下的稳定性。

#### 4. 关键代码片段说明

```c
// 事件循环，确保所有 I/O 操作事件驱动
while (running) {
    int n = epoll_wait(epfd, evs, MAX_EVENTS, -1);
    for (int i = 0; i < n; i++) {
        if (evs[i].data.fd == tun_fd)
            handle_tun_read(w);
        else if (evs[i].data.fd == sock_fd) {
            if (evs[i].events & EPOLLIN)
                handle_tls_read(w);
            if (evs[i].events & EPOLLOUT)
                handle_tls_flush(w);
        }
        else if (evs[i].data.fd == timer_fd)
            force_flush(w);
    }
}

// TLS → TUN，遇到 WANT_READ 立即退出，避免阻塞
void handle_tls_read(struct tls_worker *w)
{
    while (1) {
        int n = SSL_read(w->ssl, w->rx_buf, sizeof(w->rx_buf));
        if (n > 0) {
            write(w->tun_fd, w->rx_buf, n);
            continue;
        }
        int err = SSL_get_error(w->ssl, n);
        if (err == SSL_ERROR_WANT_READ)
            break;
        // close / error
        break;
    }
}
```

#### 5. 设计优势

- 通过事件驱动和状态管理，系统可在极端乱序场景下依然高效、稳定地收发数据。
- 解耦包与流、非阻塞 I/O、精确背压建模，确保 5G VPN 高并发和高可靠性需求。

