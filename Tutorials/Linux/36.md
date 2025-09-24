Linux 里有一个 **SOCKS5 代理在 127.0.0.1:10820**，但大多数软件只支持 **HTTP/HTTPS 代理**，所以需要“把 SOCKS5 转换成 HTTP/HTTPS 代理”，或者让系统环境变量正确指向。

---

## 方法一：环境变量（仅支持 HTTP/HTTPS，不直接支持 SOCKS5）

Linux 默认的 `http_proxy/https_proxy` 环境变量只认 `http://` 或 `https://`，大多数程序不会直接认 `socks5://`。
如果要尝试（部分 `curl`、`wget` 新版本支持 `socks5://`）：

```bash
export http_proxy="socks5://127.0.0.1:10820"
export https_proxy="socks5://127.0.0.1:10820"
```

测试：

```bash
curl -v https://ipinfo.io/ip
```

如果不支持，会报错。

---

## 方法二：用 **privoxy / polipo** 转换 SOCKS5 → HTTP 代理（推荐）

安装 privoxy：

```bash
sudo apt update
sudo apt install privoxy -y
```

编辑配置文件：

```bash
sudo nano /etc/privoxy/config
```

添加：

```
forward-socks5t / 127.0.0.1:10820 .
```

保存后启动：

```bash
sudo systemctl restart privoxy
```

默认 privoxy 会监听 `127.0.0.1:8118`，这是 HTTP 代理。

然后设置环境变量为 HTTP：

```bash
export http_proxy="http://127.0.0.1:8118"
export https_proxy="http://127.0.0.1:8118"
```

---

## 方法三：proxychains（全局命令行代理）

如果不想改环境变量，可以装 proxychains，让任何命令都走 SOCKS5：

```bash
sudo apt install proxychains4 -y
```

编辑配置：

```bash
sudo nano /etc/proxychains4.conf
```

最后加一行：

```
socks5  127.0.0.1 10820
```

使用时：

```bash
proxychains4 curl https://ipinfo.io/ip
```

---

✅ 总结：

* **只改环境变量**：部分工具支持 `socks5://`，可能不通用。
* **用 privoxy 转换成 http(s) 代理**：通用性最好。
* **proxychains**：适合命令行全局代理。


## Docker 走代理

Docker Daemon 才能真正走代理。
在 Linux (systemd) 下，可以这样：

```bash
sudo mkdir -p /etc/systemd/system/docker.service.d
sudo tee /etc/systemd/system/docker.service.d/proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://your-proxy:port/"
Environment="HTTPS_PROXY=http://your-proxy:port/"
Environment="NO_PROXY=localhost,127.0.0.1"
EOF
```

然后：

```bash
sudo systemctl daemon-reload
sudo systemctl restart docker
```

---

有时候即使 `docker pull` 成功，容器里 `apt update` 还是出不去，这时你可以在 `/etc/docker/daemon.json` 里写 DNS，或者在 `docker run` 时传代理变量：

```bash
docker run -e http_proxy=http://your-proxy:port \
           -e https_proxy=http://your-proxy:port \
           ubuntu apt update
```

---

✅ 结论：

* **系统环境变量**只对本地命令行有效，不会影响 `docker pull`。
* **必须给 Docker Daemon 配置代理**，这样 `docker pull` 才能走代理。
* 如果容器内部也要走代理，可以再传 `-e http_proxy` 进去。

要不要我帮你写一个完整的 **Docker Daemon 代理配置脚本**，直接一键设置并重启？
