---
title: Tinypng批量压缩图片
date: 2020-0507 17:20:00
description: "工作中经常会遇到图片压缩问题"
categories:
- 图片压缩
---

### [tinypng](https://tinypng.com/)

tinypng是工作中常用的免费图片压缩工作，但是一次只能压缩20张图，且需要手动上传下载，效率低！

本文通过python脚本教大家如何批量压缩图片

### 获取 API KEY

登入[开发者官网](https://tinypng.com/developers)到网站申请,只需要一个名字和一个邮箱就可以,API key会以链接的形式发到邮箱里

<font color="red">注: 一个账号一个月只能压缩500张图，但对于日常工作已经足够使用了</font>


### 编写批量压缩脚本

```
# encoding=utf8
import os
import sys

from tinify import tinify

tinify.key = "**********  此处替换为您申请到的API key *************"

useage = """

    tiny 图片|图片文件夹 目标文件夹
        
        必要参数：
            图片|图片文件夹
        
        可选参数：
            目标文件夹, 默认 `./tiny`
        
        压缩图片默认存储在当前目录 `tiny` 文件夹下

"""


def formatSize(bytes):
    bytes = float(bytes)
    kb = bytes / 1024
    if kb >= 1024:
        M = kb / 1024
        if M >= 1024:
            G = M / 1024
            return "%.2fG" % G
        else:
            return "%.2fM" % M
    else:
        return "%.2fkb" % kb


def tinifypng(source, target):
    print
    originSize = os.path.getsize(source)
    print("picture: " + formatSize(originSize) + " " + source)
    sourced = tinify.from_file(source)
    sourced.to_file(target)
    tinySize = os.path.getsize(target)
    print("tiny to: " + formatSize(tinySize) + " " + target)
    print("cut down rate:" + ("%0.2f" % ((originSize - tinySize) / float(originSize) * 100)) + "%")
    print("=" * 48)


if __name__ == "__main__":
    if len(sys.argv) >= 2:
        path = sys.argv[1]
        if os.path.exists(path):
            sourcePath = os.path.abspath(path)
            targetDir = sys.argv[2] if len(sys.argv) >= 3 else "./tiny"
            if not os.path.exists(targetDir):
                os.mkdir(targetDir)
            targetPath = os.path.abspath(targetDir)
            if os.path.isfile(sourcePath):
                targetFile = targetPath + "/" + os.path.basename(sourcePath)
                tinifypng(sourcePath, targetFile)
            else:
                files = os.listdir(sourcePath)
                for file in files:
                    if file.endswith(".png") or file.endswith(".jpg"):
                        fileSource = sourcePath + "/" + file
                        fileTarget = targetPath + "/" + file
                        tinifypng(fileSource, fileTarget)
        else:
            print("资源不存在")
    else:
        print(useage)

```

### 生成可执行文件


<br>

`Python` 脚本可使用 `pyinstaller` 模块很方便的打包成可执行文件(使用方法参看[pyinstaller官网](http://www.pyinstaller.org/))

```
pyinstaller -F [脚本文件名].py
```

### 使用示例

```
➜  res git:(master) tinypng drawable-xxhdpi
picture: 2.72kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/logo_push.png
tiny to: 1.64kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/logo_push.png
cut down rate:39.51%
================================================
picture: 10.02kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/logo_round.png
tiny to: 3.64kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/logo_round.png
cut down rate:63.69%
================================================
picture: 9.10kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/icon_member_sweet_magic_level.png
tiny to: 3.76kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/icon_member_sweet_magic_level.png
cut down rate:58.69%
================================================
picture: 3.58kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/icon_pay_default.png
tiny to: 1.45kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/icon_pay_default.png
cut down rate:59.49%
================================================
picture: 11.09kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/icon_login_weibo.png
tiny to: 2.24kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/icon_login_weibo.png
cut down rate:79.80%
================================================
picture: 2.24kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/drawable-xxhdpi/icon_tab_message_unselect.png
tiny to: 0.79kb /Users/liuleigang/workspace/android/ssos-android/app/src/main/res/tiny/icon_tab_message_unselect.png
cut down rate:64.75%
================================================
......

```