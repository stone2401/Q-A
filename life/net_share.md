## 用 Linux 双网口实现网络共享：解决家中单网口无法同时上网的问题

#### 背景故事

最近，随着 fnos（一款 NAS 系统）火热推广，我决定利用家中闲置的旧电脑来搭建 NAS。然而，问题随之而来：出租屋里只有一个网口，NAS 使用了这个网口，我的主力电脑却无法上网！幸运的是，这台旧电脑配备了两个网口，正好可以用来实现网络共享。

设备清单：

•	一台双网口的 Linux 电脑（NAS）

•	一台单网口的 Windows 主机

•	一个连接外网的有线网络

•	两根网线

接下来，我们就一步步实现如何通过 Linux 双网口设备共享网络，让 Windows 主机也能顺畅上网。
注：`思路通用，不同系统版本查找对应工具链操作即可`

---

### 1. 检查现有的网络设备

首先，通过以下命令查看 Linux 系统的网卡信息，确认哪个网口用于连接外网，哪个网口可以用来共享网络：

```bash
# 也可以是其他命令
nmcli device status
```
假设：

•	eth0 连接外网（路由器）

•	eth1 连接内网的 Windows 主机

---

### 2. 配置内网接口（eth1）静态 IP 地址

为内网接口 eth1 配置一个静态 IP 地址（例如 192.168.1.1/24）：
```bash
nmcli con add type ethernet con-name internal ifname eth1 ip4 192.168.1.1/24
nmcli con up internal
```
命令解释：

•	nmcli con add：添加一个新的网络连接。

•	type ethernet：设置连接类型为以太网。

•	ifname eth1：指定网卡接口为 eth1。

•	ip4 192.168.1.1/24：分配静态 IP 地址和子网掩码。

注意：如果你的外部路由器使用的网关地址是 192.168.1.1，需要将内网 IP 设置为其他网段，例如 192.168.2.1/24，避免冲突。

---

### 3. 启用 IP 转发

打开 Linux 的 IP 转发功能，允许网络流量从一个接口转发到另一个接口：

编辑 /etc/sysctl.conf 文件，找到或添加以下行：
```txt
net.ipv4.ip_forward=1
```
然后运行以下命令应用更改：
```bash
sudo sysctl -p
```

---

### 4. 设置 NAT 转发

使用 iptables 配置 NAT，让内网的 Windows 设备能够通过 Linux 的外网接口访问互联网：

```bash
sudo iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
```

命令解释：

•	-t nat：指定使用 NAT 表。

•	POSTROUTING：表示在数据包离开时修改其源 IP。

•	-o eth0：指定外网接口为 eth0。

•	MASQUERADE：动态隐藏源 IP，让数据包看起来是从 Linux 的外网接口发出的。

---

### 5. 持久化 iptables 规则 (可选)

如果不需要长期使用 NAT，可以跳过此步骤。否则，为了防止重启后配置丢失，可以保存 iptables 规则：
```bash
sudo apt install iptables-persistent
sudo netfilter-persistent save
```

---

### 6. 配置 Windows 主机的网络

手动配置 Windows 主机的网卡，确保它与 Linux 的内网接口在同一网段：

1.	打开 网络设置 > 网络状态 > 更改适配器选项。

2.	找到 Windows 网卡，右键选择 属性。

3.	双击 Internet 协议版本 4 (TCP/IPv4)，手动配置以下信息：

•	IP 地址：192.168.1.2（或与 Linux 内网地址相同网段的其他地址）

•	子网掩码：255.255.255.0

•	默认网关：192.168.1.1（Linux 的内网接口 IP）

•	DNS 服务器：8.8.8.8（Google 公共 DNS）或外网路由器的 DNS 地址

---

### 测试网络连接

完成配置后，在 Windows 主机上打开浏览器，尝试访问网站。如果一切正常，Windows 主机就可以通过 Linux 双网口设备实现网络共享了。

---

总结

通过 Linux 的双网口配置静态 IP、启用 IP 转发和 NAT 转发功能，可以轻松实现网络共享。这种方案不仅适合家庭环境，还可以在服务器部署和网络调试中发挥作用。如果你有类似需求，不妨按照上述步骤尝试一下！