---
layout:     post
title:      "Gerrit服务搭建"
subtitle:   "百人开发以下"
date:       2017-08-03 09:40:45
author:     "faxiang1230"
header-img: "img/post-bg-2015.jpg"
catalog: true
tags:
  - gerrit
  - 服务器
---

# 为Openthos配置Gerrit服务

## 配置gerrit
### 1.下载gerrit的jar包
下载链接 <http://download.csdn.net/download/stwstw0123/9044005>
### 2.提前安装一些工具:mysql,apache2-utils,nginx
gerrit将用户信息保存到数据库中,所以需要选择一个数据库,这里暂且使用mysql;
并且需要提前新建数据库  
```
$ sudo apt-get install mysql-server
$ mysql -u root -p
>create database reviewdb;
>set global explicit_defaults_for_timestamp=1;   fix issue[http://www.jianshu.com/p/154960f71d11]
```
添加用户名和密码时需要使用如下的操作:
```
sudo apt install apache2-utils
htpasswd /home/gerrit/gerrit/etc/htpasswd.conf admin
```
### 3.安装gerrit:
具体的选项可以看一下网上的,可以参考一下这个<http://blog.csdn.net/stwstw0123/article/details/47259425>;
最后生成的配置文件请看:
```
linux@linux-T:~/mount$ java -jar ../Downloads/gerrit-2.11.3.war  init -d ./review-site

cat ./review-site/etc/gerrit.config
Configure file:
[gerrit]
        basePath = git
        canonicalWebUrl = http://localhost:8081
[database]
        type = mysql
        hostname = localhost
        database = reviewdb
        username = root
[index]
        type = LUCENE
[auth]
        type = HTTP
[sendemail]
        enable = true
        smtpServer = smtp.163.com
        smtpServerPort = 25
        smtpEncryption =
        smtpUser = wangjianxing5210@163.com
        smtpPass = xxxxx
        from = wangjianxing5210@163.com
[container]
        user = linux
        javaHome = /usr/lib/jvm/java-8-openjdk-amd64/jre
[sshd]
        listenAddress = *:29418
[httpd]
        listenUrl = proxy-http://*:8083
[cache]
        directory = cache
[gitweb]
        cgi = /usr/lib/cgi-bin/gitweb.cgi
```
### 4.配置nginx的反向代理
```
sudo vim /etc/nginx/sites-available/mydefault.vhost
server {
  listen *:80;
  server_name localhost;
  allow   all;
  deny    all;
  auth_basic "ZJC INC. Review System Login";
  auth_basic_user_file /home/linux/mount/review-site/etc/htpasswd.conf;

  location / {
    proxy_pass  http://localhost:8083;
  }
}
ln -s /etc/nginx/sites-available/mydefault.vhost /etc/nginx/sites-enabled/mydefault.vhost
```
重新启动nginx和gerrit服务应该都会好用了
```
review-site/bin/gerrit.sh start
service nginx restart
```
### 5.apache反向代理
因为历史原因，所有的服务都使用apache2来作为服务端，所以我也只能延续使用apache来代理gerrit服务;
那上面review-site/etc/gerrit.config就有些许变化,`请将代码库路径，邮箱，域名等替换为自己的`
```
cat gerrit/review-site/etc/gerrit.config
[gerrit]
	basePath = /path/to/your/code
	canonicalWebUrl = http://域名:8083/g/
[database]
	type = h2
	database = db/ReviewDB
[index]
	type = LUCENE
[auth]
	type = HTTP
[sendemail]
	enable = true
	smtpServer = smtp.126.com
	smtpServerPort = 25
	smtpUser = 邮箱
	smtpPass = 密码
	from = 邮箱
[container]
	user = openthos
	javaHome = /usr/lib/jvm/java-7-openjdk-amd64/jre
[sshd]
	listenAddress = *:29418
[httpd]
	listenUrl = proxy-http://域名:8083/g/
[cache]
	directory = cache
```
apache2的配置文件也需要增加反向代理
```
cat /etc/apache2/sites-enabled/000-default.conf
<VirtualHost *:80>
	ServerName dev.openthos.org
  ProxyRequests Off
  ProxyVia Off
  ProxyPreserveHost On

  <Proxy *>
    Require all granted
  </Proxy>
  <Location /g/login/>
		AuthType Basic
		AuthName "OPENTHOS Gerrit"
		Require valid-user
		AuthBasicProvider file
		AuthUserFile /home/gerrit/review-site/etc/htpasswd.conf
	</Location>
  AllowEncodedSlashes On
  ProxyPass /g/ http://域名:8083/g/ nocanon

</VirtualHost>
```
重启gerrit服务`review-site/bin/gerrit.sh restart`

重启apache2服务`service apache2 restart`

访问`http://dev.openthos.org/g/`就能看到gerrit首页了
### 6.添加新的gerrit账号
1.登陆gerrit服务器，并切换到gerrit:`su gerrit`  
创建新账号:  
`htpasswd /home/gerrit/gerrit/etc/htpasswd.conf 新的账号`
2.告知用户的账户密码，修改`Contact Information`
其中在发送给你的邮件中，链接可能如下:
```
http://dev.openthos.org:8083/g/#/VE/r7NPtb5QA8cD1NEp2rdSM8ib327GsATJqDTGrw==$MTAwMDAwMjpmYXhpYW5nMTIzMEBzaW5hLmNu
```
请复制该链接并减去`:8083`变更为如下链接:
```
http://dev.openthos.org/g/#/VE/r7NPtb5QA8cD1NEp2rdSM8ib327GsATJqDTGrw==$MTAwMDAwMjpmYXhpYW5nMTIzMEBzaW5hLmNu
```
复制该链接到浏览器完成信息更改.
### 7.权限限制
1.创建新的Group:`developer,test,volunteer,team leader`

2.设置不同group的权限
![images](./imaegs/gerrit-access.png)  
# Gerrit管理repo
## 1.添加仓库到gerrit中
从上面的配置就可以看出,路径是git,即review-site/git/目录下,所有理论上我们就只需要将库拷贝到
review-site/git/

这里我们可以重新init和sync
```
repo init -u git://xxxx -b branch --mirror  必须要是使用mirror方式
repo sync
```
重启gerrit我们就可以登陆gerrit看到  
*Projects->List*
## 2.配置仓库的manifest
原来我们没有使用gerrit服务,所以现在需要指定提交的地址,如下所示:
```
   <remote  name="aosp"
-           fetch="git://192.168.0.185/lollipop-x86/" />
+           fetch="git://192.168.0.185/lollipop-x86/"
+          review="http://域名/g/" />
   <remote  name="x86"
-           fetch="." />
+           fetch="."
+          review="http://域名/g/" />
```
## 开发需要的配置
### 1.配置~/.gitconfig
```
[review "http://域名/g/"]
    username = MYNAME
```
### 2.新的上传操作
- 1.拷贝`~/.ssh/id_rsa.pub`到gerrit中的`SSH Public Keys`，具体怎么添加公钥可以自行百度
- 2.repo start newbranch .(if you still use `git checkout -b branchname`,you will see 'no branches ready for upload' when `repo upload .`)  
- 3.git add;git commit  
- 4.repo upload .

# 遇到的错误
## 1.反向代理的认证问题

## 2.repo upload 失败
```
linux@linux-T:~/mount/mul/build$ repo upload .
Upload project build/ to remote branch refs/heads/multiwindow:
  branch 123 ( 1 commit, Wed Feb 8 19:26:03 2017 +0800):
         dfc4275f test
to http://localhost (y/N)? y
Unauthorized
User: linux     
Password:
Unable to negotiate with 127.0.0.1 port 29418: no matching key exchange method found. Their offer: diffie-hellman-group1-sha1
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

----------------------------------------------------------------------
[FAILED] build/          123             (Upload failed)
```
### 1.ask you to input 'user' and 'password'
1.在~/.gitconfig中添加配置
```
[review "http://域名/g/"]
	username = admin
```
并拷贝公钥到admin账号的ssh-keys中

### 2.upload failed
```
Unable to negotiate with 127.0.0.1 port 29418: no matching key exchange method found. Their offer: diffie-hellman-group1-sha1
fatal: Could not read from remote repository.
vi ~/.ssh/config
Host localhost(这是服务器地址)
    KexAlgorithms +diffie-hellman-group1-sha1
```
错误提示改变为:
```
linux@linux-T:~/mount/mul/build$ repo upload .
Upload project build/ to remote branch refs/heads/multiwindow:
  branch 123 ( 1 commit, Wed Feb 8 19:26:03 2017 +0800):
         dfc4275f test
to http://localhost (y/N)? y
Unauthorized
User: linux
Password:
sign_and_send_pubkey: signing failed: agent refused operation
Permission denied (publickey).
fatal: Could not read from remote repository.

Please make sure you have the correct access rights
and the repository exists.

----------------------------------------------------------------------
[FAILED] build/          123             (Upload failed)
```
执行如下操作:
```
linux@linux-T:~/mount/mul/build$ eval "$(ssh-agent -s)"
Agent pid 2433
linux@linux-T:~/mount/mul/build$ ssh-add
Identity added: /home/linux/.ssh/id_rsa (/home/linux/.ssh/id_rsa)
```
然后就修复好了
```
linux@linux-T:~/mount/mul/build$ cat ~/.gitconfig
[user]
	email = wangjianxing5210@163.com
	name = linux
[color]
	ui = auto
[review "http://域名/g/"]
	username = admin
```
### 3.缺乏Change-Id信息
```
root@0c7fa77b0dfa:~/test/multiwindow/art# git push ssh://admin@192.168.0.84:29418/multiwindow/platform/art HEAD:refs/for/multiwindow
Counting objects: 7, done.
Delta compression using up to 48 threads.
Compressing objects: 100% (2/2), done.
Writing objects: 100% (2/2), 299 bytes | 0 bytes/s, done.
Total 2 (delta 1), reused 0 (delta 0)
remote: Resolving deltas: 100% (1/1)
remote: Processing changes: refs: 1, done    
remote: ERROR: missing Change-Id in commit message footer
remote:
remote: Hint: To automatically insert Change-Id, install the hook:
remote:   gitdir=$(git rev-parse --git-dir); scp -p -P 29418 admin@192.168.0.84:hooks/commit-msg ${gitdir}/hooks/
remote: And then amend the commit:
remote:   git commit --amend
remote:
To ssh://admin@192.168.0.84:29418/multiwindow/platform/art
 ! [remote rejected] HEAD -> refs/for/multiwindow (missing Change-Id in commit message footer)
error: failed to push some refs to 'ssh://admin@192.168.0.84:29418/multiwindow/platform/art'
```
里面已经给了提示信息了，只需要按着做一遍就可以
```
gitdir=$(git rev-parse --git-dir); scp -p -P 29418 admin@192.168.0.84:hooks/commit-msg ${gitdir}/hooks/
git commit --amend
```
