# 本地没有分支，远程有分支，如何在本地创建分支并且和远程对应分支关联？

```shell
git checkout --track origin/branch_name
```

# 本地和远程都有分支，但是没有关联关系，创建关联关系

## 第一种方式

```shell
git branch --set-upstream-to=origin/dev dev
```

上述内容是将本地 dev 分支和远程 dev 分支创建关联关系。

## 第二种方式

```shell
 git push --set-upstream origin dev
```

在第一次推送的同时建立联系，上面的命令是将当面所在本地推送到远程的 dev 分支（当然，最好让本地和远程分支名称保持一致），并且和远程的 dev 分支建立联系。