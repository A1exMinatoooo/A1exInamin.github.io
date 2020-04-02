---
title: 在更新 Windows 或 BIOS 之后找回丢失的 grub 引导
date: 2020-04-02 20:34:52
tags:
- Manjaro
- Linux
- grub
catagories:
- 瞎折腾
---

今天在更新之后发现 grub 引导从启动项菜单中消失了，遂折腾一番，记录在此，以备不时之需。

<!-- more -->

首先准备一个载有任意 grub 引导的 Live USB，这里使用的是 Manjaro 的安装镜像，在 Windows 下使用 Rufus 烧入 U 盘。

然后重启，使用 USB 引导，在启动项选择菜单里按 `c` 进入 `GRUB Commandline` 模式

![Live USB 引导菜单](https://i.postimg.cc/4xmRWznk/IMG-20200402-194112.jpg)

然后会进入 grub 模式，你会看到类似这样的命令符界面

```
grub>
```

首先使用 `ls` 命令查看当前的分区情况，你会得到类似于下面这样的结果：

```
grub> ls
(proc) (hd0) (hd0,msdos2) (hd1) (hd1,gpt1) (hd2) (hd2,gpt4) (hd2,gpt3) (hd2,gpt2) (hd2,gpt1)
```

这时候我们需要知道哪个是现在的 EFI 启动分区，可以使用 `ls (hd*,gpt*)/boot/grub` 命令来查看有没有 grub 引导，如果得到类似下面的结果说明已经找到了正确的 grub 分区（此处以 `(hd2,gpt4)` 为例，下同）

```
grub> ls (hd2,gpt4)/boot/grub
x86_64efi/  grubenv themes/ fonts/  grub.cfg
```

得知 grub 分区之后，依次执行下列命令，执行完最后一条命令之后便会回到引导菜单，看到熟悉的 Manjaro Linux 引导项了

```grub
set prefix=(hd2,gpt4)/boot/grub
set root=hd2,gpt4
insmod normal
normal
```

**注意：请务必根据实际情况修改分区，切勿直接照抄！**

![启动项回来了](https://i.postimg.cc/J0kB3yJb/2020-04-02-19-45-19-414.jpg)

选择 `Manjaro Linux` 引导进入系统，打开终端，输入如下命令查看 `/boot` 分区位置

```bash
mount | grep boot
```

得到类似如下的输出

```
/dev/nvme0n1p1 on /boot/efi type vfat (rw,relatime,fmask=0077,dmask=0077,codepage=437,iocharset=iso8859-1,shortname=mixed,utf8,errors=remount-ro)
```

这里的 `/dev/nvme0n1p1` 就是接下来要使用的内容

依次执行如下命令

```bash
sudo update-grub
sudo grub-install /dev/nvme0n1p1
```

重启进入 BIOS，就会发现熟悉的 manjaro 他回来了！

![BIOS 启动项设置](https://i.postimg.cc/Vk1KKtgs/912-F337-B696-B624-D33963-E2-A46821-BA0.jpg)

`# EOF`