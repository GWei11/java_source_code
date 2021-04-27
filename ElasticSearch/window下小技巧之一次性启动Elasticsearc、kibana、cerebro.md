在 windows 中一般每次都要启动 Elasticsearch 和 kibana，当然也可以启动 cerebro 来查看 es 的集群状态，这样每次启动多个服务很麻烦，所以可以将这些启动命令写成一个批处理命令，一次性启动。

```bash
@echo off

start cmd /k "D:\config\ELasticsearch\elasticsearch-7.12.0\bin\elasticsearch.bat"
start cmd /k "D:\config\ELasticsearch\cerebro-0.9.4\bin\cerebro.bat"
start cmd /k "D:\config\ELasticsearch\kibana-7.12.0-windows-x86_64\bin\kibana.bat"
```

