---
title: 一种快速 mock 数据的解决方案
date: 2019-08-16 17:10:29
description: "客户端开发联调是必不可少的过程，但经常因为接口未按时提供、环境未正常启动导致需求延期。"
categories:
- 测试
- mock
---

本文介绍一种数据mock方案，实现同事间共享mock数据，客户端开发不再依赖后端环境，提升开发效率。

<br>

### 背景

<br>

联调是必不可少的过程，但经常因为接口未按时提供、环境未正常启动导致需求延期。
应对以上情况，开发一般使用mock数据进行测试，然而大量的mock数据并没有得到统一的管理，后续若有其他同事需要这份数据，必须抓包，重新配置mock数据。

<br>

### 目标

<br>

针对以上痛点，我决定采取一定措施简化mock流程，保留mock数据，实现以下功能：

* 组内同事共享mock数据
* 支持导出配置，借助 Charles Rewrite 功能快速实现mock需求
* 下发配置，客户端接入配置，便可在客户端一键切换mock环境

<br>

### 实现

<br>

#### 数据存储

<br>

因共用一份数据，所以必须入库存储。平台使用mysql存储数据。<br>
主体信息：分组(`group`)、域名(`host`)、接口(`api`)、参数(`params`)、内容(`data`)、描述(`desc`)<br>
辅助信息：版本(`version`)、创建用户(`c_user`)、更新用户(`u_user`)、创建时间(`create_time`)、更新时间(`update_time`)<br>
说明：版本默认值为1，用户每更新1次加1。

<br>

#### 数据映射

<br>

用通配符拦截请求，提取请求uri，查询配置的返回数据，封装返回。<br>
以POST接口为例(GET接口类似)：

<img src="https://haitao.nos.netease.com/10a9d801-3157-4568-be17-ca66b4410b3f_1344_1066.jpg" alt="mock post 请求" width="672" hegiht="533">

映射伪代码如下：

        // 取出所有包含`api`的mock 配置
        mockModels = mockMapper.getMockDataByApi(requestURI)
        for (MockModel mockModel : mockModels) {
            if (TODO: 请求参数和配置参数匹配) {
                    TODO :封装返回数据
                    break;
                }
            }
        }
        TODO: 返回数据

<br>

#### 添加配置

<br>

在mock数据平台点击添加API，跳转至如下页面：

<img src="https://haitao.nos.netease.com/20190816173828_162e4a41-41d5-4e6b-90ce-c33752054004.jpg" alt="编辑mock数据" width="1221">

根据提示填写完整接口信息后，点击提交，即可创建一条新的mock配置。

<br>

#### 更新配置

<br>

mock数据平台首页点击mock配置，即可进入编辑页，如下图：

<img src="https://haitao.nos.netease.com/20190816174039_9f147df9-9aeb-4216-9305-32ff9b03a171.jpg" alt="编辑mock数据" width="1221">


编辑mock配置信息，点击提交即可。配置实时生效，随时mock各种异常情况。

<br>

#### Charles配置文件

<br>

每次新增或修改API mock配置，均会重新生成一份Charles Rewrite配置，配置文件格式如下：

```
<?xml version="1.0" encoding="UTF-8"?>
<?charles serialisation-version='2.0' ?>
<rewriteSet-array>
    <rewriteSet>
        <active>false</active>
        <name>******</name>
        <hosts>
            <locationPatterns/>
        </hosts>
        <rules>
            <rewriteRule>
                <active>false</active>
                <ruleType>6</ruleType>
                <matchHeaderRegex>false</matchHeaderRegex>
                <matchValueRegex>false</matchValueRegex>
                <matchRequest>false</matchRequest>
                <matchResponse>false</matchResponse>
                <newHeaderRegex>false</newHeaderRegex>
                <newValueRegex>false</newValueRegex>
                <matchWholeValue>false</matchWholeValue>
                <caseSensitive>false</caseSensitive>
                <replaceType>2</replaceType>
                <matchValue>******</matchValue>
                <newValue>******</newValue>
            </rewriteRule>

            ......

        <rules>
    <rewriteSet>

    ......

<rewriteSet-array>
```

<br>

### 使用

<br>

* Charles导入配置

打开`Charles` --> `Tools` --> `Rewrite` --> `Import`, 选择上面生成的配置文件，导入成功。

<img src="https://haitao.nos.netease.com/e47db172-b498-4232-9694-3084f6f485dd_1288_1008.jpg" alt="mock post 请求" width="644" hegiht="504">

接下来便可根据需要启用接口，实现数据mock

<br><br><br><br><br>
