### 1. 启动TCP加速算法
```
#启动BBR加速算法
wget -N --no-check-certificate "https://raw.githubusercontent.com/chiakge/Linux-NetSpeed/master/tcp.sh" && chmod +x tcp.sh && ./tcp.sh
```

### 2.安装3X-ui
```
#安装命令
bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
```

### 3.流媒体检测
```
bash <(curl -L -s check.unlock.media) -M 4
```

### 4.设置流媒体转发落地
```
#面板设置->Xray相关设置
#按需修改，示例如下
{
    "api": {
      "services": [
        "HandlerService",
        "LoggerService",
        "StatsService"
      ],
      "tag": "api"
    },
    "inbounds": [
      {
        "listen": "127.0.0.1",
        "port": 62789,
        "protocol": "dokodemo-door",
        "settings": {
          "address": "127.0.0.1"
        },
        "tag": "api"
      }
    ],
    "outbounds": [
      {
        "protocol": "freedom",
        "settings": {}
      },
      {
        "protocol": "blackhole",
        "settings": {},
        "tag": "blocked"
      },
      {
        "tag": "VPS1" ,
        "protocol": "vmess",
        "settings": {
          "vnext": [{
            "address": "your_server_ip",
            "port": 12345,
            "users": [{
              "id": "b6366334-6f42-491b-ae75-8d4bd9c89bde",#UserID
              "security": "auto",
              "alterId": 0
            }]
          }]
        }
      }
    ],
    "policy": {
      "system": {
        "statsInboundDownlink": true,
        "statsInboundUplink": true
      }
    },
    "routing": {
      "rules": [
        {
          "inboundTag": [
            "api"
          ],
          "outboundTag": "api",
          "type": "field"
        },
        {
          "ip": [
            "geoip:private"
          ],
          "outboundTag": "blocked",
          "type": "field"
        },
        {
          "outboundTag": "blocked",
          "protocol": [
            "bittorrent"
          ],
          "type": "field"
        },
        {
            "type": "field",
            "outboundTag": "VPS1",
            "domain": [
                "geosite:netflix",
                "geosite:disney"
            ]
        } 
      ]
    },
    "stats": {}
  }
```