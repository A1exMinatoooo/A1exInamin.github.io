---
title: 在服务器上实现对YouTube低延迟的缓冲以方便转播
date: 2020-03-03 08:53:00
tags:
- 转播
- Vtuber
- 虚拟主播
catagories:
- 瞎折腾
---

# 前言

昨天做管人转播的时候又开始疯狂转圈，然后打开统计信息发现管人果然用的还是 Ultra Low Latency （超低延迟直播）。这种缓冲区只留 2-3 秒而且还动不动归零的直播对于我这个烂网来说是真的遭不住，于是在群里吐槽之后另一位转圈 man 说他是自己建了一个缓冲区然后再进行推流的，得知这个思路之后决定开始自己折腾一波。

<!--more-->

最开始是尝试先在本地使用 `streamlink` 进行缓存，然后使用 PotPlayer 进行播放，OBS 使用 PotPlayer 画面进行转播。但是发现可能是因为本地的网络状况不佳，录制下来的文件经常出现跳帧的情况，于是决定将缓存这一环节放在服务器上进行，然后本地串流服务器保存的内容。

# 开始配置

## 配置环境

VPS 需要先配置 nginx 环境，如果使用的机器是新机器的话，建议手动配置，参照[这篇文章](https://github.com/winshining/nginx-http-flv-module/blob/master/README.CN.md)手动编译所需要的 nginx 环境

我的服务器已经配置好了 lnmp 环境，所以需要进行升级，这里参照了[这篇文章](https://lnmp.org/faq/lnmp1-2-upgrade.html)进行处理。

1. 首先下载 `nginx-http-flv-module` 扩展模块

```bash
wget https://github.com/winshining/nginx-http-flv-module/archive/v1.2.7.tar.gz
```

2. 解压

```bash
tar -xzvf v1.2.7.tar.gz
```

3. 下载最新的 lnmp 安装包并解压

```bash
wget http://soft.vpser.net/lnmp/lnmp1.7beta.tar.gz -cO lnmp1.7beta.tar.gz && tar zxf lnmp1.7beta.tar.gz && cd lnmp1.7
```

4. 修改 lnmp.conf

```bash
nano lnmp.conf
```

在 `Nginx_Modules_Options` 中添加 rtmp 扩展

```
Nginx_Modules_Options='--add-module=/path/to/nginx-rtmp-module'
```

5. 升级 Nginx

在 lnmp 目录下

```bash
./upgrade.sh nginx
```

安装完成之后输入 `/usr/local/nginx/sbin/nginx -V` 如果结尾出现 `--add-module=/usr/local/nginx/extend_module/nginx-rtmp-module` 则代表安装成功

## 添加 Vhost

使用如下命令新建一个 vhost：

```bash
lnmp vhost add
```

注：请事先在域名解析商处配置好域名解析，以便 lnmp 能自动申请 SSL 证书。

注2：这一步添加 vhost 的目的是为了方便访问推流 / 串流地址，如果不申请 vhost 的话直接使用域名进行推流也是可以的。同时，使用 vhost 设置状态监控页面也可以防止与已有站点冲突

## 修改 Nginx 配置文件

> 本阶段配置主要参考[这篇文章](https://zhangshuqiao.org/2018-01/基于Nginx搭建视频直播服务器/)，感谢原作者

1. 修改 vhost 文件

首先修改网站的 vhost 文件，添加状态监控页面的配置

```bash
nano /usr/local/nginx/conf/vhost/yourdomain.com.conf
```

在下半段的 443 端口中添加如下内容

```nginxconf
location /stat {
    rtmp_stat all;
    rtmp_stat_stylesheet stat.xsl;
}

location /stat.xsl {
    root /usr/local/nginx/extend_module/nginx-rtmp-module/;
}
```

2. 修改 Nginx 配置文件

由于 rtmp 内容不能直接添加至 vhost 配置中，因此需要单独修改 Nginx 的配置文件

```bash
nano /usr/local/nginx/conf/nginx.conf
```

添加如下内容

```nginxconf
rtmp {
    server {
        listen 1935;              # 监听的端口
        server_name yourdomain.com;
        chunk_size 4000;
        application live {        # rtmp推流请求路径
            live on;
            record off;
            #publish_notify on;   # 推流验证，和下一行需一起使用
            #on_publish http://localhost/auth.php;    # 推流验证，具体可参照后文

            hls on;
            hls_path /home/hls/;  # 视频流hls切片文件目录(自己创建)
            hls_fragment 3s;      # hls单个切片时长，会影响延迟
            hls_playlist_length 32s;  # hls总缓存时间，会影响延迟

            meta on;              # 如果http-flv播放时首帧出现问题，可改为off
            gop_cache on;         # 可以减少首帧延迟
        }
    }
}
```

接着修改前端展示页面的配置

```bash
nano /usr/local/nginx/conf/vhost/yourdomain.com.conf
```

在 SSL 端口配置下增加如下内容

```nginxconf
location /hls {
    types {
        application/vnd.apple.mpegurl m3u8;
        video/mp2t ts;
    }
    alias /home/hls;            # 和上面的 hls_path 保持一致
    expires -1;
    add_header Cache-Control no-cache;
}
```

两个配置修改完之后重启 Nginx 以使配置生效

```bash
lnmp nginx restart
```

注：此处暂不涉及前端的页面展示，如果想实现在网页端播放可自行搜索 H5 播放器控件及 `hls.js` 相关内容

# 测试直播

在 VPS 内推流可使用如下命令

```bash
streamlink YOUTUBE_LINK best -O | ffmpeg -i pipe:0 -vcodec copy -acodec copy -f flv rtmp://localhost/live/LIVE_NAME
```

注1：`YOUTUBE_LINK` 以及 `LIVE_NAME` 请根据实际情况替换。

注2：`streamlink` 及其依赖安装方法不在本文讨论范围内。

外部推流将 `localhost` 替换为前文配置的域名即可

然后在播放器内输入如下地址即可播放

```
https://yourdomain.com/hls/LIVE_NAME.m3u8
```

![效果展示](https://i.postimg.cc/KY0867gd/live-test.png)

# 后记

过程中可以说是踩了不少的坑，VPS 几年都没有动过配置，几乎可以说是“年久失修”，环境也很混乱，查了许多教程总算是“揉”出了自己的实际经验。不过折腾的过程还是挺有意思的。这次可以愉快地转圈了（误

`#EOF`