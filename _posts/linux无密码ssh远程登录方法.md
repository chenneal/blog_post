---
title: linux无密码ssh远程登录方法
date: 2017-01-08 17:03:14

tags:
- Linux
- ssh

categories: 
- Linux

---

Linux下的ssh命令设计的有些奇怪，特别是当你自动化的跑实验需要通过ssh发送命令的时候，每次输入密码肯定是非常恼人的事情，特别是ssh无法使用密码作为参数去登录，必须通过手工输入非常让人恼火。

首先提醒一下， 不要使用sshpass，非常难用而且很多 bug 。

最简单的方法就是预先「打通」两个机器之间的回路，以后就不需要再输入密码了，具体步骤如下：

<!--more-->

假设我们希望机器A无密码访问机器B，用户皆为「root」，注意只能单向，只能A->B，B->A是不行的。

### 在A机器生成ssh公钥

```bash
ssh-keygen -t rsa
```

公钥放在 /root/.ssh/ 下面

### 将A的ssh公钥发送给B机器

```bash
scp /root/.ssh/id_rsa.pub root@ip_B:/root/.ssh/A.id_rsa.pub
```

### 导入A的ssh公钥到B机器的授权秘钥

```bash
cd /root/.ssh/
cat A.id_rsa.pub > authorized_keys
```

通过上面几个步骤，以后就可以通过

```bash
ssh -f -p 22 root@192.168.0.10 'ls'
```
直接输入命令而不需要手工输入代码。