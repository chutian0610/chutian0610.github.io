# Docker 容器访问 IPv6 地址：踩坑与解决方案


最近搬家，家里的 Self-Host 服务由于限制无法通过公网 IPv4 访问，就改用 IPv6 + DDNS 的方式提供公网域名访问。

切换完成后，立即收到云服务器上 Uptime Kuma 的告警，显示无法连接 IPv6 地址。排查发现 Docker 容器默认不支持 IPv6。

<!--more-->

## 为什么默认不支持？

Docker 出于安全考虑，默认关闭 IPv6。这样做是合理的——很多生产环境并不需要 IPv6，而且开启后可能会与现有的网络规则产生意外冲突。但如果你需要容器访问 IPv6 地址，就必须手动启用。

## 检查宿主机环境

### 确认 IPv6 已启用

```bash
# 确保 IPv6 未被禁用 (输出应为 0)
cat /proc/sys/net/ipv6/conf/all/disable_ipv6

# 查看宿主机 IPv6 地址 (重点关注非 fe80 开头的全球单播地址)
ip -6 addr show

# 测试宿主机能否访问外网 IPv6
ping6 -c 4 2400:3200::1
```

### 启用 IPv6 转发

若需容器访问外网，需要开启 IPv6 包转发。这是 Docker 容器能通过宿主机路由到外网的必要条件。

临时生效（重启后失效）：

```bash
sudo sysctl -w net.ipv6.conf.all.forwarding=1
sudo sysctl -w net.ipv6.conf.default.forwarding=1
```

永久生效，编辑 `/etc/sysctl.conf` 或 `/etc/sysctl.d/99-sysctl.conf`：

```conf
net.ipv6.conf.all.forwarding=1
net.ipv6.conf.default.forwarding=1
```

执行 `sudo sysctl -p` 使配置生效。

## 配置 Docker 守护进程

编辑 Docker 配置文件：

```bash
sudo vim /etc/docker/daemon.json
```

添加以下内容：

```json
{
  "ipv6": true,
  "fixed-cidr-v6": "fd00:dead:beef::/64",
  "ip6tables": true
}
```

> **为什么用 ULA 地址？** 这里的 `fd00:dead:beef::/64` 是 ULA（Unique Local Address），类似 IPv4 的私有地址段（172.16.0.0/12），不会在公网路由。如果你有公网 IPv6 网段，可以用公网子网代替——这样容器可以直接获得公网可路由的 IPv6 地址。

重启 Docker 服务：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

验证配置：

```bash
# 确认 IPv6 已启用
docker info | grep -i ipv6

# 检查 docker0 网桥是否获取到 IPv6 地址
ip -6 addr show docker0
```

## 创建 IPv6 网络

创建专用网络，避免影响现有容器：

```bash
# 自动分配子网
docker network create --ipv6 my_ipv6_network

# 指定子网
docker network create --ipv6 --subnet fd00:1::/64 my_ipv6_network

# 双栈网络（同时支持 IPv4 和 IPv6）
docker network create --ipv6 --subnet 2001:db8::/64 --gateway 2001:db8::1 my_dual_stack_network
```

启动容器时指定网络：

```bash
docker run --network my_ipv6_network -d your_image
```

验证容器能否访问 IPv6：

```bash
# Ping 外网 IPv6 地址
docker exec <container> ping6 -c 4 2400:3200::1

# 检查容器是否获得 IPv6 地址
docker exec <container> ip -6 addr show
```

## 已知限制

1. **Windows Docker Desktop** 不支持 Docker IPv6 功能（截至 2026 年）
2. **macOS** 上 Docker Desktop 使用虚拟机，IPv6 配置方式与 Linux 不同
3. **已有容器** 不会自动获取 IPv6 地址，需要重新创建或接入新网络
4. **安全组** 云服务器需单独配置 IPv6 入站/出站规则，很多云控制台有独立的 IPv6 安全组设置

## 故障排查

配置后不生效？按顺序检查：

```bash
# 1. 确认 Docker 已加载新配置
docker info | grep -i ipv6

# 2. 检查 daemon.json 语法是否正确
cat /etc/docker/daemon.json | python3 -m json.tool

# 3. 查看 Docker 日志
sudo journalctl -u docker -f

# 4. 确认 ip6tables 规则已生成
sudo ip6tables -L -n | grep DOCKER
```

## 常见问题

**Q: 已经启用了 IPv6，但容器仍然无法访问外网？**
A: 检查两点：一是宿主机是否开启了 IPv6 转发（sysctl 配置），二是云服务器的安全组是否放行了 IPv6 流量。

**Q: 可以同时使用 IPv4 和 IPv6 吗？**
A: 可以创建双栈网络（见上文），或者容器默认会继续使用 IPv4。Docker 会优先使用 IPv6 如果可用。

**Q: 如何让已有容器支持 IPv6？**
A: 需要将容器接入新建的 IPv6 网络：`docker network connect my_ipv6_network <container>`

> **注意**：阿里云等云服务器有独立的 IPv6 网关控制台，安全组规则需要单独配置。

## 总结

启用 Docker IPv6 支持需要三步：开启宿主机转发、配置 daemon.json、重启 Docker 服务。核心要点是理解 ULA 与公网地址的区别，以及安全组配置很容易被忽略。

如果你的场景恰好需要 IPv6，这篇文章能帮你绕过默认限制。如果不需要，保持默认配置就好——毕竟大多数服务仍然以 IPv4 为主。

