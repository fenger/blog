+++
title="Git多账号配置"
date=2020-08-07T15:33:42+08:00
tags=["git"]
categories=["Git"]
url="/2020/08/07/multi-git-account-config.html"
toc=false
+++

系统: Mac

1. 查看当前是否存在
```
ls ~/.ssh/
```
默认情况下，`id_rsa`为私钥文件，`id_rsa.pub`为公钥文件，如果有的话，证明之前生成过。

2. 生成新公钥

格式为：
```
ssh-keygen -t rsa -f ~/.ssh/文件名 -C "邮箱"
```
文件名任意，邮箱为对应网站的邮箱。
例如我要生成两个，一个对应github，一个对应gitosc，那么我需要执行：
```
ssh-keygen -t rsa -f ~/.ssh/github_id_rsa -C "xxx@xxx.com"
```

```
ssh-keygen -t rsa -f ~/.ssh/gitosc_id_rsa -C "xxx@xxx.com"
```

执行后会让输入密码（passphrase）,直接回车就行。完成后目录下会生成对应的文件，其中`*_id_rsa`为生成的私钥，`*_id_rsa.pub`为生成的公钥

3. 配置

在`~/.ssh/`目录下新建名称为`config`的文件，配置以下内容
```
Host gitee.com
HostName gitee.com
User 你的名字
IdentityFile ~/.ssh/gitosc_id_rsa

Host github.com
HostName github.com
User 你的名字
IdentityFile ~/.ssh/github_id_rsa
```

4. 测试连通

```
ssh -T git@gitee.com
ssh -T git@github.com
```

5. 取消全局账号信息
```
git config --global --unset user.name
git config --global --unset user.email
```




