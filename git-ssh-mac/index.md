---
layout: post
title: git本地多ssh key管理
date: 2017-03-04 14:09:24
updated: 2020-03-30 14:09:24
tags:
  - ssh
categories: Git
---

之前在 mac 配了多 ssh key，一直用的很爽，然而前几天因为不当操作，导致 ssh 失效，一怒全部删除，准备重配的时候命令什么的已然忘记，还得再重新查，这记性，所以又是备忘录。

<!-- More -->

## 生成第一个 ssh key

假设从来没有配过 ssh，以下是生成默认 key

### 1.配置 git 用户名以及邮箱

如果配置了全局 git 用户名以及邮箱这一步可以跳过，也可以为每一个 key 都配置专属 git 用户名邮箱

```
$ git config user.name "你的用户名"
$ git config user.email "你的邮箱"
```

在 config 后面加上`--global`即可设置全局，一劳永逸

### 2.生成 ssh key

```
$ ssh-keygen -t rsa -C "你的邮箱"
```

然后一路回车即可，不要输入任何东西，就可以在根目录的`~/.ssh`目录下看到两个文件`id_rsa`、`id_rsa.pub`，其中 pub 后缀的文件里面的内容就是后面需要使用的 key

### 3.上传 key 到 Github

可以使用`clipcopy`命令，也可以自行打开文件复制

```
$ clipcopy < ~/.ssh/id_rsa.pub
```

复制完内容后就是到 Github 个人设置里面添加 SSH 公匙

### 4.验证是否成功

在 Github 添加公匙后，就可以验证是否链接成功

```
$ ssh -T git@github.com
```

按提示输入 yes 并稍等片刻看到以下内容就是配置成功

```
Hi 你的用户名! You've successfully authenticated, but GitHub does not provide shell access.
```

## 生成多个 ssh key

现在有很多 git 仓库，所以配置一个 ssh key 是远远不够的，所以需要配置多个 ssh key，但是继续使用上面步骤会覆盖之前配置的 Github 公匙，因为在生成的时候没有指定名称，所以这一步是关键，下面以 gitlab 实例

### 1.生成 ssh key 并指定名称

```
$ ssh-keygen -t rsa -f ~/.ssh/id_rsa.gitlab -C "你的邮箱"
```

然后继续一路回车，就可以在根目录的`~/.ssh`目录下看到两个文件`id_rsa_gitlab`、`id_rsa_gitlab.pub`

### 2.新增并配置或修改 config

```
$ touch ~/.ssh/config
```

并在 config 文件添加以下内容

```
Host gitlab.com
    IdentityFile ~/.ssh/id_rsa.gitlab
```

完整例子

```
Host github.com
    IdentityFile ~/.ssh/id_rsa.github
Host gitlab.com
    IdentityFile ~/.ssh/id_rsa.gitlab
Host git.coding.net
    IdentityFile ~/.ssh/id_rsa.coding
```

### 3.上传`id_rsa_gitlab.pub`内容到 gitlab 公匙

### 4.验证

```
$ ssh -T git@gitlab.com
```

看到以下内容表示配置成功

```
Welcome to GitLab, 你的用户名!
```

之后再配置其它 git 的 ssh key 就继续重复上述步骤就可以了，重点就是指定名称和配置 config 文件

## 遇到带端口的 git 地址

如果公司自搭 git 服务器带有端口号，那么需要配置 Port

```
Host xxx.xxx.xxx
    IdentityFile ~/.ssh/id_rsa.xxx
    Port xxxx
```
