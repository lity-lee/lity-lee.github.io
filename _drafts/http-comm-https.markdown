---
layout: post
category: mlai
title:  "python学习和应用"
tags: [https,python,rsa,pycrypto‎]
---

在客户端到服务端的传输中，需要实现一套安全有效的加密方式。这里在别人的基础了介绍一套解决方案。原来是用java实现的，现在改用python实现，主要是验证目的，可以在不调试代码的情况下通过抓包来解密数据，便于快速定位问题。https在安全性上没有问题，理论上是无法破解的，但在获取证书环节，有可能被做“手脚”，如中间人攻击。https相对http效率不高，所以我们基于http实现一个安全的加密方式。之所以用python实现，是因为python是一个高层次的结合了解释性、编译性、互动性和面向对象的脚本语言，且类库非常丰富。

<!-- more -->

### python基础知识

这虽然不是我第一次写python脚本，却还没有系统地学习过python语言，都是在使用中慢慢积累语言知识，这里介绍相对java语言python语言的一些特别之处。

1\. 代码块不使用大括号，使用":"和缩进来表示
```
        if 1 == len(key):
            key = '-' + key
        else:
            key = '--' + key
        for name, value in options:
            if key == name:
                return True
        return False
```

2\. 弱类型脚本语言

弱类型语言中变量可以直接使用，而不用先声明再使用。
弱类型有一个好处，就是一个变量可以引用不同的实际类型，比如：
```
    i = 0
    while i < len(args):
        data = args[i]
        data = data.encode('utf-8')
        mc = MobComm(options)
        data = mc.encrypt(data)
        data = base64.b64encode(data)
        data = str(data, "utf-8")
        print(data)
        i = i + 1
```

这样，我就可以仅使用一个data变量，分别引用str, bytearray类型，因为他们在概念都表示要处理的数据。

不过这样，在编写代码时有点不好，因为运行前不知道data的实际类型，所以编写“data.”的提示，要么不提示，要么提示的不正确。

3\. 支持面向对象，不支持函数重载

python语言是支持面向对象特性的，但与java在支持面向对象语法有很多差异。因为是弱类型，所以不支持函数重载。有人说也支持函数“重载”，他说的重载是参数默认值的表现。

```
    @staticmethod
    def intFromString(value, base=16):
        return int(value, base)
```

4\. 逻辑运算符

python的逻辑运算符不是一般其它的“&&，||， !”, 而是“and，or，not”。
```
        if ArrayUtil.array_key_exists('e', options) or ArrayUtil.array_key_exists('encrypt', options):
            if not ArrayUtil.array_key_exists('keysize', options):
                print("加密时必须指定keysize.")
                help()
                exit(1)
            if not ArrayUtil.array_key_exists('pubkey', options):
                print("加密时必须指定私钥.")
                help()
                exit(1)
            if not ArrayUtil.array_key_exists('modulue', options):
                print("加密时必须指定模数.")
                help()
                exit(1)
            if not len(ArrayUtil.array_value("modulue", options)) * 4 == int(ArrayUtil.array_value("keysize", options)):
                print("模数和keysize不对称, 可能有一个是错误的.")
                help()
                exit(1)
            if ArrayUtil.array_key_exists("aeskey", options) and len(ArrayUtil.array_value("aeskey", options)) != 32:
                print("aeskey必须是32位16进制数")
                help()
                exit(1)
            mainEncrypt(options, args)
```

5\. int计算无溢出，天生支持大数运算

python int类型没有大小限制，不像java语言占位4个字节，所以在大数运算时更有“优势”。

```
    def encryptBlock(self, data, start, length, blockSize):
        source = bytearray(blockSize)
        source[0] = 1
        ArrayUtil.copy(IntUtil.intToBytes(length, 4), 0, source, 1, 4)

        dstPos = len(source) - length
        ArrayUtil.copy(data, start, source, dstPos, length)

        message = IntUtil.intFromBytes(source)
        message = pow(message, self.rsaKey, self.modulue)

        message = IntUtil.intToBytes(message, 0)
        return message
```

6\. python版本不兼容

python现在主流3.x和2.x版本，但两个版本不兼容，语法级别就不兼容。比如print在2.7版本表示一种语法结构，可以写成“print data”；但在3.x版本以后print是一个函数，只能写成“print(data), 写成“print data”是运行不过的。

<p><font color="red">所以在编写python代码时要先确定一下版本。</font></p>

### getopt使用



一般传递参数
```
cmd 打包参数 参磊肿 磊在在有
```
更改参数顺序

命令行参数解析工具， 
```

```

### aes加密

aes padding


### rsa加密

m
mobrsa




## 仿https的http安全传输

传输base64加密后的字符串

[]byte 

0-3前四个字节是长度，表示aeskey占用的字节数
4-ttt aeskey加密后的数据
ttt-ttt+3, aes加密后的数据