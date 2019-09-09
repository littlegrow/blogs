---
title: Android 签名和多渠道打包
date: 2019-7-28 15:00:00
description: "发布APK是需要签名的, 签名机制标明了APK的发行机构，在Android应用和框架中有着十分重要的作用"
categories:
- APK签名
- V1V2
---

Android Studio 2.2以上版本打包apk的时候，我们会发现多了个签名版本（v1、v2）选择，如下图红色方框所示
![](https://haitao.nos.netease.com/20190806103143_1984930a-9b92-4060-8f3c-2ebe79a7e759.jpg)

Android 7.0中引入了APK Signature Scheme v2，v1是jar Signature来自JDK。

以下片段摘自谷歌开发者官网：

    In case you want to disable adding v1 or v2 signatures when building with the Android Gradle plugin, you can add these lines to your signingConfig section in build.gradle:

    v1SigningEnabled false
    v2SigningEnabled false

    Note: both signing schemes are enabled by default in Android Gradle plugin 2.2.

<br>

### 签名是什么
<br>

签名是摘要与非对称密钥加密相相结合的产物，摘要就像内容的一个指纹信息，一旦内容被篡改，摘要就会改变，签名是摘要的加密结果，摘要改变，签名也会失效。Android APK签名也是这个道理，如果APK签名跟内容对应不起来，Android系统就认为APK内容被篡改了，从而拒绝安装，以保证系统的安全性。

<br>

V1签名通过META-INF中的三个文件保证签名及信息的完整性。
![](https://haitao.nos.netease.com/20190806103736_6d070105-e239-4ce3-9f33-395f0ac590cf.jpg)

    MANIFEST.MF：摘要文件，存储文件名与文件SHA1摘要（Base64格式）键值对
    CERT.SF：二次摘要文件，存储文件名与MANIFEST.MF摘要条目的SHA1摘要（Base64格式）键值对
    CERT.RSA 证书（公钥）及签名文件，存储keystore的公钥、发行信息、以及对CERT.SF文件摘要的签名信息（利用keystore的私钥进行加密过）

<br>

v2签名方案进行签名时，会在APK文件中插入一个APK签名分块，该分块位于zip中央目录部分之前并紧邻该部分。在APK签名分块内，签名和签名者身份信息会存储在APK签名方案v2分块中，保证整个APK文件不可修改；
![](https://haitao.nos.netease.com/20190806104206_23d4f1f3-6a03-4f25-8acd-e6c534977d9b.webp)

<br><br>

### 为什么签名

<br>

拥有线上APP开发经验的Android开发工程师都清楚，Android系统禁止安装签名不一致的应用，一般开发过程中使用测试签名，应用上线时使用正式签名。
要破解一个APK，必然需要进行APK签名，一般情况无法再与APK原先的签名保持一致，出于软件安全的角度，我们就可以通过比对APK的签名情况，判断APK是否“官方”发行。

<br><br>

### 渠道打包

<br>

#### v1和v2的签名使用

<br>

    1. 只勾选v1签名并不会影响什么，但是在7.0上不会使用更安全的验证方式
    2. 只勾选V2签名7.0以下会直接安装完显示未安装，7.0以上则使用了V2的方式验证
    3. 同时勾选V1和V2则所有机型都没问题

<br>

#### 多渠道打包

<br>

传统的打包方法是在AndroidManifest添加渠道标示，每打一次包修改一次标示的名称。效率特别低，一个稍微大一点的项目打上几十个渠道包可能需要几个小时半天的时间。

<font color="red">注：仅使用V1签名多渠道打包可以在 `META-INF` 目录下创建一个带有渠道标识的空文件，这里不再讨论</font> 

<br><br>

##### 我的多渠道打包方案
<br>

在[糗事百科](https://www.qiushibaike.com/)工作时，负责Android项目的打包发布工作，下面简单介绍下我这边如何支持多渠道打包。

我们知道使用V2签名后的APK文件不能再修改，但如果先对文件进行修改，而后再进行签名就不会出现任何问题。

<br>

主要打包流程如下图：
![](https://haitao.nos.netease.com/20190729141512_a57eb2b5-764f-4135-afda-814f86920df7.png)

优点：这种方式仅需要一次打包，解决了V2签名后不能再修改的问题，打出基础包耗时2min左右，每修改并签名一个渠道包耗时10～15秒，渠道总量在200以内，20min内可以打出全量包。

缺点：每个渠道包均需要签名，耗时较长

<br><br>

##### 美团多渠道打包原理

Android应用使用的APK文件就是一个带签名信息的ZIP文件，根据ZIP文件格式规范，每个ZIP文件的最后都必须有一个叫 Central Directory Record 的部分，这个CDR的最后部分叫”end of central directory record”，这一部分包含一些元数据，它的末尾是ZIP文件的注释。注释包含Comment Length和File Comment两个字段，前者表示注释内容的长度，后者是注释的内容，正确修改这一部分不会对ZIP文件造成破坏，利用这个字段，我们可以添加一些自定义的数据，原理很简单，就是将渠道信息存放在APK文件的注释字段中。

详细打包过程及配置可查看[packer-ng-plugin](https://github.com/mcxiaoke/packer-ng-plugin)

<br><br>

### 参考文档

<br>

[APK packaging in Android Studio 2.2](https://android-developers.googleblog.com/2016/11/understanding-apk-packaging-in-android-studio-2-2.html)
[Android V1及V2签名原理简析](https://www.jianshu.com/p/95096ca209e1)
[Android签名机制](https://www.cnblogs.com/pulove/p/5783693.html)
[Android多渠道打包:美团多渠道打包原理以及使用](https://www.jianshu.com/p/fe20417337e0)

<br><br><br><br><br>