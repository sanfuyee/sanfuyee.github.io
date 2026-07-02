---
title: "IPTV 入门：协议、客户端、公开源与自部署方案"
date: 2026-07-02T18:30:00+08:00
draft: false
tags: ["iptv", "streaming", "hls", "media-server", "self-hosting"]
cover:
  image: "/images/iptv-cover.png"
  alt: "IPTV streaming guide cover"
  hiddenInList: true
---

IPTV（Internet Protocol Television，网络协议电视）是通过互联网传输电视内容的技术，区别于传统的卫星或有线电缆广播。用户不再被动等待广播信号，而是像访问网页一样，按需拉取视频流。

IPTV 的核心优势：

- 不依赖物理线缆，任意联网设备均可收看
- 支持实时直播与按需点播（VOD）
- 成本远低于传统有线电视
- 可自由选择频道和内容

## IPTV 的工作原理

IPTV 的传输架构分为三层：内容来源、流媒体服务器、播放客户端。

```text
内容来源                  流媒体服务器                     客户端
（广播/录播/点播）  ->  HLS / RTMP / MPEG-TS  ->  手机 / 电视 / 电脑
                        M3U 列表 / Xtream API
                              ^
                       EPG 节目单 / 频道 Logo
```

## 流媒体协议

| 协议 | 全称 | 特点 | 适用场景 |
| --- | --- | --- | --- |
| HLS | HTTP Live Streaming | 分片传输，兼容性最好 | 直播、点播均适用 |
| MPEG-TS | MPEG Transport Stream | 低延迟，容错好 | 传统直播广播 |
| RTMP | Real-Time Messaging Protocol | 延迟最低 | 推流直播 |
| RTSP | Real Time Streaming Protocol | 控制灵活 | 安防摄像头、局域网 |

HLS 是目前最主流的协议。流媒体服务器将原始视频按时间切割成若干小片段，通常 2 到 10 秒，生成一个 `.m3u8` 索引文件。客户端按顺序请求这些片段拼接播放，同时附带 EPG 节目单、频道图标等元数据。

## IPTV 源格式

IPTV 源是“告诉客户端去哪里拉流”的配置信息，主要有两种形式。

### M3U / M3U8 播放列表

M3U 是一个纯文本文件，每个频道由两行构成：

```m3u
#EXTM3U
#EXTINF:-1 tvg-id="CCTV1" tvg-name="CCTV-1" tvg-logo="https://logo.url/cctv1.png" group-title="央视",CCTV-1 综合
http://your-server/live/cctv1.m3u8

#EXTINF:-1 tvg-id="CCTV5" tvg-name="CCTV-5" group-title="体育",CCTV-5 体育
http://your-server/live/cctv5.m3u8

#EXTINF:-1 tvg-id="BBC.NEWS" group-title="国际新闻",BBC News
http://your-server/live/bbc_news.m3u8
```

字段说明：

- `tvg-id`：用于匹配 EPG 节目单的频道 ID
- `tvg-name`：频道显示名称
- `tvg-logo`：频道图标 URL
- `group-title`：客户端中的分组名称
- 最后一行：实际流地址，支持 HLS `.m3u8`、MPEG-TS、RTMP 等

### Xtream Codes API

更现代的接入方式是通过服务器地址、用户名、密码鉴权，客户端自动获取频道列表、VOD 点播库和 EPG，无需手动维护 M3U 文件。它常见于付费 IPTV 服务商。

接入格式：

```text
服务器：http://your-server:8080
用户名：your_username
密码：your_password
```

## 主流 IPTV 客户端

### TiviMate

- 平台：Android TV、Fire TV
- 价格：免费 / Premium $4.99/年
- 格式：M3U、Xtream Codes API
- 亮点：界面流畅，支持多播放列表、录制、Catch-up、画中画

配置方法：打开 App -> 设置 -> 播放列表 -> 添加播放列表 -> 输入 M3U URL 或填写 Xtream API 信息。

### IPTV Smarters Pro

- 平台：Android、iOS、PC、Android TV
- 价格：免费
- 格式：M3U、Xtream Codes API
- 亮点：全平台覆盖，多账号管理，内置 EPG，支持 VOD 和剧集

### Kodi

- 平台：Windows、macOS、Linux、Android、iOS、Android TV
- 价格：完全免费开源
- 格式：M3U，需要安装 PVR IPTV Simple Client 插件
- 亮点：高度可定制，插件生态丰富，可与 Emby/Jellyfin 深度集成

配置方法：安装 PVR IPTV Simple Client 插件 -> 设置 -> M3U 播放列表 URL 填入地址 -> 启用插件 -> 在“直播电视”中查看频道。

### VLC

- 平台：Windows、macOS、Linux、iOS、Android
- 价格：完全免费开源
- 格式：M3U、M3U8、HLS、RTMP、RTSP
- 亮点：无需配置，直接“打开网络串流”粘贴 URL 即可播放

桌面端操作：媒体 -> 打开网络串流 -> 粘贴 M3U URL -> 播放。

### OTT Navigator

- 平台：Android、Android TV
- 价格：免费
- 格式：M3U、Xtream Codes、MAG 门户
- 亮点：专为 TV 遥控操作优化，分类筛选，支持 MAG 机顶盒格式

### Infuse

- 平台：iOS、iPadOS、Apple TV、macOS
- 价格：免费 / Pro $9.99/年
- 格式：M3U8、HLS
- 亮点：Apple 原生体验，支持 4K HDR、Dolby Vision、AirPlay

### IPTVnator

- 平台：Windows、macOS、Linux
- 价格：完全免费开源
- 格式：M3U、M3U8
- 亮点：桌面端专属，界面简洁，支持 EPG，开源可审计

## 公开免费源

以下为 GitHub 上长期维护的合规公开源，内容均为各国合法免费播出的频道。

### iptv-org/iptv

全部频道：

```text
https://iptv-org.github.io/iptv/index.m3u
```

按国家，将 `cn` 替换为 ISO 国家码：

```text
https://iptv-org.github.io/iptv/countries/cn.m3u
https://iptv-org.github.io/iptv/countries/us.m3u
https://iptv-org.github.io/iptv/countries/jp.m3u
https://iptv-org.github.io/iptv/countries/hk.m3u
```

按分类：

```text
https://iptv-org.github.io/iptv/categories/news.m3u
https://iptv-org.github.io/iptv/categories/sports.m3u
https://iptv-org.github.io/iptv/categories/kids.m3u
https://iptv-org.github.io/iptv/categories/movies.m3u
```

GitHub 仓库：[iptv-org/iptv](https://github.com/iptv-org/iptv)

全球 8000+ 频道，按国家、语言、分类细分，每日自动更新。

### Free-TV/IPTV

```text
https://raw.githubusercontent.com/Free-TV/IPTV/master/playlist.m3u8
```

GitHub 仓库：[Free-TV/IPTV](https://github.com/Free-TV/IPTV)

它只收录真正免费且稳定可用的频道，数量少但质量高，全部 HD，无成人内容，覆盖主要国家公共广播频道。

## EPG 节目单配置

EPG（Electronic Program Guide）让客户端显示每个频道正在播什么、下一档是什么。常见格式是 XMLTV。

```text
# 中国大陆
https://iptv-org.github.io/epg/guides/cn.xml.gz

# 香港
https://iptv-org.github.io/epg/guides/hk.xml.gz

# 台湾
https://iptv-org.github.io/epg/guides/tw.xml.gz

# 美国
https://iptv-org.github.io/epg/guides/us.xml.gz
```

在 TiviMate、IPTV Smarters 或 Jellyfin 的“节目单”设置中填入上述 URL，客户端会通过频道的 `tvg-id` 字段自动匹配频道与节目信息。

## 自部署 IPTV 服务

### 方案一：Nginx + 静态 M3U

这种方式最简单，适合个人转发已有的流地址，只需要维护一个 M3U 文本文件。

```nginx
server {
    listen 80;
    server_name iptv.yourdomain.com;

    # 提供 M3U 播放列表
    location /playlist.m3u {
        root /var/www/iptv;
        add_header Content-Type 'application/x-mpegurl';
        add_header Access-Control-Allow-Origin '*';
    }

    # 反向代理流地址
    location /stream/ {
        proxy_pass http://upstream-source/;
        proxy_buffering off;
        proxy_read_timeout 300s;
    }
}
```

### 方案二：Jellyfin

Jellyfin 是开源媒体服务器，内置 Live TV 支持，可以同时管理电影、剧集和直播。

```bash
docker run -d \
  --name jellyfin \
  -p 8096:8096 \
  -v /path/to/config:/config \
  -v /path/to/media:/media \
  --restart unless-stopped \
  jellyfin/jellyfin
```

安装后访问 `http://your-server:8096`，进入“直播电视” -> 设置调谐器 -> 添加 M3U 调谐器 -> 填入 M3U URL。

### 方案三：Threadfin

Threadfin 可以将 M3U 源转换为 Plex、Emby、Jellyfin 可识别的网络调谐器格式，支持多源合并、频道过滤、EPG 映射。

```bash
docker run -d \
  --name threadfin \
  -p 34400:34400 \
  -v /threadfin/config:/home/threadfin/conf \
  --restart unless-stopped \
  fyb3roptik/threadfin
```

配置好后，在 Plex 或 Jellyfin 的“直播电视”中填入：

```text
http://your-server:34400/discover.json
```

系统会将它识别为网络调谐器，并自动拉取频道列表。

### 方案四：Xtream-UI

Xtream-UI 是功能完整的 IPTV 管理面板，支持用户管理、Xtream API、EPG 自动抓取、带宽限制、多服务器负载均衡。它更适合多用户共享场景。

```bash
# 需要 Ubuntu 20.04，安装前确认系统要求
bash <(curl -s https://raw.githubusercontent.com/XUI-ONE/xui.one/main/install.sh)
```

安装后通过面板添加流媒体转码服务器、导入 M3U 源、创建用户账号，用户通过 Xtream API 或 M3U 接入。

## 部署方案对比

| 方案 | 难度 | 适用人群 | 核心功能 | 最低配置 |
| --- | --- | --- | --- | --- |
| Nginx + M3U | 低 | 个人 | 静态列表托管 | 1 核 512MB |
| Jellyfin | 中 | 家庭 | 直播 + 点播 + 媒体库 | 2 核 2GB |
| Threadfin | 中 | 家庭 | 多源合并 + Plex/Emby 对接 | 1 核 1GB |
| Xtream-UI | 高 | 多用户 | 用户管理 + 全功能 | 4 核 4GB |

## 网络要求参考

| 画质 | 码率 | 推荐带宽（单路） |
| --- | --- | --- |
| SD（标清） | 1-2 Mbps | 3 Mbps |
| HD（720p） | 3-5 Mbps | 8 Mbps |
| FHD（1080p） | 5-8 Mbps | 12 Mbps |
| 4K UHD | 15-25 Mbps | 35 Mbps |

## 参考资源

- [iptv-org/iptv](https://github.com/iptv-org/iptv)：最大公开 IPTV 频道库
- [iptv-org/awesome-iptv](https://github.com/iptv-org/awesome-iptv)：IPTV 相关工具与资源汇总
- [Free-TV/IPTV](https://github.com/Free-TV/IPTV)：高质量免费频道列表
- [Jellyfin 官方文档](https://jellyfin.org/docs/)：媒体服务器配置指南
- [TiviMate 官网](https://tivimate.com/)：Android TV IPTV 播放器

> 注意：公开源收录的均为各国合法免费播出的公共频道，稳定性依赖上游链接，流失效属正常现象。如需稳定收看付费内容，请通过有版权授权的正规服务商订阅。自部署 IPTV 服务时，请确保所转发的内容具备合法授权。
