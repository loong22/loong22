* Linux 部署 frps（服务端）并设置开机自启
* Windows 部署 frpc（客户端）并用 includes 机制加载目录下所有 toml 文件
* 用 **NSSM** 实现 Windows 开机自启

---

## 🚀 一、Linux 服务器端：部署 frps

### 1️⃣ 下载 frp

```bash
cd /usr/local
wget https://github.com/fatedier/frp/releases/latest/download/frp_linux_amd64.tar.gz
tar -zxvf frp_linux_amd64.tar.gz
mv frp_* frp
cd frp
```

---

### 2️⃣ 创建 frps 配置文件

```bash
nano /usr/local/frp/frps.toml
```

写入以下内容（可根据实际修改端口和 token）：

```toml
bindPort = 7000
bindUdpPort = 7001

dashboardPort = 7500
dashboardUser = "admin"
dashboardPwd = "123456"

auth.token = "mySecretToken"
```

---

### 3️⃣ 测试运行

```bash
./frps -c ./frps.toml
```

看到类似 `start frps success` 输出，说明启动成功。

---

### 4️⃣ 设置开机自启（Systemd）

```bash
sudo nano /etc/systemd/system/frps.service
```

写入：

```ini
[Unit]
Description=FRP Server Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/frp/frps -c /usr/local/frp/frps.toml
Restart=on-failure

[Install]
WantedBy=multi-user.target
```

加载并启用：

```bash
sudo systemctl daemon-reload
sudo systemctl enable frps
sudo systemctl start frps
sudo systemctl status frps
```

> ✅ 状态为 `active (running)` 表示启动成功。

---

## 💻 二、Windows 客户端：部署 frpc

### 1️⃣ 下载 frpc

从 GitHub 获取最新版本（与服务器版本一致）：

> [https://github.com/fatedier/frp/releases](https://github.com/fatedier/frp/releases)

解压到一个目录，例如：

```
C:\frp\
```

---

### 2️⃣ 目录结构建议

```
C:\frp\
 ├─ frpc.exe
 ├─ config.toml         ← 主配置文件
 └─ config\
     ├─ rdp.toml        ← 子配置文件示例
     └─ other.toml      ← 其他代理配置（可选）
```

---

### 3️⃣ 主配置文件（C:\frp\config.toml）

```toml
serverAddr = "your.server.ip"
serverPort = 7000

auth.method = "token"
auth.token = "mySecretToken"

includes = ["./config/*.toml"]
```

> 💡 这表示 frpc 会自动加载当前目录下 `config` 文件夹中的所有 `.toml` 配置文件。

---

### 4️⃣ 创建子配置文件（C:\frp\config\rdp.toml）

```toml
[[proxies]]
name = "rdp_tcp"
type = "tcp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 3389

[[proxies]]
name = "rdp_udp"
type = "udp"
localIP = "127.0.0.1"
localPort = 3389
remotePort = 3389
```

> ✅ 这样 frpc 会通过 TCP 与 UDP 同时转发 Windows 的远程桌面端口。

---

### 5️⃣ 测试运行

在 `C:\frp\` 目录下打开 CMD 或 PowerShell：

```bash
frpc.exe -c ./config.toml
```

若输出显示连接成功：

```
start proxy success
```

则说明与 frps 通信正常。

---

### 6️⃣ 使用 NSSM 设置开机自启

#### 安装 NSSM

下载 NSSM（[https://nssm.cc/download）](https://nssm.cc/download）)
解压到如 `C:\nssm\` 目录，并添加到系统环境变量 PATH。

#### 注册服务

以管理员身份运行 CMD：

```bash
nssm install frpc
```

在弹出窗口中填写：

* **Path:** `C:\frp\frpc.exe`
* **Startup directory:** `C:\frp\`
* **Arguments:**

  ```
  -c ./config.toml
  ```

点击 **Install service**。

---

#### 启动服务并设置自动启动

```bash
nssm start frpc
nssm set frpc Start SERVICE_AUTO_START
```

检查状态：

```bash
nssm status frpc
```

显示 `SERVICE_RUNNING` 即为成功。

---

## ✅ 三、测试连接

在任意外部电脑上打开远程桌面：

```
mstsc → your.server.ip:3389
```

如果一切配置正确，即可通过 FRP 访问你的 Windows 桌面。

---

## 📋 最终结构总结

| 项目       | 文件路径                       | 说明                     |
| -------- | -------------------------- | ---------------------- |
| FRPS 主程序 | `/usr/local/frp/frps`      | Linux 服务端              |
| FRPS 配置  | `/usr/local/frp/frps.toml` | 服务端配置                  |
| FRPC 主程序 | `C:\frp\frpc.exe`          | Windows 客户端            |
| FRPC 主配置 | `C:\frp\config.toml`       | 主配置，自动包含 config/*.toml |
| FRPC 子配置 | `C:\frp\config\rdp.toml`   | RDP 代理定义               |
| 自启工具     | NSSM                       | 管理 frpc 服务             |

---
