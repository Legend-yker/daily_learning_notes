# 🛠️ 虚拟机网络问题修复复盘指南（Ubuntu 20.04 + VMware）

## 适用场景
- 桥接模式无法获取 IP 或无法与主机互通
- NetworkManager 显示设备“未托管”，图形界面网络选项缺失
- 需要设置静态 IP 并与主机同网段通信

## 1️⃣ 检查虚拟机网络模式与 VMware 设置
目的：确保桥接模式正确绑定到物理网卡。

步骤：
- 以管理员身份运行 VMware。
- 打开 “编辑 → 虚拟网络编辑器”。
- 确认存在 VMnet0，类型为“桥接模式”。
- 选中 VMnet0，在“已桥接到”下拉中手动选择当前上网的物理网卡（如无线网卡），不要选“自动”。
- 点击“应用”和“确定”。

## 2️⃣ 检查虚拟机内部网络状态
目的：确认网卡状态和 NetworkManager 运行情况。

```bash
# 查看所有网络接口状态
ip addr

# 查看 NetworkManager 管理的设备状态
nmcli device status

# 查看 NetworkManager 服务状态
systemctl status NetworkManager
```
预期：ens33 状态应为 connected 或 connecting；若为 unmanaged，继续下一步。

## 3️⃣ 修复 NetworkManager 未托管问题
### 3.1 检查 Netplan 配置
Netplan 配置文件通常位于 /etc/netplan/。

```bash
# 列出 Netplan 配置文件
ls /etc/netplan/

# 查看文件内容（以实际文件名为准）
sudo cat /etc/netplan/01-network-manager-all.yaml
```
确保内容类似如下（renderer: NetworkManager 且 ens33 启用 DHCP）：

```yaml
network:
  ethernets:
    ens33:
      dhcp4: true
  version: 2
  renderer: NetworkManager
```
应用变更：

```bash
sudo netplan apply
```

### 3.2 检查 NetworkManager 状态文件
状态文件可能记录了“网络禁用”，导致 NetworkManager 不工作。

```bash
# 查看状态文件
sudo cat /var/lib/NetworkManager/NetworkManager.state
```
若出现 `NetworkingEnabled=false`，修改为 true：

```bash
sudo sed -i 's/NetworkingEnabled=false/NetworkingEnabled=true/' /var/lib/NetworkManager/NetworkManager.state
```

### 3.3 重启 NetworkManager
```bash
sudo systemctl restart NetworkManager
```

### 3.4 再次检查设备状态与日志
```bash
nmcli device status

# 若仍为 unmanaged，查看日志定位原因
journalctl -u NetworkManager -f
```
常见原因：`/etc/network/interfaces` 中存在配置，或其他服务（如 systemd-networkd）占用。

## 4️⃣ 若 NM 无法接管 → 使用 systemd-networkd 配置静态 IP
当 NetworkManager 无法管理时，通过 Netplan + systemd-networkd 配置网络更稳定。

### 4.1 备份原 Netplan 配置
```bash
sudo cp /etc/netplan/01-network-manager-all.yaml /etc/netplan/01-network-manager-all.yaml.bak
```

### 4.2 编辑 Netplan 配置文件
```bash
sudo vim /etc/netplan/01-network-manager-all.yaml
```
替换为以下内容（根据你的网络环境修改 IP、网关）：

```yaml
network:
  ethernets:
    ens33:
      dhcp4: no
      addresses:
        - 192.168.2.100/24      # 你想要设置的静态 IP
      gateway4: 192.168.2.1      # 网关地址（与主机相同）
      nameservers:
        addresses: [8.8.8.8, 114.114.114.114]  # DNS 服务器
  version: 2
  renderer: networkd             # 改用 systemd-networkd
```

### 4.3 应用配置
```bash
sudo netplan apply
```
若提示 `systemd-networkd is not running`，先启动：

```bash
sudo systemctl start systemd-networkd
sudo systemctl enable systemd-networkd
sudo netplan apply
```

### 4.4 验证网络
```bash
# 查看 IP 是否配置成功
ip addr show ens33

# 测试网关
ping 192.168.2.1

# 测试外网
ping 8.8.8.8
```
说明：此方式下虚拟机可正常上网，但图形界面网络设置可能简陋（NM 未使用）。

## 5️⃣ 主机侧排查（Windows）
若虚拟机网络正常但无法 ping 通主机，问题通常在 Windows 端。

### 5.1 检查 Windows 防火墙
- 临时关闭防火墙测试：控制面板 → Windows Defender 防火墙 → 启用或关闭防火墙 → 全部关闭。
- 若虚拟机可 ping 通主机，则需添加 ICMP 入站规则：
  - 打开“高级安全 Windows Defender 防火墙”
  - 入站规则 → 新建规则 → 自定义 → 所有程序
  - 协议类型: ICMPv4 → 自定义 → 勾选“回显请求”
  - 允许连接 → 全选配置文件 → 命名完成

### 5.2 检查多个网络接口同网段冲突
```cmd
ipconfig /all
```
查看除物理网卡（Wi‑Fi/有线）外，是否还有其他适配器（如 VMware Virtual Ethernet Adapter）也获得了 192.168.2.x 的 IP。若有：
- 禁用该适配器（控制面板 → 网络连接 → 右键禁用），或
- 修改其 IP 为其他网段（如 192.168.99.1/24，网关留空）。

### 5.3 验证互通
```bash
# 虚拟机 ping 主机（替换为你的主机实际 IP）
ping 192.168.2.153

# 主机 ping 虚拟机
ping 192.168.2.100
```

## 6️⃣ 最终验证与持久化
```bash
# 重启虚拟机后检查 IP 是否依然存在
ip addr show ens33

# 确认能 ping 通外网和主机
ping 8.8.8.8
```
若一切正常，表示修复成功。

## 📋 常用命令速查表

| 目的 | 命令 |
| --- | --- |
| 查看 IP 地址 | `ip addr` |
| 查看 NM 设备状态 | `nmcli device status` |
| 查看 NM 服务状态 | `systemctl status NetworkManager` |
| 重启 NM | `sudo systemctl restart NetworkManager` |
| 修改 NM 状态文件 | `sudo sed -i 's/NetworkingEnabled=false/NetworkingEnabled=true/' /var/lib/NetworkManager/NetworkManager.state` |
| 应用 Netplan 配置 | `sudo netplan apply` |
| 启动 systemd-networkd | `sudo systemctl start systemd-networkd` |
| 查看 NM 日志 | `journalctl -u NetworkManager -f` |
| Windows 查看所有接口 IP | `ipconfig /all` |
| Windows 查看路由表 | `route print -4` |

## 🧭 总结
- 多数问题源于 VMware 桥接网卡选择错误、NetworkManager 状态异常、或 Windows 防火墙/多接口冲突。
- 遇到无法互通时，按“VMware 设置 → 虚拟机网络状态 → NetworkManager 修复 → systemd-networkd 静态 IP → Windows 侧防火墙与接口冲突”的顺序排查。
- 需要对外提供服务时优先桥接；仅需访问外网可用 NAT；调试/隔离场景用 Host‑Only 或内部网络。 
