---
layout: post
category: cygwin
title:  "cygwin深度使用"
tags: [cygwin,shell,cmdline]
---

### 1. Cygwin是类Unix环境，基本上所有的linux命令都可应用

```
grep, sed, cat, file, iconv, xargs, head, tail
wc, hexdump, xxd
nslookup, dig, ipconfig, nc
ssh, rsync, scp, sftp
perl, awk, python, ruby, node, pip, gem, npm
urlencode, urldecode
apt-cyg, cygpath
df,
gcc, nm, objdump, ldd, make, automake
ffmpeg, lame, openssl, pandoc
mysql, sqlite, php, nginx, redis
adb, aapt, gradle, ant, apktool, jarsigner
svn, git
find, locate(contab), which, 
sshd, cron, cygrunsrv
```

### 2. Shell的一些坑

#### 2.1 文件名中含有空格的处理

1.命令行中可使用"\ "来处理，即转义的方式

2.sh脚本中变量中含有空格, 使用"$var"来处理

#### 2.2 通配符和正则表达式

1.先是通配符，再是正则表达式

2.通配符相当于在命令后面，加上符合通配的文件名

3.通配符可以用sh -x来查看

4.可以加"", 让通配效果消失

5.通配符是shell处理的; 正则表达式是命令处理的

6.通配符和正则表达式都需要命令支持

#### 2.3 乱码

1.可以使用file查看文件编码

2.使用iconv -f gbk -t utf-8基本能解决问题

3.有时还需要使用重写向， 2>&1

4.理论是无法识别出一个文件的确切编码的

#### 2.4 路径问题

1.cygwin可以使用任何windows命令/exe

2.使用windows命令时, 绝对路径使用 d:/temp模式, 如adb

3.node的环境变量NODE_PATH, 也要使用windows路径模式

4.使用cygpath可进行切换

### 3. 守护进程 (服务)

#### 3.1 sshd (terminal server)

1.在管理员模式下运行cygwin terminal

2.运行 ssh-host-config 命令, step-step

3.使用ssh user@localhost 验证, 注意界面与linux不同

#### 3.2 crond (timer server)

1.在管理员模式下运行cygwin terminal

2.运行 cron-config 命令, step-step

3.使用crontab验证

