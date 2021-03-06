# Trojan-go + Caddy


**创建必要的文件目录**
```
$ apt update && apt upgrade -y && apt install curl wget vim unzip gnupg2 cron socat -y
$ mkdir -p /etc/trojan-go
$ mkdir -p /var/www/html
$ mkdir -p /var/log/trojan-go
$ mkdir -p /home/tls
$ mkdir -p /etc/caddy
$ mkdir -p /var/log/caddy
```

### acme.sh

```
$ wget -O -  https://get.acme.sh | sh -s email=my@example.com
# 推荐使用 DNS APi 申请证书
$ acme.sh --issue --dns dns_cf --keylength ec-256 -d example.com
```

## Caddy

### 安装 Caddy
#### Installing from package
```
$ sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https
$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
$ curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
$ apt update && apt install caddy -y
```
```
caddy start/reload/stop
```
#### Download a Caddy binary 
**Refer to : [Install Caddy use Static binaries](https://caddyserver.com/docs/install#static-binaries)**
[Run Caddy as a daemon ](https://github.com/klzgrad/naiveproxy/wiki/Run-Caddy-as-a-daemon)

**Caddy.service**
```
[Unit]
Description=Caddy
Documentation=https://caddyserver.com/docs/
After=network.target network-online.target
Requires=network-online.target

[Service]
Type=notify
User=root
Group=root
ExecStart=/usr/local/bin/caddy/caddy run --environ --config /usr/local/etc/caddy/caddy.json //json配置 Caddyfile
ExecReload=/usr/local/bin/caddy/caddy reload --config /usr/local/etc/caddy/caddy.json //json配置 Caddyfile 
TimeoutStopSec=5s
LimitNOFILE=1048576
LimitNPROC=512
PrivateTmp=true
ProtectSystem=full
AmbientCapabilities=CAP_NET_BIND_SERVICE

[Install]
WantedBy=multi-user.target
```

### caddy.json
```
cat > /etc/caddy/caddy.json <<EOF
{
  "admin": {
    "disabled": true
  },
  "logging": {
    "logs": {
      "default": {
        "writer": {
          "output": "file",
          "filename": "/var/log/caddy/access.log"
        },
        "level": "ERROR"
      }
    }
  },
  "apps": {
    "http": {
      "servers": {
        "h1": {
          "listen": [":80"], //http默认监听端口
          "routes": [{
            "handle": [{
              "handler": "static_response",
              "headers": {
                "Location": ["https://{http.request.host}{http.request.uri}"] //http自动跳转https
              },
              "status_code": 301
            }]
          }]
        },
        "h1h2c": {
          "listen": ["127.0.0.1:853"], //http/1.1与h2c server本地监听端口
          "routes": [{
            "match": [{
              "host": ["example.com"]
            }],
            "handle": [{
              "handler": "subroute",
              "routes": [{
                "handle": [{
                  "handler": "headers",
                  "response": {
                    "set": {
                      "Strict-Transport-Security": ["max-age=31536000; includeSubDomains; preload"]
                    }
                  }
                }]
              },
              {
                "handle": [{
                  "handler": "file_server",
                  "root": "/var/www/html" //修改为自己存放的WEB文件路径
                }]
              }]
            }]
          }],
          "automatic_https": {
            "disable": true
          },
          "allow_h2c": true
        }
      }
    }
  }
}
EOF
```

## Trojan-go

```
$ wget --no-check-certificate https://github.com/p4gefau1t/trojan-go/releases/download/v0.10.6/trojan-go-linux-amd64.zip
$ unzip trojan-go-linux-amd64.zip 
$ mv trojan-go /usr/local/bin/
$ mv geosite.dat geoip.dat /etc/trojan-go/
$ chmod +x /usr/local/bin/trojan-go
```

### Trojan-go.service

```
cat > /etc/systemd/system/trojan-go.service <<EOF
[Unit]
Description=Trojan-Go - An unidentifiable mechanism that helps you bypass GFW
Documentation=https://p4gefau1t.github.io/trojan-go/
After=network.target nss-lookup.target

[Service]
User=root
CapabilityBoundingSet=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
AmbientCapabilities=CAP_NET_ADMIN CAP_NET_BIND_SERVICE
NoNewPrivileges=true
ExecStart=/usr/local/bin/trojan-go -config /etc/trojan-go/config.json
Restart=on-failure
RestartSec=10s
LimitNOFILE=infinity

[Install]
WantedBy=multi-user.target
EOF
```

### Trojan-go config.json

```
cat > /etc/trojan-go/config.json <<EOF
{
    "run_type": "server",
    "local_addr": "0.0.0.0",
    "local_port": 443,
    "remote_addr": "127.0.0.1",
    "remote_port": 853, //h2与http/1.1回落端口
    "password": [
        "Your-Trojan-Password"
    ],
    "log_level": 2,
    "log_file": "/var/log/trojan-go/access.log",
    "ssl": {
        "cert": "/home/tls/example.com/fullchain.cer", //换成自己的证书，绝对路径。
        "key": "/home/tls/example.com/private.key", //换成自己的密钥，绝对路径。
        "key_password": "",
        "cipher": "TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384:TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256:TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305_SHA256",
        "prefer_server_cipher": true,
        "alpn":[
            "http/1.1"
        ],
        "reuse_session": true,
        "session_ticket": false,
        "plain_http_response": "",
        "fallback_addr": "",
        "fallback_port": 0
    },
    "tcp": {
        "no_delay": true,
        "keep_alive": true,
        "prefer_ipv4": true
    },
    "router": {
        "enabled": true,
        "block": [
            "geoip:private",
            "geoip:cn",
            "bittorrent"
        ],
        "geoip": "/etc/trojan-go/geoip.dat",
        "geosite": "/etc/trojan-go/geosite.dat"
    }
}

}
EOF
```

```
$ /usr/local/bin/trojan-go  -config /etc/trojan-go/config.json
$ systemctl enable trojan-go
$ systemctl daemon-reload
$ systemctl start/stop/restart trojan-go
$ systemctl status trojan-go
$ journalctl -u trojan-go
```
