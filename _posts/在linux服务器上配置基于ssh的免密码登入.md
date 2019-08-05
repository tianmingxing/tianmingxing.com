---
title: 在linux服务器上配置基于ssh的免密码登入
date: 2019-08-05 15:38:00
tags:
- ssh
- linux
- 免密码登入
categories:
- linux
---

![](/images/ssh-key-auth-flow.png)

所谓“免密码”登入远程linux系统，其实并不是真的不需要“密码”，而是不再使用明文密码。我们通过利用RSA加密算法构建一个安全的SSH通道，实际使用时通过证书来双向验证彼此的身份，证书本身可以选择是否加密，如果是那最终还是要有输入密码的过程。

<!-- more -->
下面介绍如何在两台机器之间配置SSH密钥身份认证。

## 创建SSH密钥

在本地计算机上执行命令 `ssh-keygen` 并回车：

```bash
Generating public/private rsa key pair.
Enter file in which to save the key (/home/username/.ssh/id_rsa):
```

默认情况下，将在 `~/.ssh` 目录中生成两个文件 `id_rsa` 和 `id_rsa.pub`。再次回车确定使用上面的存储位置。

```bash
/home/username/.ssh/id_rsa already exists.
Overwrite (y/n)?
```

如果发现这两个文件已经存在，将提示你是否要进行覆盖，如果是首次生成是不会有这个提示的。

```bash
Created directory '/home/username/.ssh'.
Enter passphrase (empty for no passphrase):
Enter same passphrase again: 
```

上面提示你是否为生成的证书设置密码，此时根据自己的需要设置即可，如果确实不想后续连接过程中输出密码，也可以不用设置，直接回车即可。

```bash
our identification has been saved in /home/username/.ssh/id_rsa.
Your public key has been saved in /home/username/.ssh/id_rsa.pub.
The key fingerprint is:
a9:49:2e:2a:5e:33:3e:a9:de:4e:77:11:58:b6:90:26 username@remote_host
The key's randomart image is:
+--[ RSA 2048]----+
|     ..o         |
|   E o= .        |
|    o. o         |
|        ..       |
|      ..S        |
|     o o.        |
|   =o.+.         |
|. =++..          |
|o=++.            |
+-----------------+
```

此时本地证书就生成好了。

### 注意

1. 如果本地计算机是windows，可以安装git以使用git bash，在里面能够输入上面的生成命令。
2. 如果希望证书上添加邮箱等信息，可以在生成时输入命令 `ssh-keygen -t rsa -b 4096 -C "your_email@example.com"`，其中 `-b` 选择表示证书字节长度，一般值越大证书破解难度越高。

## 在远程服务器中导入公钥

有两种方法可以把本地生成的公钥导入到远程服务器上：

1. 手动复制：打开文件 `~/.ssh/id_rsa.pub` 并复制里面的内容，粘贴到远程服务器的 `~/.ssh/authorized_keys` 文件中。如果本地计算机是windows则公钥位于 `C:\Users\Administrator\.ssh`。
2. 使用ssh-copy-id：执行命令 `ssh-copy-id username@remote_host` 连接远程主机并直接复制公钥到对应文件中，在连接过程中要输入远程服务器密码。

## 使用SSH密钥对远程服务器进行身份认证

如果你已成功完成上述过程，则应该能够在没有远程帐户密码的情况下登录远程主机。

```bash
ssh username@remote_host
```

如果这是你第一次连接到此主机，可能会看到如下内容：

```bash
The authenticity of host '111.111.11.111 (111.111.11.111)' can't be established.
ECDSA key fingerprint is fd:fd:d4:f9:77:fe:73:84:e1:55:00:ad:d6:6d:22:fe.
Are you sure you want to continue connecting (yes/no)? yes
```

此时输入yes并回车即可。

## 其它

可以禁用密码登录功能，防止服务器遭暴力破解。

```bash
vim /etc/ssh/sshd_config

# 修改下面的属性为no，默认为yes
PasswordAuthentication no
```

然后重启ssh服务即可。

```bash
service ssh restart
service sshd restart
```
