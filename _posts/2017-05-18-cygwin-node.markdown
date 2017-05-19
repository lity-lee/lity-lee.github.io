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

#### 4. 设置环境变量，否则你全局安装的module还是找不到的

```
export NODE_PATH=D:/cygwin64/home/USERNAME/bin/node_modules
```
注意这里的路径必须是windown可识别的路径，因为必须node.exe是纯window版本的，如果不太清楚怎么得到这样的路径，可以使用cygpath命令
```
cygpath.exe -am bin/node_modules
```

至此，cygwin平台下的nodejs环境已经搭建完成。
#### 5. 配置python, cygwin 的python 无法使用
这种情况稍微有点特别，在使用npm安装个别module时（比如：js-dom）, 会出现错误，比如找不到python，这时需要配置这步了， 同样python的路径是windown可识别的路径，原因和第4步一样。
```
cygpath -am `which python2.7.exe`
npm config set python D:/cygwin64/bin/python2.7.exe
```
但是还是报出莫名其妙的错误，比如找到不module。
```
ImportError: No module named gyp
```
具体原因还没有找到，可能还是与路径相关。网上有一种解决方案，安装一个windows版的python，虽然我不太喜欢这样子，但它确实解决了问题。
```
npm install --global --production windows-build-tools
npm config set python "C:\Users\USERNAME\.windows-build-tools\python27\python.exe"
```
