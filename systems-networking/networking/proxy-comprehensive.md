# 代理（Proxy）完全指南

> 代理是一台帮你转发网络请求的中间服务器。它最核心的价值在于：让不能直接通信的两端，通过一个中间服务器完成数据交换。

这篇指南拆成三部分读：先建立概念边界，再看系统代理、TUN 和 DNS 的机制，最后处理 Windows、WSL、Docker 里的配置和排错。

## 阅读路径

- [代理概念：方向、协议与边界](proxy-concepts.md)
- [代理机制：系统代理、TUN 与 DNS 分流](proxy-system-tun-dns.md)
- [代理实践：Windows、WSL、Docker 与排错](proxy-practice-troubleshooting.md)

## 先看地图：概念关系全景

```text
                       ┌── 正向代理（Forward Proxy）
                       │    ├─ 显式代理（客户端主动配置）
                       │    └─ 透明代理（客户端无感知）  ←── 与 TUN 模式紧密相关
                       │
代理（Proxy） ──────────┤
                       ├── 反向代理（Reverse Proxy）
                       │
                       └── 按协议分：
                            ├─ HTTP 代理（应用层，CONNECT 隧道）
                            └─ SOCKS5 代理（会话层，协议无关）

工作机制层（这些是「怎么用代理」而不是「代理是什么」）：
   系统代理 ──→ 规则模式 / 全局模式 / 直连模式
   环境变量代理（HTTP_PROXY / ALL_PROXY）
   TUN 模式 ──→ 虚拟网卡接管，近似透明代理效果
   DNS 分流 ──→ 防止 DNS 泄漏 & CDN 错误

易混淆概念（不是代理，但常被拿来比较）：
   VPN（全设备隧道）  vs  路由（IP 层路径选择）  vs  NAT（地址翻译）
```

## 常见误区

1. **「代理 = VPN」**：代理按应用粒度、VPN 按设备粒度。两者机制不同，TUN 模式是代理向 VPN 靠拢的形态，但本质仍是代理的分流转发。
2. **「SOCKS5 比 HTTP 代理快」**：快慢取决于具体实现和网络，不是协议本身决定的。SOCKS5 的优势在协议无关和 UDP 支持，不在速度。
3. **「系统代理设了，所有流量都走代理」**：系统代理是约定不是强制。游戏、终端工具、UDP 流量都可能绕开。
4. **「代理能保护隐私」**：代理隐藏 IP 但不一定加密内容。HTTP 代理对目标是明文；SOCKS5 本身不加密。只有代理协议内置加密层（如 Shadowsocks、VMess）才加密。
5. **「关掉客户端就停止代理了」**：关闭客户端后系统代理设置可能没恢复，浏览器仍尝试连到已死的代理端口 → 表现为「断网」。
6. **「正向代理和反向代理是对立的」**：同一个软件（如 Nginx）可以同时做正向和反向代理，取决于配置方向。正向/反向是**角色**不是**类别**。

---

## 小结

1. 代理的核心价值是**提供一条不同的网络路径**——不是穿透防火墙，是走一条不同的路。
2. 正向代理（含透明代理）服务于客户端，反向代理服务于服务端。区分标准：谁配置、为谁服务。
3. 透明代理的关键是「客户端无感知」——流量被网络/系统拦截重定向。TUN 模式是透明代理在操作系统级别的最常见实现。
4. HTTP 代理理解 HTTP 语义但只能代理 HTTP/HTTPS；SOCKS5 协议无关，支持任意 TCP/UDP。
5. 系统代理 = 应用层约定（不可靠），TUN 模式 = 网络层劫持（全覆盖）。
6. DNS 分流防止 CDN 错误和 DNS 泄漏；Fake IP 让 TUN 模式下的规则匹配回到域名层面。
7. 排错从底层到上层：连通性 → 端口 → 协议 → DNS → 规则 → 应用配置。

---

## 自测

1. 代理与 NAT 的本质区别是什么？为什么说代理「终结 TCP 连接」？
2. 正向代理和反向代理各自隐藏了什么？给出生活中遇到的一个具体例子。
3. 透明代理「透明」是对谁透明？它和 TUN 模式是什么关系？
4. 什么时候必须用 SOCKS5 而不是 HTTP 代理？HTTP 代理的 CONNECT 方法和 SOCKS5 转发的区别是什么？
5. 系统代理模式下 curl 不走代理，有哪几种解决办法？
6. 开启 TUN 模式后网络断开，最可能的原因是什么？怎么排查？
7. 什么是 DNS 泄漏？Fake IP 模式如何解决 IP 层面规则匹配困难的问题？
8. WSL 2 中配置代理为什么不能用 127.0.0.1？
9. 代理客户端显示已连接但浏览器打不开网页，请按排查框架列出完整检查顺序。

---

## 参考资料

- [RFC 1928 — SOCKS Protocol Version 5](https://datatracker.ietf.org/doc/html/rfc1928)
- [RFC 7230 — HTTP/1.1 Message Syntax and Routing](https://datatracker.ietf.org/doc/html/rfc7230)
- [Clash 官方文档](https://dreamacro.github.io/clash/)
- [Wikipedia: Proxy server](https://en.wikipedia.org/wiki/Proxy_server)
- [TUN/TAP 接口说明 — Linux Kernel Documentation](https://www.kernel.org/doc/Documentation/networking/tuntap.txt)
- [Docker 官方文档：Configure Docker to use a proxy server](https://docs.docker.com/network/proxy/)
- [Microsoft WSL 文档：网络](https://learn.microsoft.com/en-us/windows/wsl/networking)
