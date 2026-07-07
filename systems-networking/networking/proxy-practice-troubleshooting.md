# 代理实践：Windows、WSL、Docker 与排错

这一篇处理实际配置和排错。重点是 Windows、WSL、Docker 这三类环境中 localhost、宿主机和容器网络的边界。

[返回代理指南入口](proxy-comprehensive.md)

## 8. Windows、WSL 与 Docker 中的代理

三个环境的难点都围绕一个核心问题：**网络隔离导致 `localhost` 指向的不是你以为的那台机器**。

### 8.1 Windows 代理配置

| 配置方式 | 说明 |
|----------|------|
| 系统代理 | 设置 → 网络和 Internet → 代理，本质写注册表 |
| 自动配置脚本 | PAC 文件 URL（JavaScript 函数定义哪些域名走代理） |
| 自动检测 | WPAD（通过 DHCP/DNS 发现 PAC 文件） |
| 环境变量 | `set HTTP_PROXY=http://127.0.0.1:7890`（CMD/PowerShell） |

代理客户端（Clash Verge、V2RayN）自动控制系统代理，开启时写入 `127.0.0.1:7890`，关闭时还原。

### 8.2 WSL 代理配置

WSL 2 有独立网络命名空间——WSL 内部的 `localhost` ≠ Windows 的 `localhost`。

**问题**：Windows 代理客户端监听 `127.0.0.1:7890`，WSL 内 `127.0.0.1:7890` 访问不到。

**解决**：WSL 2 中，`/etc/resolv.conf` 的 nameserver 指向 Windows 宿主机：

```bash
# 获取 Windows 宿主 IP
export HOST_IP=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')

# 设置代理
export HTTP_PROXY="http://${HOST_IP}:7890"
export HTTPS_PROXY="http://${HOST_IP}:7890"
export ALL_PROXY="socks5://${HOST_IP}:7890"
```

写入 `~/.bashrc` 或 `~/.zshrc` 持久化。

> [!WARNING]
> WSL 1 共享 Windows 网络栈，可以直接用 `localhost`；WSL 2 是独立虚拟网络，以上配置仅适用 WSL 2。

**如果 Windows 端开了 TUN 模式且允许局域网连接**：WSL 可以直接将流量路由到宿主机的 TUN 网卡。这是高级玩法，大多数场景设环境变量就够。

### 8.3 Docker 代理配置

Docker 的代理分三个层面：

| 层面 | 用途 | 配置方式 |
|------|------|----------|
| Docker daemon | 拉取镜像（`docker pull`） | `daemon.json` 的 `proxies` 字段或 systemd override |
| `docker build` | 构建过程中 `RUN` 的网络请求 | `--build-arg HTTP_PROXY=...` 或 Dockerfile 中 `ARG` |
| 容器运行时 | 容器内应用的网络请求 | `docker run -e HTTP_PROXY=...` 或 `docker-compose.yml` 的 `environment` |

**关键问题**：容器内访问宿主机的代理时，`localhost` 指向容器自身。

解决方法：
- **Linux Docker**：用 `host.docker.internal`（Docker 20.10+，需 `--add-host host.docker.internal:host-gateway`）或 Docker 网桥 IP（通常 `172.17.0.1`）
- **Docker Desktop（Mac/Windows）**：`host.docker.internal` 开箱即用

```json
// /etc/docker/daemon.json（Docker daemon 代理）
{
  "proxies": {
    "http-proxy": "http://host.docker.internal:7890",
    "https-proxy": "http://host.docker.internal:7890",
    "no-proxy": "localhost,127.0.0.1,*.local"
  }
}
```

---

## 9. 常见故障与排查方法

### 9.1 排查框架

按从底层（网络可达）到上层（应用配置）的顺序——这是网络排错的基本原则：

```text
1. 代理服务器是否可达？    ← ping / telnet / nc 测试连通性
2. 代理端口正确？          ← 客户端配置的端口是否匹配代理监听端口
3. 协议是否匹配？          ← HTTP 代理端口 ≠ SOCKS5 端口
4. DNS 正常？             ← nslookup / dig 检查解析，确认未被污染
5. 分流规则正确？          ← 当前访问的域名命中了预期规则吗？
6. 应用是否用了代理？      ← 系统代理设了吗？环境变量设了吗？TUN 开了吗？
7. TLS/证书？             ← 是否被中间人拦截（如企业防火墙）？
```

### 9.2 常见问题速查

#### 问题 1：代理显示「已连接」，但浏览器无法访问任何网站

最可能原因：浏览器没走代理。

排查：
```bash
# 确认代理端口在监听
netstat -an | grep 7890         # Linux/macOS
netstat -ano | findstr 7890     # Windows

# 用 curl 直连代理测试
curl -x http://127.0.0.1:7890 https://www.google.com -v
```

检查项：
- 浏览器自身代理设置（可能覆盖了系统代理）
- 系统代理地址/端口是否正确
- 代理客户端是否处于「直连模式」

#### 问题 2：部分网站能访问，部分不能

通常指向**规则或 DNS 分流出问题**。

排查：
- 看代理客户端日志：当前访问的域名命中了哪条规则？最终走的直连还是代理？
- 临时切全局模式——如果能访问，就是规则问题
- 开 DNS 分流？没开的话，国内域名可能被解析到海外 CDN

#### 问题 3：终端工具不走代理

CLI 工具不读系统代理。

```bash
export HTTP_PROXY=http://127.0.0.1:7890
export HTTPS_PROXY=http://127.0.0.1:7890
export ALL_PROXY=socks5://127.0.0.1:7890

# 验证
curl -v https://www.google.com
```

#### 问题 4：开启 TUN 模式后网络完全断开

通常是**路由环路**——流向代理服务器的流量自身也进了 TUN。

排查：
- 关 TUN 切回系统代理确认基本连通性
- `ip route`（Linux）/ `route print`（Windows）检查路由表
- 确认到代理服务器 IP 的路由**不经过** TUN 网卡

#### 问题 5：DNS 泄漏

现象：访问 `ipleak.net` 显示本地运营商 DNS。

原因：浏览器的 DNS 查询没走代理（可能走了系统 DNS 或 DoH）。

解决：
- 代理客户端开启「DNS 通过代理」
- 浏览器开启 DNS over HTTPS（DoH），设为代理友好的服务
- TUN 模式确认 DNS 请求被 TUN 截获

#### 问题 6：WSL/Docker 中无法连接代理

核心问题：`127.0.0.1` 指向的不是宿主机。

```bash
# WSL 中
cat /etc/resolv.conf | grep nameserver     # 获取宿主机 IP
nc -zv <宿主机IP> 7890                      # 测试连通

# Docker 中
docker run --rm alpine nc -zv host.docker.internal 7890
```

---

