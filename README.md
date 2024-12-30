以下是针对 **Ubuntu 最新版** 服务端和客户端（均为 Ubuntu）配置 WireGuard 的完整详细步骤，包括 UFW 配置，确保可以成功连接并正常访问网络。

---

## **服务端配置**

### **1. 安装 WireGuard**
在服务端上安装 WireGuard：
```bash
sudo apt update
sudo apt install wireguard -y
```

---

### **2. 生成服务端密钥对**
运行以下命令生成服务端的私钥和公钥：
```bash
umask 077
wg genkey | tee /etc/wireguard/server_private.key | wg pubkey > /etc/wireguard/server_public.key
```

- **私钥**：`/etc/wireguard/server_private.key`
- **公钥**：`/etc/wireguard/server_public.key`

记下私钥和公钥，稍后需要用到。

---

### **3. 创建服务端配置文件**
创建并编辑 `/etc/wireguard/wg0.conf`：
```bash
sudo nano /etc/wireguard/wg0.conf
```

填入以下内容：

```ini
[Interface]
# 服务端配置
PrivateKey = <服务端的私钥>         # 替换为服务端生成的私钥
Address = 10.0.0.1/24             # 服务端虚拟 IP
ListenPort = 51820                # 监听端口
# 启用 NAT 转发和路由
PostUp = iptables -A FORWARD -i wg0 -j ACCEPT; iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
PostDown = iptables -D FORWARD -i wg0 -j ACCEPT; iptables -t nat -D POSTROUTING -o eth0 -j MASQUERADE

# 客户端1
[Peer]
PublicKey = <客户端1的公钥>         # 替换为客户端生成的公钥
AllowedIPs = 10.0.0.2/32          # 分配给客户端1的虚拟 IP
```

替换以下内容：
- `<服务端的私钥>`：服务端私钥。
- `<客户端1的公钥>`：客户端生成的公钥。

保存并退出。

---

### **4. 启用 IP 转发**
编辑 `/etc/sysctl.conf` 文件：
```bash
sudo nano /etc/sysctl.conf
```

确保以下内容未被注释：
```ini
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
```

保存并应用配置：
```bash
sudo sysctl -p
```

---

### **5. 配置防火墙 (UFW)**
#### **开放 WireGuard 端口和启用转发规则**
1. 允许 WireGuard 使用的 UDP 端口：
   ```bash
   sudo ufw allow 51820/udp
   ```

2. 允许 VPN 流量转发：
   编辑 `/etc/ufw/sysctl.conf`，确保以下内容未被注释：
   ```ini
   net/ipv4/ip_forward=1
   net/ipv6/conf/all/forwarding=1
   ```

3. 添加路由规则：
   ```bash
   sudo ufw route allow in on wg0 out on eth0
   sudo ufw route allow in on eth0 out on wg0
   ```

4. 启用 UFW：
   ```bash
   sudo ufw enable
   sudo ufw reload
   ```

---

### **6. 启动 WireGuard 服务**
1. 启动 WireGuard 接口：
   ```bash
   sudo wg-quick up wg0
   ```

2. 设置开机启动：
   ```bash
   sudo systemctl enable wg-quick@wg0
   ```

3. 检查 WireGuard 状态：
   ```bash
   sudo wg
   ```

---

## **客户端配置**

### **1. 安装 WireGuard**
在客户端上安装 WireGuard：
```bash
sudo apt update
sudo apt install wireguard -y
```

---

### **2. 生成客户端密钥对**
运行以下命令生成客户端的私钥和公钥：
```bash
umask 077
wg genkey | tee client_private.key | wg pubkey > client_public.key
```

- **私钥**：`client_private.key`
- **公钥**：`client_public.key`

将公钥发送给服务端，并将其添加到服务端的 `wg0.conf` 文件中。

---

### **3. 创建客户端配置文件**
创建并编辑 `/etc/wireguard/wg0.conf`：
```bash
sudo nano /etc/wireguard/wg0.conf
```

填入以下内容：

```ini
[Interface]
PrivateKey = <客户端私钥>         # 替换为客户端生成的私钥
Address = 10.0.0.2/24            # 客户端虚拟 IP
DNS = 8.8.8.8                    # 使用 Google DNS

[Peer]
PublicKey = <服务端公钥>         # 替换为服务端的公钥
Endpoint = <服务端公网IP>:51820  # 替换为服务端的公网 IP 和端口
AllowedIPs = 0.0.0.0/0, ::/0     # 路由所有流量通过 VPN
PersistentKeepalive = 25         # 防止 NAT 超时
```

替换以下内容：
- `<客户端私钥>`：客户端生成的私钥。
- `<服务端公钥>`：服务端生成的公钥。
- `<服务端公网IP>`：服务端的公网 IP。

保存并退出。

---

### **4. 启动 WireGuard 客户端**
1. 启动 WireGuard：
   ```bash
   sudo wg-quick up wg0
   ```

2. 设置开机启动：
   ```bash
   sudo systemctl enable wg-quick@wg0
   ```

3. 检查状态：
   ```bash
   sudo wg
   ```

---

### **5. 测试连接**
1. 在客户端上，测试连接到服务端的虚拟 IP（如 `10.0.0.1`）：
   ```bash
   ping 10.0.0.1
   ```

2. 测试客户端是否可以访问外网：
   ```bash
   curl ifconfig.me
   ```
   如果返回服务端的公网 IP，表示连接正常。

---

## **总结**

### **服务端关键点**
1. 配置 `/etc/wireguard/wg0.conf` 并启用 NAT 转发。
2. 设置 UFW 防火墙规则，确保允许 UDP 端口 51820 和转发流量。
3. 启用 IP 转发。

### **客户端关键点**
1. 配置 `/etc/wireguard/wg0.conf`，确保填写正确的服务端公钥和 IP。
2. 添加 `DNS` 参数解决外网问题。
