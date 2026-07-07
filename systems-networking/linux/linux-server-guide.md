# Linux 服务器使用指南与常用命令

> 面向日常运维和开发的 Linux 服务器操作手册。从连接服务器、文件管理、权限系统到进程监控、文本处理和生产环境排错，按使用场景组织，每个命令配实际用例而非语法词典。

---

## 先看地图

```text
                     ┌── 连接服务器 ── SSH 密钥 / 免密登录 / tmux
                     │
                     ├── 用户与权限 ── useradd / chmod / chown / sudo / umask
                     │
                     ├── 文件系统   ── ls / find / rsync / df / mount / ln
                     │
Linux 服务器操作 ─────┤
                     ├── 进程管理   ── ps / top / htop / kill / nice / nohup
                     │
                     ├── 文本处理   ── grep / awk / sed / tail / jq
                     │
                     ├── 网络排查   ── ss / curl / nc / tcpdump / iptables
                     │
                     ├── 包管理     ── apt / snap / pip / 源码编译
                     │
                     ├── 系统管理   ── systemd / journalctl / cron / 环境变量
                     │
                     └── 安全加固   ── ufw / fail2ban / 审计日志
```

**阅读建议**：按你当前的任务场景跳读——刚拿到一台新服务器从 §1 开始；排查线上故障直接跳到 §5（进程）、§6（文本）、§7（网络）。

---

## 1. 连接服务器

### 1.1 SSH 基础

SSH（Secure Shell）是登录 Linux 服务器的标准方式——**加密的远程终端**。

```bash
# 密码登录
ssh user@192.168.1.100

# 指定端口（默认 22）
ssh -p 2222 user@host

# 指定私钥
ssh -i ~/.ssh/my_key user@host

# 调试连接问题（-v 越多越详细）
ssh -vvv user@host
```

**第一次连接时会提示指纹验证**——这是 SSH 的防中间人攻击机制。如果某天突然又弹指纹警告，说明服务器指纹变了（可能是重装了系统，也可能是被劫持），必须确认后再连接。

### 1.2 SSH 密钥登录（免密）

密码登录不安全（暴力破解）+ 不方便。生产环境一律用密钥。

```bash
# 本机生成密钥对（一路回车）
ssh-keygen -t ed25519 -C "your_email@example.com"

# 把公钥拷到服务器（方式一：自动）
ssh-copy-id user@host

# 方式二：手动追加
cat ~/.ssh/id_ed25519.pub | ssh user@host "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

> **`ed25519` vs `rsa`**：ed25519 是更现代的算法，安全性高、密钥短、速度快。除非兼容老旧系统，优先用 ed25519。

**服务器端安全加固**（`/etc/ssh/sshd_config`）：

```bash
# 禁止 root 直接登录
PermitRootLogin no

# 禁止密码登录（只允许密钥）
PasswordAuthentication no

# 改掉默认端口（减少扫描噪音）
Port 2222
```

改完重启 sshd：`sudo systemctl restart sshd`

### 1.3 保持会话不中断：tmux

SSH 连接断开 = 正在运行的命令被杀掉。**tmux** 解决这个问题——它创建一个独立于 SSH 连接的「会话」，断开后任务继续跑，重连后恢复现场。

```bash
tmux new -s work          # 创建名为 work 的会话
tmux ls                   # 列出所有会话
tmux attach -t work       # 重连到 work 会话

# 在 tmux 内的快捷键（Ctrl+B 是前缀）：
Ctrl+B d                  # 脱离会话（detach），后台继续跑
Ctrl+B %                  # 左右分屏
Ctrl+B "                  # 上下分屏
Ctrl+B 方向键             # 切换窗格
```

> **tmux vs screen vs nohup**：tmux 是完整终端复用器（分屏、多窗口、可重连）；nohup 只防挂断不防丢失；screen 是 tmux 的前辈，tmux 体验更好。日常选 tmux，一次性长任务用 `nohup command &`。

---

## 2. 用户与权限

Linux 的权限系统是服务器安全的第一道防线。核心概念：**一切皆文件**，权限管的是「谁」能对文件「做什么」。

### 2.1 用户管理

```bash
# 查看
whoami                    # 我是谁
id                        # 我的 UID、GID、所属组
who                       # 当前登录的所有用户
last                      # 历史登录记录

# 创建用户
sudo useradd -m -s /bin/bash username    # -m 创建家目录, -s 指定 shell
sudo passwd username                     # 设置密码

# 修改用户
sudo usermod -aG docker username         # 追加到 docker 组（-a 是追加，不加 -a 会覆盖）
sudo usermod -s /bin/zsh username        # 改默认 shell

# 删除
sudo userdel -r username                 # -r 同时删家目录

# 临时切换
su - username              # 切换到 username，- 表示加载其完整环境
sudo -i                    # 切换到 root（模拟登录）
sudo -u username command   # 以 username 身份执行一条命令
```

> [!WARNING]
> `usermod -G docker username` 不加 `-a` 会把用户从**所有其他组**中移除，只保留 docker 组。记住：加组用 `-aG`。

### 2.2 文件权限

```
-rw-r--r--  1 sycamore dev  4096 Jun 15 10:00 file.txt
 │├─┤├─┤├─┤
 │ │  │  └── 其他人权限 (r--)
 │ │  └───── 组权限 (r--)
 │ └──────── 所有者权限 (rw-)
 └────────── 文件类型：- 文件 / d 目录 / l 软链接
```

每个权限位：`r`=读(4), `w`=写(2), `x`=执行(1)。三位一组 = 八进制值。

```bash
# 符号模式
chmod u+x script.sh        # 所有者加执行
chmod g-w file.txt         # 组去掉写
chmod o= file.txt          # 其他人清空所有权限
chmod u=rwx,g=rx,o= script.sh

# 数字模式（常用组合）
chmod 755 script.sh        # rwxr-xr-x（所有者可读写执行，其余只读执行）
chmod 644 file.txt         # rw-r--r--（所有人可读，所有者可写）
chmod 600 ~/.ssh/id_ed25519 # rw-------（仅所有者可读写，私钥应该这样）
chmod 700 ~/.ssh            # rwx------（.ssh 目录应该只有所有者可进入）

# 递归改目录
chmod -R 755 /var/www       # 目录下所有文件和子目录
find . -type d -exec chmod 755 {} \;  # 只改目录
find . -type f -exec chmod 644 {} \;  # 只改文件

# 所有者
chown user:group file.txt
chown -R www-data:www-data /var/www   # 递归改
```

> `chmod 777` 是危险操作——任何人可读写执行。永远不要在生产环境用 `777`。如果需要多用户协作，用**组权限 + umask** 而不是放开权限。

### 2.3 默认权限：umask

`umask` 决定了新创建文件的默认权限——它是一张「权限掩码」，从最大权限中减去。

```bash
umask         # 查看当前值，通常是 0022
umask 027     # 设置：所有者不变，组去掉写，其他全关

# 计算逻辑：最大权限 - umask = 默认权限
# 文件：666 - 022 = 644 (rw-r--r--)
# 目录：777 - 022 = 755 (rwxr-xr-x)
```

### 2.4 sudo 与 root

```bash
sudo command           # 以 root 执行单条命令
sudo !!                # 以 root 执行上一条命令（最常用技巧）
sudo -i                # 进入 root shell

# 查看 sudo 授权
sudo -l                # 当前用户能 sudo 什么

# visudo 编辑 /etc/sudoers（不要直接 vim 编辑！会因语法错误锁死 sudo）
sudo visudo

# 免密 sudo（生产环境谨慎使用）
# username ALL=(ALL) NOPASSWD: ALL
```

> **原则**：永远不要以 root 身份做日常操作。`sudo` 提供审计日志（每条命令记录在 `/var/log/auth.log`），直接 root 没有。

---

## 3. 文件系统

### 3.1 目录结构与导航

```text
/          ← 根目录，一切从这里开始
├── /bin   ← 基本命令（ls, cp, mv）
├── /etc   ← 配置文件（nginx, ssh, systemd）
├── /var   ← 变化数据（日志 /var/log, 网站 /var/www）
├── /home  ← 用户家目录
├── /tmp   ← 临时文件（重启清空）
├── /opt   ← 第三方软件（手动安装的大包）
├── /usr   ← 用户程序（/usr/bin, /usr/lib, /usr/local）
├── /proc  ← 进程和内核信息（虚拟文件系统）
└── /dev   ← 设备文件
```

```bash
pwd                    # 当前目录
cd -                   # 回到上一个目录
cd ~                   # 回到家目录
pushd /path; popd      # 目录栈（pushd 压入，popd 弹出）
```

### 3.2 文件浏览

```bash
# 列出文件
ls -la                 # 详情 + 隐藏文件
ls -lh                 # 人类可读的文件大小（1K, 234M, 2G）
ls -ltr                # 按时间排序，最新的在最后
tree -L 2              # 树状显示，深度 2 层

# 查看内容
cat file               # 一次输出全部
less file              # 分页浏览（/搜索 n下一个 N上一个 q退出）
head -20 file          # 前 20 行
tail -f file           # 跟踪末尾（实时看日志）
tail -n 50 file        # 最后 50 行

# 文件信息
file unknown.bin       # 判断文件类型
stat file              # 详细元信息（大小、权限、时间戳）
wc -l file             # 行数
du -sh /path           # 目录总大小
du -h --max-depth=1 /  # 一级子目录各自大小
```

### 3.3 文件操作

```bash
# 复制
cp source dest
cp -r dir1 dir2        # 递归复制目录
cp -p file1 file2      # 保留权限和时间戳

# 移动/重命名
mv old new
mv file /target/dir/

# 删除（危险操作，没有回收站！）
rm file
rm -r dir              # 递归删目录
rm -rf dir             # 强制递归删——极度危险，确认两遍再按回车

# 链接
ln -s /actual/path link_name     # 软链接（符号链接）——类似快捷方式
ln /actual/path link_name        # 硬链接——同一个 inode 的两个入口

# 查找文件
find . -name "*.log"                         # 按文件名
find . -type f -size +100M                    # 大于 100MB 的文件
find . -mtime -7                              # 最近 7 天修改的
find . -name "*.log" -mtime +30 -delete       # 删 30 天前的日志
find . -type f -exec grep -l "ERROR" {} \;   # 查找含 ERROR 的文件
```

### 3.4 文件传输

```bash
# scp（基于 SSH，小文件）
scp file.txt user@host:/remote/path/          # 上传
scp user@host:/remote/file.txt ./             # 下载
scp -r dir/ user@host:/remote/path/           # 递归传目录

# rsync（增量同步，大文件/目录首选）
rsync -avz /local/dir/ user@host:/remote/dir/  # 同步（保留权限、压缩传输）
rsync -avz --delete /local/ user@host:/remote/ # 镜像同步（删掉目标多余文件）
rsync -avz --progress /large/file user@host:/path/ # 显示进度
```

> **scp vs rsync**：小文件传一两次用 scp，目录同步/大量文件/断点续传用 rsync。rsync 只传输差异部分，效率远高于 scp。

### 3.5 磁盘管理

```bash
df -h                  # 分区使用情况（人类可读）
df -i                  # inode 使用情况（inode 耗尽也会报磁盘满）
du -sh ~               # 家目录总大小
du -sh * | sort -rh | head -10  # 当前目录下最大的 10 个文件/目录
ncdu                   # 交互式磁盘使用分析（需安装）

# 挂载
lsblk                  # 列出所有块设备
mount /dev/sdb1 /mnt/data
umount /mnt/data

# 强制卸载（设备忙时）
fuser -m /mnt/data     # 看谁在用
lsof /mnt/data         # 同上
umount -l /mnt/data    # 懒卸载（程序释放后再卸）
```

---

## 4. 文本处理：命令行核心

> Linux 哲学：每个工具只做一件事并做好，通过**管道 `|`** 组合。文本处理是最体现这种哲学的场景。

### 4.1 grep：搜索文本

```bash
grep "ERROR" app.log                       # 搜索字符串
grep -i "error" app.log                    # 忽略大小写
grep -r "TODO" ./src/                      # 递归搜索目录
grep -v "DEBUG" app.log                    # 排除匹配行（反向）
grep -c "ERROR" app.log                    # 计数
grep -B 3 -A 5 "ERROR" app.log            # 匹配行前 3 行、后 5 行的上下文
grep -E "(ERROR|FATAL)" app.log            # 正则（扩展模式）
grep -l "TODO" *.py                        # 只列文件名
grep "pattern" huge_file.log | head -20    # 早早截断，不扫全文件
```

### 4.2 sed：流编辑器（批量替换/删除/提取）

```bash
# 替换
sed 's/old/new/' file          # 每行第一个
sed 's/old/new/g' file         # 全局替换
sed -i 's/old/new/g' file      # 就地修改（-i 会改原文件！）
sed -i.bak 's/old/new/g' file  # 改前先备份为 .bak

# 删除
sed '/^$/d' file               # 删除空行
sed '5,10d' file               # 删除第 5-10 行

# 提取
sed -n '5,10p' file            # 只输出第 5-10 行
sed -n '/ERROR/p' app.log      # 只输出含 ERROR 的行

# 实战
sed -i 's/\r$//' script.sh     # 修复 Windows 换行符
sed -i 's/^#Port 22/Port 2222/' /etc/ssh/sshd_config  # 改配置
```

### 4.3 awk：结构化文本处理

awk 擅长处理**按列组织的文本**（日志、CSV、命令输出）。

```bash
# 基本结构：awk '条件 {动作}' 文件

# 打印第 2 列和第 5 列
awk '{print $2, $5}' data.txt

# 按条件过滤
awk '$3 > 100 {print $1, $3}' data.txt       # 第 3 列 > 100
awk '/ERROR/ {print $1, $2, $NF}' app.log    # 含 ERROR 的行，输出第 1、2、最后一列

# 实战：统计日志中各状态码出现次数
awk '{print $9}' access.log | sort | uniq -c | sort -rn

# 实战：计算某列的总和/平均值
awk '{sum += $3} END {print sum}' data.txt
awk '{sum += $3; n++} END {print sum/n}' data.txt

# 实战：按分隔符解析
awk -F: '{print $1, $3}' /etc/passwd          # -F 指定分隔符
awk -F, '{print $2}' data.csv
```

> **grep → sed → awk 的分工**：grep 负责「找到行」，sed 负责「编辑行」，awk 负责「解析结构化文本并计算」。一个典型管道：`grep "ERROR" app.log | awk '{print $1, $5}' | sort | uniq -c`

### 4.4 管道与其他文本工具

```bash
# sort
sort file                  # 排序
sort -n                    # 数字排序
sort -rn                   # 倒序数字
sort -t, -k2 file.csv      # 按逗号分隔的第 2 列排序

# uniq
uniq                       # 去重（仅相邻行，通常先 sort）
sort file | uniq           # 全文件去重
sort file | uniq -c        # 去重并计数
sort file | uniq -d        # 只显示重复项

# cut
cut -d: -f1,3 /etc/passwd  # 按冒号分割，取第 1、3 列
cut -c1-10 file            # 每行取第 1-10 个字符

# xargs：把标准输入变成命令参数
find . -name "*.log" | xargs rm         # 删所有 .log 文件
grep -l "TODO" *.py | xargs sed -i 's/TODO/FIXME/g'
find . -name "*.log" -print0 | xargs -0 rm  # 安全版（处理文件名含空格）

# tee：同时输出到文件和屏幕
make install 2>&1 | tee install.log     # 看输出的同时保存日志

# jq：JSON 处理（需安装）
curl -s api.example.com/data | jq '.items[].name'
jq '.data[] | select(.status=="active") | {name, email}' data.json
```

### 4.5 输入输出重定向

```bash
command > file          # 标准输出写到文件（覆盖）
command >> file         # 标准输出追加到文件
command 2> file         # 标准错误写到文件
command 2>&1            # 标准错误合并到标准输出
command &> file         # 标准输出 + 标准错误都写到文件
command < file          # 从文件读取输入
command <<EOF           # Here Document：多行输入
多行内容
EOF
```

---

## 5. 进程管理

### 5.1 查看进程

```bash
ps aux                    # 所有进程
ps aux | grep nginx       # 找特定进程
ps -ef                    # 另一种格式

pgrep nginx               # 直接拿 PID

# 进程树（父子关系）
pstree -p
ps auxf

# 实时监控
top                       # 交互式（按 P 按 CPU 排, M 按内存排, q 退出）
htop                      # top 的更好看版（需安装，支持鼠标和颜色）
```

### 5.2 控制进程

```bash
# 后台运行
command &                     # 放到后台（仍随终端退出而死）
nohup command &               # 忽略 SIGHUP，终端退出后继续跑
nohup command > output.log 2>&1 &  # 输出重定向

# 查看后台任务
jobs                          # 当前终端会话的任务
bg %1                         # 把挂起的任务 1 放后台继续
fg %1                         # 把后台任务 1 拿到前台

# 终止进程
kill PID                      # 友好终止（SIGTERM，给程序清理资源的机会）
kill -9 PID                   # 强制杀（SIGKILL，立即终止，不清理——最后手段）
kill -15 PID                  # 同 kill PID
killall nginx                 # 按名字终止所有
pkill -f "python app.py"      # 模糊匹配

# 优先级
nice -n 10 command            # 以较低优先级启动
renice -n 5 -p PID            # 调整运行中进程的优先级（-20 最高, 19 最低）
```

### 5.3 进程信息深度查看

```bash
# /proc 文件系统：一切进程信息都在这里
ls /proc/PID/                 # 进程的完整信息目录
cat /proc/PID/cmdline         # 启动命令
cat /proc/PID/environ         # 环境变量（用 \0 分隔）
cat /proc/PID/limits          # 资源限制
cat /proc/PID/status          # 状态摘要
ls -l /proc/PID/fd/           # 打开的文件描述符（排查 fd 泄漏）

# lsof：列出打开的文件
lsof -p PID                   # 某个进程打开了哪些文件
lsof -i :80                   # 谁在监听 80 端口
lsof /var/log/syslog          # 谁在写这个文件

# strace：跟踪系统调用（调试神器）
strace -p PID                 # 附着到运行中的进程
strace -e trace=network command # 只看网络相关调用
strace -c command             # 统计各类系统调用耗时
```

---

## 6. 网络工具

```bash
# 连通性测试
ping -c 4 host               # 发 4 个包（-c 限制次数，否则无限 ping）
traceroute host              # 看数据包经过了哪些路由器
mtr host                     # ping + traceroute 合体（需安装）

# 端口与连接
ss -tlnp                     # TCP 监听端口（-t tcp, -l 监听, -n 数字, -p 进程）
ss -tunap                    # 所有 TCP/UDP 连接
ss -s                        # 连接统计

# curl：HTTP 客户端
curl -I https://example.com                    # 只看响应头
curl -v https://example.com                    # 详细过程（含 TLS 握手）
curl -X POST -d '{"key":"val"}' -H 'Content-Type: application/json' https://api.example.com
curl -o file.zip https://example.com/file.zip  # 下载文件
curl -x http://127.0.0.1:7890 https://google.com  # 通过代理

# nc（netcat）：TCP/UDP 瑞士军刀
nc -zv host 80                # 测试端口是否开放
nc -l 8080                    # 监听 8080 端口（简单测试服务器）
echo "test" | nc host 8080    # 发送数据
nc -U /var/run/docker.sock    # 连接 Unix socket

# DNS
dig example.com               # 详细 DNS 查询
dig +short example.com        # 只输出结果
dig @8.8.8.8 example.com      # 指定 DNS 服务器
nslookup example.com          # 简单 DNS 查询

# 抓包
tcpdump -i eth0 port 80       # 抓 eth0 上 80 端口的流量
tcpdump -i any host 10.0.0.1  # 抓与指定 IP 的通信
tcpdump -i eth0 -w dump.pcap  # 保存到文件（用 Wireshark 分析）
```

---

## 7. 包管理

### 7.1 apt（Debian/Ubuntu）

```bash
sudo apt update                     # 更新包索引（不会升级任何包）
sudo apt upgrade                    # 升级所有可升级的包
sudo apt full-upgrade               # 更彻底的升级（可删冲突包）
sudo apt install nginx              # 安装
sudo apt remove nginx               # 卸载（保留配置）
sudo apt purge nginx                # 彻底卸载（含配置）
sudo apt autoremove                 # 清理不再需要的依赖
apt search keyword                  # 搜索包
apt show nginx                      # 查看包详情
apt list --installed                # 已安装列表
apt list --upgradable               # 可升级列表
```

### 7.2 源码编译安装

```bash
# 典型流程
git clone https://github.com/xxx/project.git
cd project
./configure --prefix=/usr/local/myapp   # 检测环境、设置安装路径
make -j$(nproc)                          # 并行编译
sudo make install                        # 安装
```

### 7.3 其他包管理器

```bash
# snap（Ubuntu 自带，沙箱化打包）
snap install code --classic             # --classic 给更多系统权限

# pip（Python）
pip install package
pip install -r requirements.txt

# npm（Node.js）
npm install -g package

# cargo（Rust）
cargo install package
```

---

## 8. 系统管理

### 8.1 systemd：服务管理

```bash
# 生命周期
sudo systemctl start nginx
sudo systemctl stop nginx
sudo systemctl restart nginx
sudo systemctl reload nginx            # 不重启进程，重载配置
sudo systemctl enable nginx            # 开机自启
sudo systemctl disable nginx           # 取消自启
sudo systemctl status nginx            # 状态（含最近日志）

# 查看
systemctl list-units --type=service    # 所有服务
systemctl list-units --type=service --state=running
systemctl list-unit-files              # 所有可自启单元

# 日志
journalctl -u nginx                    # nginx 的日志
journalctl -u nginx -f                 # 跟踪
journalctl -u nginx --since "1 hour ago"
journalctl -u nginx --since "2026-06-15" --until "2026-06-15 12:00"
```

### 8.2 编写 systemd 服务文件

`/etc/systemd/system/myapp.service`：

```ini
[Unit]
Description=My Application
After=network.target

[Service]
Type=simple
User=appuser
WorkingDirectory=/opt/myapp
ExecStart=/opt/myapp/bin/start.sh
Restart=on-failure
RestartSec=5
Environment="APP_ENV=production"
EnvironmentFile=-/etc/myapp/env

[Install]
WantedBy=multi-user.target
```

创建后：

```bash
sudo systemctl daemon-reload       # 重载 systemd 配置
sudo systemctl enable --now myapp  # 启用并立即启动
```

### 8.3 定时任务：cron

```bash
crontab -e                     # 编辑当前用户的定时任务
crontab -l                     # 列出当前用户的定时任务

# 格式：分 时 日 月 周 命令
# 0 2 * * * /opt/backup.sh    ← 每天凌晨 2 点执行
# */5 * * * * /opt/health.sh  ← 每 5 分钟
# 0 9 * * 1-5 /opt/report.sh  ← 工作日早上 9 点

# 查看 cron 日志
grep CRON /var/log/syslog
```

> cron 的环境变量极少——脚本里要用**绝对路径**，或者自己在脚本开头 `source ~/.bashrc`。

### 8.4 环境变量

```bash
export VAR=value          # 设置（当前 shell 及子进程）
echo $VAR                 # 查看
printenv                  # 列出所有环境变量
env                       # 同上

# 持久化位置
~/.bashrc                 # 交互式 shell 每次启动加载
~/.profile                # 登录 shell 加载
/etc/environment          # 系统级（所有用户）
/etc/profile.d/           # 脚本片段，登录时加载

# 给 systemd 服务设环境变量的方式见 §8.2（Environment / EnvironmentFile）
```

---

## 9. 安全加固

```bash
# UFW 防火墙（Ubuntu）
sudo ufw enable
sudo ufw allow 22/tcp           # 允许 SSH
sudo ufw allow 80,443/tcp       # 允许 HTTP/HTTPS
sudo ufw allow from 10.0.0.0/8  # 允许内网全部访问
sudo ufw status verbose

# fail2ban：防暴力破解
sudo apt install fail2ban
sudo systemctl enable --now fail2ban
sudo fail2ban-client status sshd   # 查看 SSH 封禁状态

# 审计
sudo ausearch -m avc              # SELinux 审计日志
sudo aureport -au                 # 认证事件报告

# 检查谁在登录
last -20                          # 最近 20 条登录
lastb                             # 失败登录尝试
w                                 # 当前登录用户及正在做什么
```

---

## 10. 快速参考表

### 10.1 按场景速查

| 场景 | 命令 |
|------|------|
| 磁盘满了找大文件 | `du -sh * \| sort -rh \| head` |
| 查哪个进程占用端口 | `ss -tlnp \| grep :80` 或 `lsof -i :80` |
| 内存占用前 10 | `ps aux --sort=-%mem \| head -11` |
| CPU 占用前 10 | `ps aux --sort=-%cpu \| head -11` |
| 实时跟踪日志 | `tail -f /var/log/syslog` |
| 查看历史命令 | `history \| grep keyword` 或 Ctrl+R |
| 上次关机时间 | `last -x \| grep shutdown` |
| 系统已运行多久 | `uptime` |
| 内核版本 | `uname -a` |
| 发行版信息 | `lsb_release -a` 或 `cat /etc/os-release` |
| 所有硬件信息 | `lshw -short` |
| 内存信息 | `free -h` |
| CPU 信息 | `lscpu` |
| 块设备列表 | `lsblk` |

### 10.2 快捷键

| 快捷键 | 作用 |
|--------|------|
| Ctrl+C | 终止前台进程 |
| Ctrl+Z | 挂起前台进程 |
| Ctrl+D | EOF（退出 shell） |
| Ctrl+R | 搜索历史命令 |
| Ctrl+L | 清屏 |
| Ctrl+A / Ctrl+E | 行首 / 行尾 |
| Ctrl+W | 删除前一个词 |
| Ctrl+U | 删除光标前全部 |
| !! | 重复上条命令 |
| !$ | 上条命令的最后一个参数 |

---

## 常见误区

1. **「`rm -rf /` 会删干净」**：现代 Linux 有 `--preserve-root` 保护，裸 `rm -rf /` 不会执行。但 `rm -rf /*` 仍然能干掉一切。
2. **「`kill -9` 是正常的关闭方式」**：`-9`（SIGKILL）是最后手段——它直接杀进程不给清理机会。应先用 `kill`（SIGTERM）、`kill -1`（SIGHUP 重载）、`kill -2`（SIGINT，等同 Ctrl+C）。
3. **「关闭终端后台任务还在跑」**：默认 `command &` 仍随终端退出而死。用 `nohup`、`tmux` 或 systemd service 才能真正脱离终端。
4. **「`chmod 777` 解决了权限问题」**：制造了更大的安全问题。用组权限或 ACL 代替。
5. **「`sudo apt update` 升级了我的系统」**：`update` 只刷新包索引，`upgrade` 才是真正升级。
6. **「`.bashrc` 和 `.profile` 一样」**：`.bashrc` 每次开交互式 shell 都执行；`.profile` 仅在登录 shell 执行。环境变量放 `.profile`（登录一次就够了），别名和函数放 `.bashrc`。

---

## 小结

1. SSH 密钥登录是连接服务器的标准方式，生产环境**禁用密码登录和 root 直接登录**。
2. Linux 权限三条线：所有者/组/其他人 × 读/写/执行。`chmod 644` 文件、`755` 目录是安全默认值。
3. 文本处理三板斧：`grep`（找到）、`sed`（编辑）、`awk`（解析计算），通过管道组合。
4. 进程后台运行三方案：`nohup`（一次性）、`tmux`（交互长任务）、`systemd service`（生产环境服务）。
5. systemd 管服务生命周期、cron 管定时任务、`journalctl` 管日志——三者覆盖日常运维。
6. 排查问题从外到内：网络连通（ping/ss/curl）→ 进程状态（ps/top）→ 日志（tail/journalctl）→ 系统调用（strace）。

---

## 自测

1. 如何配置 SSH 免密登录？服务端需要改哪两个配置来提高安全性？
2. 一个文件的权限是 `-rw-r-----`，所有者、组、其他人各有什么权限？用 `chmod` 的数字模式怎么写？
3. 列出查找并删除 30 天前的 `.log` 文件的命令。
4. `scp` 和 `rsync` 各适用于什么场景？`rsync` 比 `scp` 快的原因是什么？
5. 用一条管道命令统计 `/var/log/syslog` 中 `ERROR` 出现的次数。
6. `nohup`、`tmux`、`systemd service` 分别用于什么场景？它们的本质区别是什么？
7. 你登录服务器发现磁盘满了，列出你的排查步骤和关键命令。
8. 如何查看某个进程打开了哪些文件？如何查看谁在监听 80 端口？
9. `kill`、`kill -9`、`kill -1` 的区别是什么？为什么不应该优先用 `-9`？
10. 写一个 systemd service 文件，让 `/opt/app/start.sh` 以 `appuser` 用户身份在开机时启动，崩溃后 5 秒自动重启。
