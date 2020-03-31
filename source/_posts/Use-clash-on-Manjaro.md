---
title: 在 Manjaro 上使用 Clash
date: 2020-03-31 10:59:34
tags:
- Clash
- 代理
catagories:
- 瞎折腾
---

# 前言

我的笔记本之前装的是 Windows 10 Pro (LTSC 魔改) + Ubuntu 18.04 LTS，这个组合用了差不多一年。但是可能是因为 Windows 10 魔改过的原因，总会有一些奇奇怪怪的 bug，比如一直无法正常运行 Win32 版本的网易云音乐。终于在前两天下定决心重装了系统，然后也顺便把 Ubuntu 换成了心水已久的 Manjaro。

<!-- more -->

# 尝试

## 初探 Clashy

装好了 Manjaro 之后自然要配置科学上网的一套东西，先是下载了 [Clashy](https://github.com/SpongeNobody/Clashy)，但是运行起来发现更新配置文件好像总会有些问题，外加上实在接受不了这个界面，索性换成了原版 Clash。

## 转战 Clash

### 安装 Clash

原版 Clash 在 Manjaro 下的安装十分方便，有现成的 AUR 包，直接在终端内运行

```bash
yaourt -S clash
```

便可以直接安装了。

注：`yaourt` 安装方法本文不涉及。

### 配置后台运行及开机自启动

这里使用的是官方文档所推荐的 `pm2`，这里同样使用

```bash
yaourt -S pm2
```

便可以安装。安装后运行如下命令

```bash
pm2 start clash
pm2 startup
pm2 save
```

这时候 Clash 就会在后台运行且加入到开机自启动的列表之中了。

### 创建 Clash 配置文件更新脚本

运行 Clash 后默认会在 `$HOME/.config/clash` 目录下生成 `config.yaml` 并下载 `Country.mmdb`，如果 `Country.mmdb` 下载失败的话可以手动下载一份然后复制到这个目录。

重点是 `config.yaml`，在 CFW / ClashX / Clash for Android 中都提供了内建的配置文件托管功能，但是在 Linux 下我们就要自力更生了。

首先要获取你的托管配置地址，然后新建一个脚本文件（这里以 `update_subscription.sh` 为例），写入如下内容：

```sh
#!/bin/bash
source /etc/profile
source /home/your_user_name/.bash_profile
SUBSCRIPTION_LINK='https://path/to/your/subscription/link'
wget $SUBSCRIPTION_LINK -O /home/your_user_name/.config/clash/config.yaml
curl -X PUT -H "Content-Type: application/json" -d '{"path": "/home/your_user_name/.config/clash/config.yaml"}' http://127.0.0.1:port/configs
```

注意以下几点：
1. 将 `your_user_name` 替换为实际的用户名
2. `SUBSCRIPTION_LINK` 后的内容请替换成实际的托管地址
3. `port` 替换为配置文件中所设置的 RESTful API 端口

这个脚本的作用是，从托管地址下载配置文件并覆盖原有的 `config.yaml`，同时通过 Clash 提供的 RESTful API 重载配置文件

### 配置自动更新

这里所使用的工具是 `crontab`，使用如下命令安装并配置开机自启动：

```bash
yaourt -S cronie
sudo systemctl enable cronie.service && sudo systemctl start cronie.service
```

新建一个空文件以方便载入 crontab

```bash
nano ~/MyCrontab
```

在文件中写入以下内容

```bash
0 */3 * * * cd /home/your_user_name/.config/clash; bash /home/your_user_name/.config/clash/update_subscription.sh > /home/your_user_name/.config/clash/subscription_history.log 2>&1
```

同样要将 `your_user_name` 替换为实际用户名。

这里的执行效果是每 3 个小时更新一次配置文件并将日志写入文件，crontab 语法此处不涉及，请自行搜索

接下来就是装入 crontab

```bash
crontab ~/MyCrontab
```

这样就可以实现脚本的自动更新以及配置重载啦

# 后记

归根结底还是学艺不精，光是最后折腾 crontab 前前后后就用了好几个小时的时间（捂脸），好在最后成果还是不错的。