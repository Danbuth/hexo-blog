title: 搭建git-gitosis私有仓库
author: Danbu
tags:
  - git
  - gitosis
categories:
  - 运维
date: 2018-03-28 19:15:00
---
# 搭建Git(Gitosis)私有仓库

> 本文以 centos6.8 为例，搭建Git私有仓库，并辅以Gitosis对用户公钥和分组进行控制，并在centos7.4上实践

## 1. 搭建Git

### 1.1 安装依赖库

```shell
yum install curl-devel expat-devel gettext-devel openssl-devel zlib-devel
yum install  gcc perl-ExtUtils-MakeMaker
```

### 1.2 卸载旧版git

> 假如原先有用yum安装过git，则需要先卸载一下

```shell
yum remove git
```


### 1.3 下载源码

> 查找git版本可以到 [https://www.kernel.org/pub/software/scm/git/](https://www.kernel.org/pub/software/scm/git/)下查看git的版本号自行选择下载

![paste image](http://p657mjf4s.bkt.clouddn.com/1522236983846mwy7im1n.png?imageslim)

>这里我选择下载git-2.16.3.tar.gz

```shell
cd /usr/local/src
wget https://www.kernel.org/pub/software/scm/git/git-2.16.3.tar.gz

```

### 1.4 解压、编译和安装

```shell
tar -zvxf git-2.16.3.tar.gz
cd git-2.16.3
make prefix=/usr/local/git all
make prefix=/usr/local/git install
```
### 1.5 将git目录加入PATH

```shell
echo 'export PATH=$PATH:/usr/local/git/bin' >> /etc/bashrc
source /etc/bashrc
```
> 安装成功后就可以查看到git版本了。

![paste image](http://p657mjf4s.bkt.clouddn.com/1522237215431kwx360yl.png?imageslim)

### 1.6 创建git账号并设置密码

> 此用户将作为git仓库的管理员

```shell
useradd -m git
passwd git
```

## 2 Gitosis 配置

> Gitosis 就是一套用来管理 authorized_keys文件和实现简单连接限制的脚本，它是Python开发的，所以要保证Python和Python setuptools提前安装好。

### 2.1 git配置准备


	# 创建git仓库存储目录
	mkdir -p /home/git/repositories
	# 设置git仓库权限
	chown -R git:git /home/git/repositories
	chmod -R 755 /home/git/repositories


> 创建gitosis管理员(git)的个人公钥和私钥。首先su到git用户下面

	su - git

> 默认生成2048位，可以提高到4096位来提高安全级别，通过下面的命令创建公钥和私钥

	ssh-keygen -t rsa -b 4096

	# 公私钥保存路径，回车即可
	Enter file in which to save the key (/home/git/.ssh/id_rsa):
	# 提交密码即使用到ssh时需要输入此密码，无特殊要求回车即可 
	Enter passphrase (empty for no passphrase):

> 默认情况下，公钥和私钥会保存在~/.ssh目录下，如下所示：

		
	id_rsa  id_rsa.pub  known_hosts

> 初始化全局设置

```shell
git config --global user.name "git"
git config --global user.email "git@test.com"
```

### 2.2 获取并安装gitosis
> 注：gitosis脚本在ssh.py(文件位于/gitosis)中对用户名称做了正则匹配
> <p>This regular expression (regex) restricts the allowed characters in user/key names, as well as restricting the length indirectly. The above regex can be explained with the following rules.</p>

* The user/key name can be just a user name or in an email-like format (user-part@domain-part.com)
* The user part has to start with a letter character “a-zA-Z”.
* The user part can then contain 0 or more of letters “a-zA-Z”, digits “0-9), “_”, “.” and “-” character.
* The domain part needs to start with a letter character “a-zA-Z”.
* The domain part then can contain 0 or more of letters “a-zA-Z”, digits “0-9), “.” and “-” character.
* The user part needs to be 1 character minimum.
* The domain part is optional but if present needs to be separated from the user part by “@”.
![paste image](http://p657mjf4s.bkt.clouddn.com/1522284618377dgux4crr.png?imageslim)

> 此正则会导致用户名以数字开头时提示为不安全用户而导致用户无法正确加入

```shell
(root)
cd /tmp
git clone https://github.com/res0nat0r/gitosis.git
sudo python setup.py install
```

### 2.3 管理gitosis

> 首先要用之前创建的管理员公钥初始化gitosis。

```shell
su - git
gitosis-init < ~/.ssh/id_rsa.pub
chmod 755 /home/git/repositories/gitosis-admin.git/hooks/post-update
```

> git clone gitosis的控制目录gitosis-admin

```shell
mkdir myrepo
cd myrepo
git clone git@localhost:gitosis-admin.git
```

> 这样gitosis的控制目录gitosis-admin就被clone下了。里面结构如下：

```shell
cd gitosis-admin/
$ ls -R
.:
gitosis.conf  keydir
 
./keydir:
git@my-server.pub
```
> 其中：
* gitosis.conf 文件是用来设置用户、仓库和权限的配置文件。
* keydir 目录则是保存所有具有访问权限用户公钥的地方，允许访问gitosis的用户的公钥都保存在这里。

```shell
$ cat gitosis.conf 
[gitosis]
 
[group gitosis-admin]
members = git@my-server
writable = gitosis-admin
```
> 这表明了用户git（初始化 Gitosis 公钥的拥有者）是拥有唯一管理 gitosis-admin这个仓库的权限。

> 下面我们可以新增一个项目。为此我们要建立一个名为dev的组(group)，以及他们拥有写权限的项目。并允许’debugo’这个用户有权利读写’proj1’这个新项目：

```shell
[group dev]
members = debugo@my-server
writable = proj1
```

> debugo虽然已经添加到了配置文件中，但它的公钥还没有被gitosis获知。所以我们要将debugo的公钥改名为debugo.pub，拷贝到keydir中。

```shell
cp /tmp/id_rsa.pub keydir/debugo@my-server.pub
git add keydir/debugo@my-server.pub
```

> 修改完之后，提交 gitosis-admin 里的改动，并push到服务器使其生效。
```shell
git commit -am 'add new group'
git push origin master
```

### 2.4 git项目管理

> 在debugo用户目录下，首先先初始化用户信息。
```shell
git config --golbal user.name debugo
git config --golbal user.email debugo@
```
> debugo用户已经有对proj1这个项目有读写权限了，但是proj1这个项目并没有任何内容。下面我们首先初始化一个本地项目。

```shell
mkdir proj1
cd proj1/
git init
# Initialized empty Git repository in /home/debugo/proj1/.git/
# 添加几个新文件,并将所有文件列入索引中。
touch main.py hello.py
git add .
# 添加文件内容并进行一次提交
echo 'print "hello world"' > hello.py 
git commit -a -m "origin"
[master (root-commit) b9ecf8e] origin
 2 files changed, 1 insertion(+)
 create mode 100644 hello.py
 create mode 100644 main.py
# 检查该git仓库状态，保证它是最新的。
git status
On branch master
nothing to commit, working directory clean
# 下面push到远程的server上。
$ git remote add origin git@localhost:proj1.git
$ git push origin master
```

## 3 其他

### 3.1 获取用户公钥

> windows 下的用户公钥一般存放于登录用户的.ssh文件夹下
![paste image](http://p657mjf4s.bkt.clouddn.com/1522286399602dq8s0o6q.png?imageslim)

![paste image](http://p657mjf4s.bkt.clouddn.com/1522286414362xjlc6vaw.png?imageslim)

> linux 下的用户公钥一般存放于登录用户家目录下的.ssh文件夹下

	cd  ~/.ssh

### 3.2 gitosis 仓库用户权限管理
![paste image](http://p657mjf4s.bkt.clouddn.com/15222866252308sugfhob.png?imageslim)

如图所示：
<p>git托管了三个仓库：</p>
* gitosis-admin
* devProj
* testProj
<p>创建了四个组</p>
* gitosis-admin
* dev
* test
* testRead

<p>表示gitosis-admin组下的用户对gitosis-admin仓库有读写权限</p>
<p>表示dev组下的用户对devProj仓库有读写权限</p>
<p>表示test组下的用户对testProj仓库有读写权限</p>
<p>表示testRead组下的用户对testProj仓库有读权限</p>