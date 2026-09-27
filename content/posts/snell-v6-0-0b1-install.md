---
title: "Snell v6.0.0b1 服务端安装实录"
date: 2025-06-12T10:00:00Z
draft: false
tags: ["snell", "proxy", "linux", "debian"]
categories: ["技术"]
---

![服务器部署](/images/remote/ext-e58d452b8212.jpg)

Snell 是 Surge 团队开发的私有代理协议，相比 Shadowsocks 在抗检测和性能上都有明显优势。本文记录一次在 Debian 服务器上安装 Snell v6.0.0b1 的完整过程，包括踩坑与修复。

异响方式：本次使用标记版本为 `v6.0.0b1` 的服务端构建。

## 下载与解压二进制

```bash
wget https://dl.nssurge.com/snell/snell-server-v6.0.0b1-linux-amd64.zip
unzip snell-server-v6.0.0b1-linux-amd64.zip -d /usr/local/bin && rm -rf snell-server-v6.0.0b1-linux-amd64.zip
chmod +x /usr/local/bin/snell-server
```

## 修复 libssl 依赖

在较新的 Debian 系统上直接运行会报错，因为系统默认不再提供 `libssl1.1`：

```bash
snell-server --version
# 报错：缺少 libssl.so.1.1
```

确认依赖关系：

```bash
readelf -d /usr/local/bin/snell-server | grep NEEDED
```

推荐的修复方式是从 Debian 快照仓库获取兼容版本：

```bash
wget https://snapshot.debian.org/archive/debian/20240311T000000Z/pool/main/o/openssl/libssl1.1_1.1.1w-0+deb11u1_amd64.deb
dpkg -i libssl1.1_1.1.1w-0+deb11u1_amd64.deb
```

> ⚠️ 注意：某些较旧的镜像源提供的 deb11u4 版本在某些新系统上安装会失败，建议使用上述 snapshot 链接中的 deb11u1 版本。

## 生成配置文件

```bash
snell-server --wizard -c /etc/snell-server.conf
```

按提示生成密钥即可，配置文件示例：

```ini
[snell-server]
listen = 0.0.0.0:43626
psk = 你的预共享密钥
obfs = off
```

## 配置 systemd 服务

创建独立用户并写入服务文件：

```bash
useradd -r -s /usr/sbin/nologin snell

cat <<'EOF' > /lib/systemd/system/snell.service
[Unit]
Description=Snell Proxy Service
After=network.target

[Service]
Type=simple
User=snell
Group=snell
LimitNOFILE=32768
ExecStart=/usr/local/bin/snell-server -c /etc/snell-server.conf
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=snell-server

[Install]
WantedBy=multi-user.target
EOF
```

加载并启动：

```bash
systemctl daemon-reload
systemctl enable snell --now
```

## 验证运行状态

```bash
systemctl status snell
ss -tlnp | grep 43626
```

若有问题可查看详细日志：

```bash
journalctl -u snell -n 100 --no-pager
```

## 客户端配置参考

在 Surge / Shadowrocket 等客户端中添加 Snell 节点时，协议版本选择 v3，填入服务器地址、端口及 PSK 即可。

---

以上就是 Snell v6.0.0b1 在 Debian 系统上的安装全过程。核心坑点在于 `libssl1.1` 的兼容性——务必使用 snapshot 源中正确的 .deb 包，否则服务无法正常启动。搭建后建议检查防火墙放行及端口占用情况。