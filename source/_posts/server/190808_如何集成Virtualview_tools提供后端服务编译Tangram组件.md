---
title: 如何在后端服务中集成virtualview_tools编译Tangram组件
date: 2019-8-8 19:00:00
description: "目前开发的线上APP首页动态化使用Tangram实现，本文讲述如何在后端服务中集成virtualview_tools"
categories:
- virtualview_tools
- Tangram
---

### 背景

<br>

采用Tangram动态化方案后，组件编译、发布、历史版本管理变得很复杂，因此我们决定开发一个组件管理平台提供这些服务，[官方文档--virtualview_tools](https://github.com/alibaba/virtualview_tools/blob/master/README-ch.md) 对于后端服务集成仅简单的提示了一小段，见下图：
![](https://haitao.nos.netease.com/20190808185018_c4ac875f-ac55-40ea-a5ce-e6fa3fc5d0fa.jpg)
描述并不清晰,因此本文讲述我是如何集成 `virtualview_tools` 编译Tangram组件的。

<br><br>

### 下载 `virtualview_tools`

<br>

[下载地址](https://github.com/alibaba/virtualview_tools/releases)

下载后解压，找到`TemplateWorkSpace`目录，我们集成所需的文件均在这个目录下，其查找路径为：

    `virtualview_tools` --> `compiler-tools` --> `TemplateWorkSpace`

<br><br>

### 添加jar包依赖

<br>

将上述目录中 `compiler.jar` 拷贝到自己的后端工程内，并添加打包依赖：

```
<dependency>
    <groupId>com.compile</groupId>
    <artifactId>compile</artifactId>
    <version>1.0</version>
    <scope>system</scope>
    <systemPath>${project.basedir}/libs/compiler.jar</systemPath>
</dependency>
```

<br><br>

### 编写服务层接口

<br>

用于给业务模块提供组件编译功能

```
import java.io.InputStream;

public interface ICompileService {

    /**
     * 编译组件
     */
    String compileXml(InputStream inputStream, String name, int version);

    /**
     * 预览组件
     */
    byte[] previewXml(InputStream inputStream, String name, int version);
}
```

注：编译组件最终产物是二进制 `.out` 文件，用于组件发布；预览组件编译产物是二进制 `byte[]` 和 `mock数据` 一起合成测试数据，用于开发测试。

<br><br>

### 服务层接口实现

<br>

```
@Service
public class CompileServiceImpl implements ICompileService {
    @Resource
    private IConfigService configService;

    ......

    @Override
    public String compileXml(InputStream inputStream, String name, int version) {
        ViewCompilerApi viewCompiler = new ViewCompilerApi();
        viewCompiler.setConfigLoader(new XmlConfigLoader(getConfigStr()));
        byte[] result = viewCompiler.compile(inputStream, name, version);
        String outPath = OUT_PATH + name + "_" + version + ".out";
        return FileUtils.saveData(outPath, result);
    }


    @Override
    public byte[] previewXml(InputStream inputStream, String name, int version) {
        ViewCompilerApi viewCompiler = new ViewCompilerApi();
        viewCompiler.setConfigLoader(new XmlConfigLoader(getConfigStr()));
        return viewCompiler.compile(inputStream, name, version);
    }

    private String getConfigStr() {
        DynamicConfig config = configService.queryDynamicConfig();
        if (config == null) {
            return "";
        }
        return config.getData();
    }
}
```

<br><br>

### 配置文件处理

<br>

后端服务我们需要集成的文件有 `config.properties` 和 `templatelist.properties`, 现在关注一下编译接口：

```
ViewCompilerApi.compile(inputStream, name, version)
```

其中 `inputStream` 为组件源码， `name` 为组件名, `version` 为组件版本， 这些信息均在数据库组件表信息内存储(<font color="red">具体存储结构请根据自己的业务实现，此处不透漏细节</font>)，也就是我们的配置可以不关注 `templatelist.properties` 文件，因为其配置信息均在组件数据结构上包含。

重点关注一下 `config.properties` 配置的获取，查看工具类参考配置：

```
public class LocalConfigLoader implements ConfigLoader {
    private boolean buildJar;

    public LocalConfigLoader(Boolean buildJar) {
        this.buildJar = buildJar;
    }

    public InputStream getConfigResource() {
        CompilerConfig config = new CompilerConfig();
        URL u = config.getClass().getClassLoader().getResource("");
        String path = u.getPath();
        FileInputStream inputStream = null;

        try {
            inputStream = new FileInputStream(path + "config.properties");
        } catch (FileNotFoundException var6) {
            var6.printStackTrace();
        }

        return inputStream;
    }
}
```

因此我们仅需自定义配置的获取方式就行了，可以参考我的配置, 如下：

```
public class XmlConfigLoader implements ConfigManager.ConfigLoader {
    private String config;

    XmlConfigLoader(String config) {
        this.config = config;
    }

    @Override
    public InputStream getConfigResource() {
        if (config == null) {
            config = "";
        }
        return new ByteArrayInputStream(this.config.getBytes());
    }
}
```

在每次编译或预览前，先去配置中心拉取最新配置，而后设置给编译器，随时支持自定义新组件。

例如：配置可以存入数据库内，提供接口支持新增和更新

<br><br>

### 业务接入

<br>

实现上面步骤后，业务方就可以根据需要快速接入 `Tangram组件编译` 功能了。

<br><br><br><br><br>









