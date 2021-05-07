# 下载源代码

[Elasticsearch的github地址](https://github.com/elastic/elasticsearch)

在 tags 里面选择 7.12.1 版本

下载完成之后导入 idea 中，需要注意的是 Elasticsearch 主要使用 gradle 来进行编译。

idea 中导入源码之后出现如下错误：（这个错误可以在导入进来之后先不编译，先按照下面的方法进行修改，然后在进行编译，这样可以节省时间）

![](https://gitee.com/GWei11/picture/raw/master/20210429071320.png)

从上面错误能看到两个信息

* 编译的时候并没有使用我们配置在 idea 中的 gradle 版本，而是去重新在网上下载了 gradle-6.8.2-all.zip , 所以对于这个问题我们可以自己去下载对应的 gradle版本然后放在 Elasticsearch 源码目录中的  `grdle\wrapper` 目录下面（和gradle-wrapper.properties 文件在同一个目录）

* 再将 gradle-wrapper.properties 文件的内容修改成如下所示

```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
#distributionUrl=https\://services.gradle.org/distributions/gradle-6.8.2-all.zip
distributionUrl=gradle-6.8.2-all.zip  # 第一个修改的地方
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
#distributionSha256Sum=1433372d903ffba27496f8d5af24265310d2da0d78bf6b4e5138831d4fe066e9   # 这行信息需要你注释掉
```

然后刷新 gradle 重新进行编译，发现出现如下所示的错误，者提示我们 jdk 需要使用 15 版本或者更新的版本

【注意】：最好是按照提示使用 jdk15， 我使用过 jdk16 的版本，但是发现还是报错，最后还是使用 15 版本的就可以编译成功。

![](https://gitee.com/GWei11/picture/raw/master/20210429073237.png)

安装 jdk15 之后是没有 jre 的，此时的解决方式如下所示

* 首先进入安装了 jdk15 的目录里面（就是安装根目录就好）
* 然后执行下面的命令

```shell
bin\jlink.exe --module-path jmods --add-modules java.desktop --output jre
```

执行完上面的命令即可。

