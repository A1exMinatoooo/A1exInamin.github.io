---
title: 乐天杂志的图片后处理
date: 2020-09-15 21:14:31
tags:
- 杂志
- Python
---
昨天偶然浏览到了[烈之斩](https://fireattack.wordpress.com)的[文章](https://fireattack.wordpress.com/2020/08/14/rakuten-magazine)，发现原来我一直在用的乐天杂志是有高清图片提供的，但只在手动触发后才会加载，文章内也谈到了对保存下载的文件的处理方式，便诞生了写一个小程序来实现批量处理的想法，以下是实现思路。

<!--more-->

# 获取高清图片文件

乐天杂志的图片资源疑似没有访问鉴权，所以这里不用实际地址展示，仅简单说明使用 aria2c 获取文件的方法。

## 生成文件列表

使用 `echo` 以及 `sed` 来生成顺序文件列表，具体使用方式如下

e.g. 已知高清图片最后一页的文件名为 `AVED0_A0_L0_P83.pdf`，需要生成从 `P00` 开始到 `P83` 结束的列表，那么分别执行如下两条命令：

```bash
echo https://path/to/AVED0_A0_L0_P{00..83}.pdf > dllist.txt
# sed 命令因操作系统不同可能会有变化，以下为 macOS 下的使用方法
# 这条命令会将 echo 生成的字符串中的空格替换成换行符，方便 aria2c 调用
sed -i '' 's/[ ]/\'$'\n/g' dllist.txt
```

这样我们会得到一个包含所有图片下载地址的文本文档

![](/img/2020/09/15/rakuten-1.png)

## 下载文件

下载使用 aria2c，在终端内输入如下命令

```bash
aria2c -c -j 16 -i dllist.txt
```

便会开始下载，`-j` 为指定同时下载的文件数量，`-c` 代表开启断点续传

下载好的文件如下图所示

![](/img/2020/09/15/rakuten-2.png)

# 编写处理脚本

## 处理文件头

首先，为了方便下一步重命名处理，我们需要先将文件名头部的 `AVED0_A0_L0_P` 去除，实现代码如下

```python
# 省略遍历部分，需导入 re 包
result = re.match(r"(AVED0\_A0\_L0\_P)(.*)", filename).group(2)
if result:
    os.rename(filename, result)
```

然后准备好新文件的命名

```python
portion = os.path.splitext(result)
if portion[1] == ".pdf":
    newname = portion[0] + ".jpg"
    # 打开 pdf 文件
    infile = open(result, 'rb')
    # 打开待写入文件
    outfile = open(newname, 'wb')
```

接下来对文件进行切割

```python
# 读取文件头部，并查找 JPEG 文件头
hex_data = binascii.b2a_hex(infile.read(1500))
str_sidx = bytes.decode(hex_data).find('ffd8ff')
str_sidx = str_sidx // 2
# 确定位置后，重置文件指针至文件头部
infile.seek(0, 0)
# 切割文件
outfile.write(infile.read()[str_sidx:])
infile.close()
outfile.close()
```

遍历文件时保存了一个计数器变量，作为函数的返回值，接下来要用到

## 对文件重命名

⚠️因为我所需要下载的杂志都是右开本，而乐天杂志的文件命名是按照左开本设计的，所以需要进行这一步处理。如果需要保存左开本杂志，请自行对脚本进行修改

本函数有一个参数，为上一步返回的计数器变量值，下文以 `end_cnt` 称呼

首先遍历文件，并对文件列表进行排序，需要额外新开一个计数器变量

这里要讲一下重命名的思路，上文提到乐天杂志的文件命名是从 `00` 开始的，`00` 代表杂志的封面，然后从 `01` 开始，奇数页为左侧页面，偶数页面为右侧页面，但是这与右开本的排序是相反的，所以需要将奇数页 + 1 变成右侧页面，将偶数页 - 1 变成左侧页面。同时还要注意对封底的处理，如果杂志有单独的封底，那么最后一个奇数页是不可以处理的。

```python
count = 0
for root, dirname, filenames in os.walk(dir):
    filenames.sort()
    for filename in filenames:
        if os.path.splitext(filename)[1] == '.jpg':
            portion = os.path.splitext(filename)
            # 对第一个文件直接重命名
            if count == 0:
                newname = "P000.jpg"
                os.rename(filename, newname)
                count += 1
                print("Renamed: \t" + filename + "\t ---> \t" + newname)
            # 对最后一个文件分情况处理，如果为封底，则不修改文件名；如果为双页式，则继续执行
            elif count == end_cnt - 1 and count % 2 == 1:
                i = int(portion[0])
                newname = "P" + str(i).zfill(3) + ".jpg"
                os.rename(filename, newname)
                print("Renamed: \t" + filename + "\t ---> \t" + newname)
            # 奇数页面
            elif count != 0 and count % 2 == 1:
                i = int(portion[0]) + 1
                newname = "P" + str(i).zfill(3) + ".jpg"
                os.rename(filename, newname)
                count += 1
                print("Renamed: \t" + filename + "\t ---> \t" + newname)
            # 偶数页面
            elif count != 0 and count % 2 == 0:
                i = int(portion[0]) - 1
                newname = "P" + str(i).zfill(3) + ".jpg"
                os.rename(filename, newname)
                count += 1
                print("Renamed: \t" + filename + "\t ---> \t" + newname)
            else:
                pass
```

## 删除处理好的 pdf 文件

删除当前目录下扩展名为 `.pdf` 的文件

```python
def del_files():
    dir = "./"
    for root, dirs, files in os.walk(dir):
        for name in files:
            if name.endswith(".pdf"):
                os.remove(os.path.join(root,name))
```

## 拼接图片

因为杂志经常会出现使用两页来显示一张图片，所以使用 `ImageMagick` 进行拼接，Ubuntu 或 macOS 可以直接使用 `apt` 或 `brew` 进行安装

首先遍历当前目录下的文件，并获得一份只有上面处理好的 jpg 图片的文件列表

```python
def stitch_image():
    dir = "./"
    count = 0
    for root, dirname, filenames in os.walk(dir):
        filenames.sort()
        newlist = []
        for filename in filenames:
            if re.match("P.*\.jpg", filename):
                newlist.append(filename)
```

对拼接图片的处理思路是这样的，首先封面不需要拼接，同时为了避免含有封底时的重复处理，这里直接选择偶数页面进行拼接

```python
for filename in newlist:
    if os.path.splitext(filename)[1] == '.jpg':
        if count %2 == 1 or count == 0:
            count += 1
            continue
        else:
            portion = os.path.splitext(newlist[count - 1])
            print("Stitching " + newlist[count - 1] + " and " + filename + " to stitch-" + portion[0] + "-" + filename)
            subprocess.check_output("convert " + filename + " " + newlist[count - 1] + " " + "+append " + "stitch-" + portion[0] + "-" + filename, shell=True)
            count += 1
```

最后直接在主函数内进行调用即可

```python
if __name__ == "__main__":
    end_cnt = process_files()
    rename_files(end_cnt)
    del_files()
    stitch_image()
```

# 使用示例

![](/img/2020/09/15/rakuten-3.png)

标准输出会提示重命名操作以及合并操作，待程序执行完后就可以获得源文件以及拼接好的图片了

![](/img/2020/09/15/rakuten-4.png)

# 说在最后

之前用过 Kindle Unlimited / Magazine Walker / Book Walker 読み放題，可以说这三家都比乐天的画质要好，但是 Kindle Unlimited 这几个月的书越来越少，Magazine Walker 停止服务后新开的 Book Walker 読み放題没有声 G 而且阅读器看一部分杂志的时候多了一圈恶心的白框，乐天杂志现在反而成了性价比最高的选择。

说白了还是穷（捂脸）

`# EOF`
