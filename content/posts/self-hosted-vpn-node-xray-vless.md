---
title: "自建 VPN 节点：Xray VLESS 部署简明指南"
date: 2026-06-30T15:40:00+08:00
draft: false
tags: ["xray", "vless", "vpn", "networking"]
cover:
  image: "/images/self-hosted-vpn-xray-vless-cover.png"
  alt: "Self-hosted VPN: Xray and VLESS cover"
  hiddenInList: true
---

> 本文记录一套自建节点的最小化部署流程。示例中的域名、IP、UUID、私钥、端口均已脱敏，请替换为自己的配置。

## 前提

先购买域名，并将子域名解析到服务器公网 IP：

```text
vpn.example.com -> <SERVER_IP>
```

服务器建议使用干净的 Ubuntu / Debian 系统，并确认可以使用 `root` 或具备 `sudo` 权限的账号登录。

## 1. 安装 Xray

```bash
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
systemctl status xray.service
```

安装完成后，核心配置路径通常是：

```text
/usr/local/etc/xray/config.json
```

## 2. 申请 TLS 证书

如果使用普通 VLESS + TLS，需要为域名申请证书。下面以 `acme.sh` 为例：

```bash
ufw disable
curl https://get.acme.sh | sh
apt install -y socat
ln -s /root/.acme.sh/acme.sh /usr/local/bin/acme.sh

acme.sh --set-default-ca --server letsencrypt
acme.sh --issue -d vpn.example.com --standalone -k ec-256

acme.sh --installcert -d vpn.example.com --ecc \
  --key-file /usr/local/etc/xray/server.key \
  --fullchain-file /usr/local/etc/xray/server.crt
```

## 3. 服务端配置：VLESS + TLS

编辑：

```bash
nano /usr/local/etc/xray/config.json
```

写入以下配置，并替换 `<UUID>`、域名和端口：

```json
{
  "log": {
    "loglevel": "info",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "block"
      }
    ]
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 8443,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "tls",
        "tlsSettings": {
          "rejectUnknownSni": false,
          "minVersion": "1.3",
          "certificates": [
            {
              "ocspStapling": 3600,
              "certificateFile": "/usr/local/etc/xray/server.crt",
              "keyFile": "/usr/local/etc/xray/server.key"
            }
          ]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```

## 4. 可选配置：VLESS + REALITY

如果使用 REALITY，不需要申请自己的 TLS 证书，但需要生成 REALITY 密钥对：

```bash
xray x25519
```

将生成的 `Private key` 写入服务端，将 `Public key` 写入客户端。

服务端示例：

```json
{
  "log": {
    "loglevel": "info",
    "access": "/var/log/xray/access.log",
    "error": "/var/log/xray/error.log"
  },
  "routing": {
    "domainStrategy": "AsIs",
    "rules": [
      {
        "type": "field",
        "ip": ["geoip:private"],
        "outboundTag": "block"
      }
    ]
  },
  "inbounds": [
    {
      "listen": "0.0.0.0",
      "port": 59032,
      "protocol": "vless",
      "settings": {
        "clients": [
          {
            "id": "<UUID>",
            "flow": "xtls-rprx-vision"
          }
        ],
        "decryption": "none"
      },
      "streamSettings": {
        "network": "tcp",
        "security": "reality",
        "realitySettings": {
          "show": false,
          "dest": "dl.google.com:443",
          "xver": 0,
          "serverNames": ["dl.google.com"],
          "privateKey": "<REALITY_PRIVATE_KEY>",
          "shortIds": ["<SHORT_ID>"]
        }
      },
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls", "quic"]
      }
    }
  ],
  "outbounds": [
    {
      "protocol": "freedom",
      "tag": "direct"
    },
    {
      "protocol": "blackhole",
      "tag": "block"
    }
  ]
}
```

## 5. 重启服务

```bash
systemctl restart xray.service
systemctl status xray.service
```

如果状态异常，先看日志：

```bash
journalctl -u xray.service -e --no-pager
tail -n 100 /var/log/xray/error.log
```

## 6. 客户端配置要点

客户端需要保持以下字段与服务端一致：

```text
地址：vpn.example.com
端口：8443 或 59032
协议：VLESS
UUID：<UUID>
Flow：xtls-rprx-vision
传输：tcp
安全：tls 或 reality
```

VLESS + TLS 客户端核心配置：

```json
{
  "protocol": "vless",
  "settings": {
    "vnext": [
      {
        "address": "vpn.example.com",
        "port": 8443,
        "users": [
          {
            "id": "<UUID>",
            "encryption": "none",
            "flow": "xtls-rprx-vision"
          }
        ]
      }
    ]
  },
  "streamSettings": {
    "network": "tcp",
    "security": "tls",
    "tlsSettings": {
      "serverName": "vpn.example.com",
      "fingerprint": "chrome",
      "allowInsecure": false
    }
  }
}
```

REALITY 客户端需要额外填写：

```text
Server Name：dl.google.com
Public Key：<REALITY_PUBLIC_KEY>
Short ID：<SHORT_ID>
Fingerprint：chrome
```

## 7. 常见检查项

* 域名是否正确解析到服务器 IP。
* 服务器安全组是否放行对应端口。
* Xray 服务是否正常运行。
* TLS 模式下证书路径是否正确。
* REALITY 模式下公钥、私钥、Short ID、Server Name 是否匹配。
* 客户端和服务端 UUID、端口、Flow 是否一致。

## 总结

最简单的路线是先跑通 **VLESS + TLS**，确认域名、证书、端口和客户端配置都正常后，再切换到 **VLESS + REALITY**。配置时不要复用公开示例里的 UUID、私钥或 Short ID，所有敏感字段都应自行生成并妥善保存。
