在 Ubuntu 24.04 Server 上配置网络主要通过 `Netplan` 工具进行，它是 Ubuntu 默认的网络配置管理工具。以下是配置网络的详细步骤：

### 1. **确认网络设备名称**
首先，查看当前网络设备的名称：
```bash
ip link
```
输出会显示类似 `eth0`（有线接口）或 `wlan0`（无线接口）的设备名称。记录需要配置的接口名称。

### 2. **编辑 Netplan 配置文件**
Ubuntu 24.04 使用 Netplan 配置文件来管理网络设置。配置文件通常位于 `/etc/netplan/` 目录下，文件名可能是 `01-netcfg.yaml` 或类似名称。

1. 打开配置文件：
   ```bash
   sudo nano /etc/netplan/01-netcfg.yaml
   ```

2. 根据需求配置网络。以下是常见的配置示例：

#### **静态 IP 配置**
如果需要设置静态 IP 地址，配置文件示例如下：
```yaml
network:
  version: 2
  ethernets:
    eth0:  # 替换为你的网络接口名称
      dhcp4: no
      addresses:
        - 192.168.1.100/24  # 静态 IP 地址和子网掩码
      gateway4: 192.168.1.1  # 默认网关
      nameservers:
        addresses:
          - 8.8.8.8  # DNS 服务器
          - 8.8.4.4
```
- `eth0`：替换为实际的网络接口名称。
- `addresses`：设置静态 IP 地址和子网掩码（例如 `/24` 表示 255.255.255.0）。
- `gateway4`：网关地址。
- `nameservers`：DNS 服务器地址。

#### **动态 IP 配置（DHCP）**
如果使用 DHCP 自动获取 IP 地址，配置文件示例如下：
```yaml
network:
  version: 2
  ethernets:
    eth0:  # 替换为你的网络接口名称
      dhcp4: yes
```

#### **无线网络配置（Wi-Fi）**
如果需要配置 Wi-Fi，配置文件示例如下：
```yaml
network:
  version: 2
  wifis:
    wlan0:  # 替换为你的无线接口名称
      dhcp4: yes
      access-points:
        "SSID_NAME":  # 替换为 Wi-Fi 的 SSID
          password: "YOUR_PASSWORD"  # 替换为 Wi-Fi 密码
```
- `wlan0`：替换为实际的无线接口名称。
- `SSID_NAME` 和 `password`：替换为 Wi-Fi 的名称和密码。

### 3. **应用配置**
编辑完成后，保存文件并应用配置：
```bash
sudo netplan apply
```

### 4. **验证网络配置**
1. 检查 IP 地址是否正确分配：
   ```bash
   ip addr show
   ```
2. 测试网络连通性：
   ```bash
   ping 8.8.8.8
   ```
3. 验证 DNS 是否工作：
   ```bash
   nslookup google.com
   ```

### 5. **常见问题排查**
- **配置文件语法错误**：如果 `netplan apply` 报错，检查 YAML 文件的缩进（必须使用空格，不能用 Tab）。
- **网络服务未启动**：确保网络服务正常运行：
  ```bash
  sudo systemctl restart systemd-networkd
  ```
- **无线网络问题**：确保 `wpasupplicant` 已安装并启用：
  ```bash
  sudo systemctl enable --now wpa_supplicant
  ```

### 6. **备份配置文件**
在修改配置文件之前，建议备份：
```bash
sudo cp /etc/netplan/01-netcfg.yaml /etc/netplan/01-netcfg.yaml.bak
```

### 注意事项
- 确保配置文件中的接口名称与 `ip link` 输出一致。
- 如果使用的是云服务器（如 AWS、阿里云），可能需要参考云服务商的网络配置指南。
- 对于复杂网络配置（如 VLAN、桥接），请参考 Netplan 官方文档：`https://netplan.io/`。

