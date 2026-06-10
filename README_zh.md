# Numa
<p align="center">
  <b>中文</b> | <a href="./README.md">English</a>
</p>

[![CI](https://github.com/razvandimescu/numa/actions/workflows/ci.yml/badge.svg)](https://github.com/razvandimescu/numa/actions)
[![crates.io](https://img.shields.io/crates/v/numa.svg)](https://crates.io/crates/numa)
[![License: MIT](https://img.shields.io/badge/license-MIT-blue.svg)](LICENSE)

**思之所向，梦之所至** — [numa.rs](https://numa.rs)


Numa是一个便携的DNS解析器。主动屏蔽广告，命名本地服务(`frontend.numa`)，自动还原覆盖任何主机名，还能封存所有外呼查询(ODoH(RFC 9230))。这对于隐私保护俩书至关重要！这一切都不需要注册任何账号以及自建路由，只需要[安装](#快速开始)即可体验。

Rust重写世界！但本软件不依赖任何现成的DNS库。提供了开箱即用的**缓存**，**广告拦截**和**本地服务域名**功能
可选从根域名服务器进行递归解析，并支持完整的 DNSSEC 信任链验证，同时还提供一个基于 DNS-over-TLS 的监听器，用于加密的客户端连接（如 iOS 私有 DNS、systemd-resolved 等）。
运行`numa relay`功能，即可变成一个公开的 ODoH 端点——目前经过筛选的 DNSCrypt 列表中仅存一个可用的中继。因此每部署一个 Numa，ODoH的生态都将得到实质性的提升！

再加上Rust祖传的代码优化，一切都压缩在约8MB的二进制文件中。

![Numa 控制面板](assets/hero-demo.gif)

## 快速开始

```bash
# macOS
brew install razvandimescu/tap/numa

# Linux
curl -fsSL https://raw.githubusercontent.com/razvandimescu/numa/main/install.sh | sh

# Arch Linux
pacman -S numa

# Windows — 从 GitHub Releases下载
# 全平台
cargo install numa

# Docker
docker run -d --name numa --network host ghcr.io/razvandimescu/numa

# Nix
nix run github:razvandimescu/numa
```

```bash
sudo numa               # 前台运行（由于使用了53端口，需要使用root权限）
```

打开控制面板: **http://numa.numa** (或 http://localhost:5380)

安装为系统服务:

| 操作系统 | 安装服务                       | 卸载                             |
| -------- | ------------------------------ | -------------------------------- |
| macOS    | `sudo numa install`            | `sudo numa uninstall`            |
| Linux    | `sudo numa install`            | `sudo numa uninstall`            |
| Windows  | `numa install` (管理员) + 重启 | `numa uninstall` (管理员) + 重启 |

在 macOS 和 Linux 上，numa 以系统服务的形式运行（通过 launchd / systemd 管理）。在 Windows 上，numa 通过注册表实现在登录时自动启动。此外，Windows 会将服务绑定到 127.0.0.2:53（因为系统内置的 Dnscache 服务占用了 127.0.0.1:53），并安装 NRPT 规则将查询路由至此地址——因此在配置 bind_addr 和 api_bind_addr 时请使用 127.0.0.2，而不是 127.0.0.1。

## 本地服务部署

为你的开发服务命名，无需再记忆端口号：

```bash
curl -X POST localhost:5380/services \
  -d '{"name":"frontend","target_port":5173}'
```

现在访问`https://frontend.numa`即可直接自动重定向到你的自定义服务！同时自动适配SSL，证书，同时支持用于 HMR（热模块替换）的 WebSocket 透传。无需 mkcert、无需 nginx、也无需手动修改老掉牙的`/etc/hosts`。

支持按路径路由（app.numa/api → :5001），通过局域网发现机制在多台机器之间共享服务。使用配置文件`numa.toml`完全控制你的numa！

## 广告屏蔽和隐私

通过 [Hagezi Pro](https://github.com/hagezi/dns-blocklists) 自动屏蔽385K+ 广告。 只要你使用网络，一切就在numa掌控之中！

我们提供三种模式：

- forward（转发模式）—— 将查询透传至系统现有 DNS。一切如常运行，唯添缓存以提速，加广告拦截以清朗。强制门户、VPN、企业 DNS，无不兼容！此选项为默认模式。

- recursive（递归模式） —— 不倚上游，不假外求，无有谁能窥得全貌。使用 `[dnssec] enabled = true`，即可开启完整信任链验证，步步为营，层层印证。

- auto（自动模式） —— 启动时探测根服务器，如果可达则使用递归模式，若被阻断则回退到加密的 DoH。

> 关于DNSSEC 可验证完整的信任链：涵盖 RRSIG 签名、DNSKEY 验证、DS 委派认证，以及 NSEC/NSEC3 的否定存在证明，请[阅读此文章](https://numa.rs/blog/posts/dnssec-from-scratch.html)

**DNS-over-TLS 监听器** (RFC 7858) — 在 853 端口接收来自 iOS Private DNS、systemd-resolved 或 stubby 等严格客户端的加密查询。提供两种模式：

- 自签名模式（默认）—— numa 会自动生成本地 CA。运行 numa install 会将该 CA 添加到以下系统的信任存储区：macOS、Linux（Debian/Ubuntu、Fedora/RHEL/SUSE、Arch）以及 Windows。在 iOS 上，可通过 numa setup-phone 生成的 .mobileconfig 配置文件进行安装。Firefox 使用独立的 NSS 存储，不跟随系统信任存储 —— 若需要在 Firefox 中为 .numa 服务启用 HTTPS，请手动将 CA 添加到 Firefox 的信任存储中。

- 自有证书模式 —— 通过配置 [dot] cert_path 和 key_path 指向使用公开受信任的证书（例如，通过 DNS-01 challenge 为指向你 numa 实例的域名从 Let's Encrypt 获取的证书）。客户端无需任何信任存储配置即可连接 —— 使用体验与 AdGuard Home 或 Cloudflare 1.1.1.1 一致。

> 两种模式下均会通告并强制执行 ALPN 协议 "dot"；若握手时 ALPN 不匹配，则视为跨协议混淆攻击并予以拒绝。

**为Phone安装服务**

```bash
numa setup-phone
```

扫描生成的二维码，然后安装提供的描述文件，再开启证书信任。
你手机的 DNS 便会通过 TLS 经由 Numa 进行解析。当然，需要在 `numa.toml` 中配置 `[mobile] enabled = true`。

## 局域网发现

在多台机器上运行 Numa，它们会通过 mDNS 自动发现彼此：:

```
机器 A (192.168.1.5)              机器 B (192.168.1.20)
┌──────────────────────┐             ┌──────────────────────┐
│ Numa                 │    mDNS     │ Numa                 │
│  - api (port 8000)   │◄───────────►│  - grafana (3000)    │
│  - frontend (5173)   │    发现     │                      │
└──────────────────────┘             └──────────────────────┘
```

在机器 B 上执行：curl http://api.numa → 请求会被代理到机器 A 的 8000 端口。通过 numa lan on 即可启用该功能。

**中心模式**: 通过设置 bind_addr = "0.0.0.0:53" 运行一个中心实例，并将其他设备的 DNS 指向该实例——这些设备无需安装任何软件即可获得广告拦截 + .numa 域名解析功能。bind_addr 也支持配置为列表，以仅绑定特定网络接口的子集。


## Docker

```bash
# 推荐 — 本地网络 (Linux)
docker run -d --name numa --network host ghcr.io/razvandimescu/numa

# 端口映射 (macOS/Windows Docker Desktop)
docker run -d --name numa -p 53:53/udp -p 53:53/tcp -p 5380:5380 ghcr.io/razvandimescu/numa
```

管理面板地址：`http://localhost:5380`。默认情况下，镜像将 API 和代理绑定到 `0.0.0.0`。可通过自定义配置文件覆盖此行为：

```bash
docker run -d --name numa --network host \
  -v /path/to/numa.toml:/root/.config/numa/numa.toml \
  ghcr.io/razvandimescu/numa
```

支持多架构： `linux/amd64` 和 `linux/arm64`.

开箱即用的 Docker Compose 方案：
- [`packaging/client/`](packaging/client/) — ODoH 客户端模式（匿名 DNS），包含 Numa 及入门级 `numa.toml` 配置文件。
- [`packaging/relay/`](packaging/relay/) — 公共 ODoH 中继，整合了 Numa、Caddy 和 ACME。

## 横向对比

|                                  | Pi-hole                | AdGuard Home | Unbound             | Numa                              |
| -------------------------------- | ---------------------- | ------------ | ------------------- | --------------------------------- |
| Local service proxy + auto TLS   | —                      | —            | —                   | `.numa` domains, HTTPS, WebSocket |
| LAN service discovery            | —                      | —            | —                   | mDNS,零配置                       |
| Developer overrides (REST API)   | —                      | —            | —                   | 自动恢复与脚本化控制              |
| Recursive resolver               | —                      | —            | 是                  | 智能调度                          |
| DNSSEC validation                | —                      | —            | 是                  | 支持 (RSA, ECDSA, Ed25519)        |
| Ad blocking                      | 是                     | 是           | 是                  | 385K+ 域名                        |
| Web admin UI                     | 完全支持               | 完全支持     | —                   | 控制面板                          |
| Encrypted upstream (DoH/DoT)     | 需要 `cloudflared`     | DoH only     | DoT only            | DoH + DoT (`tls://`)              |
| Encrypted clients (DoT listener) | 需要 `stunnel sidecar` | 是           | 是                  | 动态 (RFC 7858)                   |
| DoH server endpoint              | —                      | 是           | —                   | 是 (RFC 8484)                     |
| Request hedging                  | —                      | —            | —                   | 所有协议 (UDP, DoH, DoT)          |
| Serve-stale + prefetch           | —                      | —            | Prefetch at 90% TTL | RFC 8767, prefetch at 90% TTL     |
| Conditional forwarding           | —                      | 是           | 是                  | 是 (per-suffix rules)             |
| Portable (laptop)                | 否 (仅应用)            | 否 (仅应用)  | 仅服务端            | 单应用, macOS/Linux/Windows       |
| Community maturity               | 56K stars, 10 years    | 33K stars    | 20 years            | 非常新（确信                      |

## 性能

缓存查询仅需 0.1 毫秒——与 Unbound 和 AdGuard Home 持平。在线缆级别的缓存中，以原始字节形式存储，并支持原地 TTL 修补。请求对冲消除了 p99 延迟尖峰：冷递归 p99 为 538 毫秒，相比 Unbound 的 748 毫秒降低了 28%，标准差收紧了 4 倍。[Benchmarks →](benches/)
## 更多

- [Blog: Numa as your tailnet resolver](https://numa.rs/blog/posts/numa-tailnet-resolver.html)
- [Blog: DNS-over-TLS from Scratch in Rust](https://numa.rs/blog/posts/dot-from-scratch.html)
- [Blog: Implementing DNSSEC from Scratch in Rust](https://numa.rs/blog/posts/dnssec-from-scratch.html)
- [Blog: I Built a DNS Resolver from Scratch](https://numa.rs/blog/posts/dns-from-scratch.html)
- [Configuration reference](numa.toml) — all options documented inline
- [REST API](src/api.rs) — 27 endpoints across overrides, cache, blocking, services, diagnostics

## TODO

- [x] DNS 转发、缓存、广告拦截、开发者覆盖规则
- [x] `.numa` 本地域名 —— 自动 TLS、路径路由、WebSocket 代理
- [x] 局域网服务发现 —— mDNS、跨机器 DNS + 代理
- [x] DNS-over-HTTPS — 加密上游 + 服务端端点 (RFC 8484)
- [x] DNS-over-TLS — 加密客户端监听器 (RFC 7858) + 上游转发 (`tls://`)
- [x] 递归解析 + DNSSEC —  信任链验证, NSEC/NSEC3
- [x] 基于 SRTT 的域名服务器选择
- [x] 多转发器故障转移 —— 多个上游配合 SRTT 排序及备用池
- [x] 请求对冲 —— 并行请求挽回丢包和长尾延迟（支持所有协议）
- [x] 提供过期数据 + 预取 —— RFC 8767，在 TTL <10% 时后台刷新，以及过期时仍可提供服务
- [x] 条件转发 —— 按后缀规则实现 split-horizon DNS（Tailscale、VPN 等场景）
- [x] 缓存预热 —— 对配置的域名进行主动解析
- [x] 移动端快速配置 —— setup-phone 二维码流程、移动 API、mobileconfig 描述文件
- [ ] pkarr 集成 —— 基于 Mainline DHT 的自主主权 DNS
- [ ] 全局 `.numa` 后缀 —  基于 DHT，无需注册

## 开源许可证

MIT
