---
title: SSH免密登录
layout: post
categories: ssh
tags:
  - ssh
last_modified_at: 2020-02-10T15:40:00+08:00
---
### SSH免密登录
分享一下如何使`SSH`免密登录和可能的坑

---

按如下步骤进行&排查：
1. 在目标机器上进行`sshd`服务的设置，编辑**/etc/ssh/sshd_config**，打开这两个选项：
```shell
PubkeyAuthentication yes
RSAAuthentication yes
```
2. 重启`sshd`服务：
```shell
sudo systemctl restart sshd
# or
sudo service sshd restart
```
3. 将本机的公钥加入目标机器的**authorized_keys**里：
```shell
ssh-copy-id [-i [identity_file]] [user@]hostname
```
4. 设置目标机器的各项权限：
```shell
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
chmod 700 ~
```

其中我遇到的最大的坑是目标机器的用户家目录权限一定不能是**777**！最高权限只能是**775**，而且我试了**702**也是不成功的，也就是说要使得`SSH`可以免密登录，不能让*others*有对家目录的写权限！

