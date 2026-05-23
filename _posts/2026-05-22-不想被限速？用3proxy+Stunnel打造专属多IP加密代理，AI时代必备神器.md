---
title: 2026-05-22-不想被限速？用3proxy+Stunnel打造专属多IP加密代理，AI时代必备神器
date: 2026-05-22 17:06:23 +/-TTTT
categories: Notes
tags: reading
---
在多 IP 站群服务器或跨境网络优化场景中，**3proxy（轻量级多协议代理）** 与 **Stunnel（TLS 加密隧道）** 的组合是业界公认的黄金搭档。3proxy 负责高效调度与限速，Stunnel 负责将明文流量伪装为安全的 TLS 流量。

本文将带你快速完成这一架构的落地、客户端验证，并提供**百级 IP 自动限速脚本**与**并发连接数控制（connlim）**的完美解决方案。

---

## 💡 核心架构原理
```plain
[ 客户端 (curl/浏览器) ] --(TLS 加密)--> [ Stunnel (解密) ] --(本地明文)--> [ 3proxy (限速/并发/代理) ] --> [ 目标网站 ]
```

---

## 🛠️ 核心配置文件
## 1. Stunnel 配置 (`<font style="background-color:#f4f6f7;">/etc/stunnel/stunnel.conf</font>`)
作为前端接收端，负责解密客户端发来的 TLS 流量，并转发给后端的 3proxy。

```plain
pid = /var/run/stunnel.pid
cert = /etc/stunnel/stunnel.pem
key = /etc/stunnel/stunnel.key
client = no
sslVersion = TLSv1.3

[3proxy-tls]
accept = 0.0.0.0:443
connect = 127.0.0.1:1080
```

## 2. 3proxy 核心配置 (`<font style="background-color:#f4f6f7;">/etc/3proxy/3proxy.cfg</font>`)
⚠️ **避坑核心机制（至关重要）**：

1. **规则顺序**：3proxy 的规则链由上至下执行。**必须先写限速（bandlim）和并发（connlim）规则，最后写服务启动命令**（如 `<font style="background-color:#f4f6f7;">socks</font>`），否则限制直接失效。
2. **独立池机制**：3proxy 的每一行规则是一个**独立共享池**。若要实现多 IP 或多用户各自独享带宽/并发，**必须每个对象独占一行配置**。

## 方案 A：针对不同「客户端真实来源 IP」限速与并发控制
适用于 3proxy 能直接获取客户端真实外网 IP 的直连场景。

```plain
daemon
pidfile /var/run/3proxy.pid
nserver 8.8.8.8
nscache 65536
auth none

# 清理历史规则，确保不受全局策略干扰
flush

# ==================== 核心：多 IP 独立控制区 ====================
# 【客户端 1】 192.168.1.100 —— 独享下载 4M、上传 2M、最大 50 并发
bandlimin  4194304 * 192.168.1.100 * * *
bandlimout 2097152 * 192.168.1.100 * * *
connlim    50      * 192.168.1.100 * * *

# 【客户端 2】 192.168.1.200 —— 独享下载 8M、上传 4M、最大 100 并发
bandlimin  8388608 * 192.168.1.200 * * *
bandlimout 4194304 * 192.168.1.200 * * *
connlim    100     * 192.168.1.200 * * *
# ================================================================

# 启动服务（严格位于规则下方）
socks -p1080
```

## 方案 B：Stunnel 隧道环境下的「多用户独立限制」
由于流量经过 Stunnel 解密中转后，3proxy 识别到的来源 IP 均会变成本地回环 `<font style="background-color:#f4f6f7;">127.0.0.1</font>`。此时**根据来源 IP 限制会失效**，最佳替代方案是**针对不同用户分配独立带宽与并发数**：

```plain
daemon
pidfile /var/run/3proxy.pid
nserver 8.8.8.8
nscache 65536

# 启用强认证并创建用户
auth strong
users user_a:CL:pass_a user_b:CL:pass_b
flush

# ==================== 核心：多用户独立控制区 ====================
# 【用户 A】 独享下载 4M / 上传 2M / 最大 50 并发
bandlimin  4194304 user_a * * * *
bandlimout 2097152 user_a * * * *
connlim    50      user_a * * * *

# 【用户 B】 独享下载 8M / 上传 4M / 最大 100 并发
bandlimin  8388608 user_b * * * *
bandlimout 4194304 user_b * * * *
connlim    100     user_b * * * *
# ================================================================

socks -p1080
```

---

## 🚀 高级进阶：自动化生成百级 IP 限制配置
当拥有成百上千个 IP 需要独立限速时，手动编写配置文件极易出错。可以使用以下 Shell 脚本自动生成配置片段，并自动将 Mbps 转换为 3proxy 所需的 bps 单位。

## 1. 自动化生成脚本 (`<font style="background-color:#f4f6f7;">generate_rules.sh</font>`)
```bash
#!/bin/bash

# 定义输出文件路径
OUTPUT_FILE="/etc/3proxy/rules_generated.cfg"
# 清空旧文件
> "$OUTPUT_FILE"

# 限速参数设置 (单位: Mbps)
DOWNLOAD_MBPS=4
UPLOAD_MBPS=2
MAX_CONN=50

# 自动计算 3proxy 对应的 bps 速率值 (Mbps * 1024 * 1024)
BPS_IN=$(( DOWNLOAD_MBPS * 1024 * 1024 ))
BPS_OUT=$(( UPLOAD_MBPS * 1024 * 1024 ))

echo "# --- 自动生成的百级 IP 限制规则 ---" >> "$OUTPUT_FILE"

# 循环生成 192.168.1.10 到 192.168.1.250 的独立规则
for i in {10..250}; do
    IP="192.168.1.$i"
    echo "# 限制 IP: $IP" >> "$OUTPUT_FILE"
    echo "bandlimin  $BPS_IN * $IP * * *" >> "$OUTPUT_FILE"
    echo "bandlimout $BPS_OUT * $IP * * *" >> "$OUTPUT_FILE"
    echo "connlim    $MAX_CONN * $IP * * *" >> "$OUTPUT_FILE"
done

echo "规则生成成功，共计导出 $(expr $i - 9) 个 IP 的控制策略到 $OUTPUT_FILE"
```

## 2. 在主配置中加载
运行脚本生成规则文件后，只需在主配置文件 `<font style="background-color:#f4f6f7;">3proxy.cfg</font>` 中通过 `<font style="background-color:#f4f6f7;">include</font>` 引入即可：

```plain
# ... [基础环境配置与 auth none 略] ...
flush

# 引入自动化生成的百级 IP 控制规则
include /etc/3proxy/rules_generated.cfg

# 最后启动服务
socks -p1080
```

---

## 🧪 客户端 `<font style="background-color:#f4f6f7;">curl</font>` 加密与限制验证
部署完成后，使用 `<font style="background-color:#f4f6f7;">curl</font>` 穿透安全隧道进行代理测速与并发可用性验证：

## 1. 验证安全 Socks5 代理（推荐：防止 DNS 泄露）
```bash
curl -v --socks5h-hostname https://user_a:pass_a@服务器IP:443 https://ipify.org
```

## 2. 验证 HTTPS 代理
若 3proxy 运行在 HTTP 代理模式下（即配置文件中为 `<font style="background-color:#f4f6f7;">proxy -p1080</font>`），则使用：

```bash
curl -v -x https://user_a:pass_a@服务器IP:443 https://ipify.org
```

---

## 🔍 限速与并发控制失效排查清单
如果配置后发现限制未生效，或遭遇“多 IP 互相抢占”的连带干扰，请严格对照以下四点检查：

1. **检查上下顺序**：服务启动命令（`<font style="background-color:#f4f6f7;">socks</font>`/`<font style="background-color:#f4f6f7;">proxy</font>`）是否错误地写在了 `<font style="background-color:#f4f6f7;">bandlimin</font>`/`<font style="background-color:#f4f6f7;">connlim</font>` 的上方？
2. **确认 IP 穿透**：在前置 Stunnel 架构下，是否错误地使用了“来源 IP 限制”导致规则全部失效？（若有隧道中转，请务必参考方案 B 改用用户限速）。
3. **缺少规则刷新**：配置中是否遗漏了 `<font style="background-color:#f4f6f7;">flush</font>` 指令，导致当前精细规则被前置的全局规则强行覆盖？
4. **速率单位错误**：3proxy 的带宽单位是 **bps（比特/秒）**而非字节或 Kbps，目标 Mbps 必须乘以 `<font style="background-color:#f4f6f7;">1,048,576</font>`（推荐使用上文脚本自动计算）。

---

通过引入 `<font style="background-color:#f4f6f7;">connlim</font>` 以及自动化配置脚本，这套 3proxy + Stunnel 的方案已经具备了在海量 IP 生产环境中提供工业级网络隔离和控流的能力。

---

如果您的业务对高可用有更极致的要求，未来还可以考虑引入 `**<font style="background-color:#f4f6f7;">allow</font>**`** 和 **`**<font style="background-color:#f4f6f7;">deny</font>**`** 规则进行多级 ACL 访问控制**，随时联系我继续扩展您的架构方案！

