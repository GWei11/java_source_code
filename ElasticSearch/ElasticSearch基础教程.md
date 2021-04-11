# Elasticsearch 集成 IK 分词器

## 下载插件并安装

**【注意】**：因为 Elasticsearch 不能使用 root 用户启动，所以这里也是使用启动 Elasticsearch 用户的那个用户来操作。

* 插件地址 https://github.com/medcl/elasticsearch-analysis-ik/ 

![](https://gitee.com/GWei11/picture/raw/master/20210411091917.png)

**【注意】**打开 github 地址后，不是直接从 code 那里下载，而是往下看 install 那里，有两种安装方式，如果使用第一种也就是先下载文件下来本地安装的话，需要从这个链接进去下载文件，如果是直接从 code 那里下载，安装的时候会出现如下错误

```shell
Exception in thread "main" java.nio.file.NoSuchFileException: /usr/elasticsearch/plugins/.installing-2567752034376027906/plugin-descriptor.properties
```

* 解压插件

```shell
cd /usr/elasticsearch/plugins  # 进入elasticsearch安装目录里面的plugins目录
mkdir ik # 在 plugins目录里面创建一个目录
# 然后将下载的ik分词器压缩包放到创建的 ik 目录里面
unzip elasticsearch-analysis-ik-7.3.0.zip # 解压
```

* 安装插件（当前目录在 /usr/elasticsearch/plugins/ik 中）

```shell
/usr/elasticsearch/bin/elasticsearch-plugin install elasticsearch-analysis-ik-7.3.0.zip
```

如果，不是在这个目录里面，可以使用下面的命令来进行安装

```shell
/usr/elasticsearch/bin/elasticsearch-plugin install file:///usr/elasticsearch/plugins/elasticsearch-analysis-ik-7.3.0.zip
```

* 安装完成之后，重新启动 Elasticsearch 和 Kibana 即可。