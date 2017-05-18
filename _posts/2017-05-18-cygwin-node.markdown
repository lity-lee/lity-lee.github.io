---
layout: post
category: program
title:  "cygwin安装nodejs环境"
tags: [cygwin,nodejs]
---

### cygwin环境中使用nodejs与类Unix和纯windows环境有点复杂
1. 无法通过apt-cyg命令或者cygwin的setup.exe来安装
2. 无法通过build源代码的形式安装nodejs（官方明确提示不支持）.

### 所以只能通过变相的方式来解决
#### 1. 在下载windows对应的安装包，安装nodejs版本
#### 2. 找到安装好的nodejs, 里面有一个node.exe 复制到cygwin目录下，这个目录最好是环境变量path的一部分，建议使用这个目录，然后你就可以卸载nodejs了
```
~/bin/
```

#### 3. 安装npm, 便用官方的方式

```
curl -L https://www.npmjs.com/install.sh | sh
```
#### 4. 配置python, cygwin 的python 无法使用

```
gdb或者arm-linux-androideabi-gdb.exe
```

#### 5. 设置环境变量，否则你全局安装的module还是找不到的

```
file bin或者so
```
