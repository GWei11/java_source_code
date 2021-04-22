下载项目 [rocketmq-externals](https://github.com/apache/incubator-rocketmq-externals)

* 将上面地址中的 git 项目下载下来
* 进入 rocketmq-console 项目中， 修改该项目的 application.properties 文件，修改下面的地址

```properties
# 在配置文件中，将后面的localhost:9876 换成对应的 namesrv 地址
rocketmq.config.namesrvAddr=localhost:9876
```

* 使用 maven 编译项目

![](https://gitee.com/GWei11/picture/raw/master/20210420063001.png)

```java
mvn clean package -Dmaven.test.skip=true
```

* 将编译后的 jar 包（jar包位置默认是在 你配置的 maven 仓库中）上传到服务器，当然如果你不是使用 centos，那就不用上传了，在本地运行也是一样
* 运行该 jar 包

![](https://gitee.com/GWei11/picture/raw/master/20210420063238.png)

1. 在 jar 包所有目录运行下面的命令

```shell
nohup java -jar rocketmq-console-ng-2.0.0.jar --server.port=8090 > log.out &
```

2. 使用 --server.port=8090 是指定端口，其实在 application.properties 中默认指定了是 8080，但是如果和别的端口冲突了，那也可以在运行的时候按照上述方式重新指定一下。
3. 最后面的 & 表示在后台运行
4. nohup 表示关闭当前窗口的时候该项目也不会被关闭
5. `> log.out` 表示将日志输出到这个文件里面，而不是显示在屏幕中

* 最后在浏览器中输入 ip:8090 就可以访问控制台了