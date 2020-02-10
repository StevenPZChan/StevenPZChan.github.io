---
title: Jenkins编译配置Merge before build
layout: post
categories: git
tags:
  - git
  - merge
  - build
  - Jenkins
last_modified_at: 2020-02-10T15:40:00+08:00
---
### Jenkins编译配置Merge before build
分享一下`Jenkins`编译服务配置**Merge before build**编译前先合并的坑。

---

首先`Jenkins`支持对`GitHub`上的`Pull Request`进行跟踪编译，偶然发现`Jenkins`的配置里面也支持编译前先合并。
编译前先合并是可以有效地避免PR分支增加了一些文件而没有先合并主分支，导致有可能PR合并完主分支会编译失败的情况。因此这个选项非常有用。然而设置上面并没有很顺利。

1. 首先按照`Jenkins` `Github Pull Request Builder`的帮助文档，添加一个名为**sha1**的参数，并指定编译分支为**${sha1}**，这样就可以让`Jenkins`跟踪PR的状态，并且编译PR分支
2. 然后打开额外行为里的`Merge before build`，添加一个名为**ghprbTargetBranch**的参数并设置合并分支为**${ghprbTargetBranch}**，这样就可以在编译前先合并目标分支
3. 最关键的一步，要让`Jenkins`自动地把PR分支和目标分支的代码都*fetch*下来才能成功合并，所以要设置仓库的*fetch*定义（*refspec*）为
> +refs/pull/${ghprbPullId}/*:refs/remotes/origin/pr/${ghprbPullId}/* +refs/heads/${ghprbTargetBranch}:refs/remotes/origin/${ghprbTargetBranch}

至此，**Merge before build**设置完成，当有冲突不能合并的时候会编译不通过，而合并之后编译不通过的情况也能够反映出来了。

