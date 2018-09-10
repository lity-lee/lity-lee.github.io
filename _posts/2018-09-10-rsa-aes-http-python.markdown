---
layout: post
category: mlai
title:  "一套基于http的安全传输解决方案"
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

这里我就可以仅使用一个data变量，分别引用str, bytearray类型，因为他们在概念都表示要处理的数据。

不过这里在编写代码时有点不好，因为运行前不知道data的实际类型，所以编写“data.”的提示，要么不提示，要么提示的不正确。

3\. 支持面向对象，不支持函数重载

python语言是支持面向对象特性的，但与java在支持面向对象语法有很多差异。因为是弱类型，所以不支持函数重载。有人说也支持函数“重载”，他说的重载是参数默认值的表现。

```
    @staticmethod
    def intFromString(value, base=16):
        return int(value, base)
```

4\. 逻辑运算符

python的逻辑运算符不是一般其它语言的“&&，||， !”, 而是“and，or，not”。
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

<p><font color="red">所以在编写python代码前要先确定一下版本。</font></p>

### getopt使用

作用是解析命令行参数。在这个脚本里大概实现这样的参数传递：

```
加密/解密公共库网络通信协议的数据, 若没有DATA将从输入流中读取.
Options:
  -h/--help     print help. 
  -d/--decrypt      decrypt, not encrypt.
  -e/--encrypt      encrypt, not decrypt.
  --/--keysize      rsa key length(most be 1024).
  --/--modulue      rsa 加密的模数(加密/解密)都需要提供.
  --/--pubkey       rsa 加密公钥, 仅加密时需要提供.
  --/--prikey       rsa 解密私钥, 仅解密时需要提供.
  --/--aeskey       aes key, 加密时若指定aes key时需要(一般不需要指定).
```

补充之前提示到通配符需要命令支持，在getopt帮忙下实现“多”操作是很容易的。比如像这样：
```
cat "*.java" | wc -l
```

把当前目录下的所有java文件内容串接起来统计行数。通配“*.java”，汇聚java文件，通过getopt返回的args参数，就是所有java文件的数组。

```
options, args = getopt.getopt(sys.argv[1:], 'hde', ['help', 'decrypt', 'encrypt', 'keysize=', 'pubkey=', 'prikey=', 'modulue=', 'aeskey='])
```

### aes加密

python用aes加密，需要使用类库pycrypto‎。若使用类unix系统使用安装pycrypto‎很容易：

```
pip3 install pycrypto‎
```

在cygwin平台上安装pycrypto‎也是上面的方法，这里需要安装编译器，如果安装过了就可以顺利安装成功。但如果在windows平台安装pycrypto‎就比较麻烦啦，因为需要安装VS，这里就不展开了。

这里推荐一个[python开发环境]({{ site.url }}{{ site.baseurl }}{% post_url 2018-08-24-python-cygwin-debugger %})

aes填充，aes加密是分块的，每一块是16个字节，这里的填充，填的长度是16 减去总长度除以16的余数，如果刚好除尽就是16。

```
    def padding(self, data):
        BS = Crypto.Cipher.AES.block_size
        BS = (BS - len(data) % BS)
        data += (BS * chr(BS)).encode()
        return data
```

以前认为如果数据块长度刚好是16的倍数，就不需要填充，然而在unpad时却发现，无法确实解密后数据是否填充过，所以每次都填充。

从上面代码可以看出填充的值，每个字节是一样的，都是填充字节的个数。后来想想如果填充的内容是0，那么unpad该怎么确定填充的长度呢？应该确定不了吧，如果无法确实那zeropadding填充填充的内容是什么？这里是unpad代码：

```
    def unpad(self, data):
        return data[:-ord(data[len(data)-1:])]
```

### rsa加密

rsa算法的数字知识就不普及了，在这里我们知道三个重要的大数，一个是私钥、一个是公钥、一个是模数，当然还有不太重要的key长度，其实就是模数的长度。私钥和模数用于解密数据，公钥和模数用于加密数据。

由于python的int类型是支持大数运算的，那加解密实现很简单：

```
// 加密
message = pow(message, self.pubKey, self.modulue)

// 解密
message = pow(message, self.priKey, self.modulue)
```


rsa加密也是分块的，我们设置每块是128个字节，数据最大长度是117。前5位可以叫头区，数据存放在尾部，中间空隙全填充0，具体参考下表：

|起始地址|终止地址(不包含)|说明|长度(字节)|
|0|1|校验位，永远是1|1|
|1|1 + 4|数据长度(大头)|4|
|1 + 4|数据长度|0填充|一定存在|
|128 - 数据长度|128|数据|数据长度|

加密算法代码如下：
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

依据数据定义规范，也很容易写出解密算法：
```
    def decryptBlock(self, data):
        message = IntUtil.intFromBytes(data)
        message = pow(message, self.rsaKey, self.modulue)
        message = IntUtil.intToBytes(message, 0)
        length = IntUtil.intFromBytes(message[1:5])
        start = len(message) - length
        return message[start:start + length]
```


## 实现http安全传输方案

我们知道rsa加密的效率相对aes要低，但有aes加密替代不了的特性，非对称加密。所以这里rsa仅仅是加密/解密aeskey的， 数据部分还是使用aes加解密。

如果模拟网络分层思想，大概可以分为：

|base64|网络层|
|加密aeskey长度+加密aeskey长度+加密数据长度+加密数据|数据层|

包装的数据层详细规范定义如下表：

|起始地址|终止地址(不包含)|说明|长度|
|0|4|aeskey密文长度(大头)|4|
|4|4 + aeskey密文长度|aeskey密文数据|不定|
|接上|接上 + 4|数据密文长度|4|
|拉上|接上 + 数据密文长度|数据|不定|

加密代码实现如下：

```
    def encrypt(self, data):
        rsa = Rsa(ArrayUtil.array_value("pubkey", self.options), ArrayUtil.array_value("modulue", self.options), ArrayUtil.array_value("keysize", self.options))
        if ArrayUtil.array_key_exists("aeskey", self.options):
            srcAesKey = ArrayUtil.array_value("aeskey", self.options)
            srcAesKey = IntUtil.intFromString(srcAesKey, 16)
            srcAesKey = IntUtil.intToBytes(srcAesKey, 16)
        else:
            srcAesKey = Aes.randomAesKey()
        aesKey = rsa.encrypt(srcAesKey)
        plainText = b''
        plainText += IntUtil.intToBytes(len(aesKey), 4)
        plainText += aesKey

        aes = Aes(srcAesKey)
        data = aes.encrypt(data)
        plainText += IntUtil.intToBytes(len(data), 4)
        plainText += data
        return plainText
```

解密代码实现如下：

```
    def decrypt(self, data):
        streamPos = 0
        aesKeyLength = IntUtil.intFromBytes(data[streamPos:streamPos + 4])
        streamPos = streamPos + 4

        rsa = Rsa(ArrayUtil.array_value("prikey", self.options), ArrayUtil.array_value("modulue", self.options), ArrayUtil.array_value("keysize", self.options))
        aesKey = data[streamPos:streamPos + aesKeyLength]
        aesKey = rsa.decrypt(aesKey)

        streamPos = streamPos + aesKeyLength
        aesDataLength = IntUtil.intFromBytes(data[streamPos:streamPos + 4])
        streamPos = streamPos + 4
        aesData = data[streamPos:streamPos + aesDataLength]
        aes = Aes(aesKey)
        result = aes.decrypt(aesData)
        return result
```

最终附加完整代码：
```
#!/usr/bin/env python3
# -*- coding:utf-8 -*-
#@author: litl
# dependencies:
# pip install pycrypto‎

# import pydevd
# pydevd.settrace('localhost', port=6565, stdoutToServer=True, stderrToServer=True)

import sys
import base64
import getopt
import random
import Crypto.Cipher.AES

def help():
    print(''''Usage: rahcomm [OPTION]... [DATA/-]...
加密/解密公共库网络通信协议的数据, 若没有DATA将从输入流中读取.
Options:
  -h/--help     print help. 
  -d/--decrypt      decrypt, not encrypt.
  -e/--encrypt      encrypt, not decrypt.
  --/--keysize      rsa key length(most be 1024).
  --/--modulue      rsa 加密的模数(加密/解密)都需要提供.
  --/--pubkey       rsa 加密公钥, 仅加密时需要提供.
  --/--prikey       rsa 解密私钥, 仅解密时需要提供.
  --/--aeskey       aes key, 加密时若指定aes key时需要(一般不需要指定).''')

class Aes:

    def __init__(self, key):
        self.key = key

    def padding(self, data):
        BS = Crypto.Cipher.AES.block_size
        BS = (BS - len(data) % BS)
        data += (BS * chr(BS)).encode()
        return data
    def unpad(self, data):
        return data[:-ord(data[len(data)-1:])]

    def encrypt(self, data):
        cipher = Crypto.Cipher.AES.new(self.key, Crypto.Cipher.AES.MODE_ECB)
        data = self.padding(data)
        data = cipher.encrypt(data)
        return data

    def decrypt(self, data):
        cipher = Crypto.Cipher.AES.new(self.key, Crypto.Cipher.AES.MODE_ECB)
        data = cipher.decrypt(data)
        data = self.unpad(data)
        return data

    @staticmethod
    def randomAesKey():
        aeskey = random.randint(0, 0x7fffffffffffffffffffffffffffffff)
        aeskey = IntUtil.intToBytes(aeskey, 16)
        return aeskey

class Rsa:

    def __init__(self, key, modulue, keySize=1024):
        self.rsaKey = IntUtil.intFromString(key)
        self.modulue = IntUtil.intFromString(modulue)
        self.keySize = IntUtil.intFromString(keySize, 10)

    def encrypt(self, data):
        blockSize = int(self.keySize / 8)
        realBlockSize = blockSize - 11
        start = 0
        cypherText = b''
        while len(data) > start:
            length = min(realBlockSize, len(data) - start)
            cypherBlock = self.encryptBlock(data, start, length, blockSize)
            cypherLength = len(cypherBlock)
            cypherText += IntUtil.intToBytes(cypherLength, 4)
            cypherText += cypherBlock
            start += length
        return cypherText

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

    def decrypt(self, data):
        start = 0
        plainText = b''
        while start < len(data):
            blockSize = IntUtil.intFromBytes(data[start:4])
            start += 4
            cypherBlock = data[start:start + blockSize]
            plainBlock = self.decryptBlock(cypherBlock)
            plainText += plainBlock
            start += blockSize

        return plainText

    def decryptBlock(self, data):
        message = IntUtil.intFromBytes(data)
        message = pow(message, self.rsaKey, self.modulue)
        message = IntUtil.intToBytes(message, 0)
        length = IntUtil.intFromBytes(message[1:5])
        start = len(message) - length
        return message[start:start + length]

class IntUtil:

    @staticmethod
    def intToBytes(value, size):
        if size < 1:
            size = int((value.bit_length() + 7) / 8)
        return value.to_bytes(size, byteorder='big')

    @staticmethod
    def intFromBytes(value):
        return int.from_bytes(value, byteorder='big')

    @staticmethod
    def intFromString(value, base=16):
        return int(value, base)

class ArrayUtil:

    @staticmethod
    def array_key_exists(key, options):
        if 1 == len(key):
            key = '-' + key
        else:
            key = '--' + key
        for name, value in options:
            if key == name:
                return True
        return False

    @staticmethod
    def array_value(key, options):
        if 1 == len(key):
            key = '-' + key
        else:
            key = '--' + key
        for name, value in options:
            if key == name:
                return value
        return ''

    @staticmethod
    def in_array(value, options):
        for name, value1 in options:
            if value == value1:
                return True
        return False

    @staticmethod
    def copy(src, srcPos, dst, dstPos, length):
        i = 0
        while i < length:
            dst[dstPos + i] = src[srcPos + i]
            i = i + 1
            pass
class RahComm:
    def __init__(self, options):
        self.options = options

    def decrypt(self, data):
        streamPos = 0
        aesKeyLength = IntUtil.intFromBytes(data[streamPos:streamPos + 4])
        streamPos = streamPos + 4

        rsa = Rsa(ArrayUtil.array_value("prikey", self.options), ArrayUtil.array_value("modulue", self.options), ArrayUtil.array_value("keysize", self.options))
        aesKey = data[streamPos:streamPos + aesKeyLength]
        aesKey = rsa.decrypt(aesKey)

        streamPos = streamPos + aesKeyLength
        aesDataLength = IntUtil.intFromBytes(data[streamPos:streamPos + 4])
        streamPos = streamPos + 4
        aesData = data[streamPos:streamPos + aesDataLength]
        aes = Aes(aesKey)
        result = aes.decrypt(aesData)
        return result
    def encrypt(self, data):
        rsa = Rsa(ArrayUtil.array_value("pubkey", self.options), ArrayUtil.array_value("modulue", self.options), ArrayUtil.array_value("keysize", self.options))
        if ArrayUtil.array_key_exists("aeskey", self.options):
            srcAesKey = ArrayUtil.array_value("aeskey", self.options)
            srcAesKey = IntUtil.intFromString(srcAesKey, 16)
            srcAesKey = IntUtil.intToBytes(srcAesKey, 16)
        else:
            srcAesKey = Aes.randomAesKey()
        aesKey = rsa.encrypt(srcAesKey)
        plainText = b''
        plainText += IntUtil.intToBytes(len(aesKey), 4)
        plainText += aesKey

        aes = Aes(srcAesKey)
        data = aes.encrypt(data)
        plainText += IntUtil.intToBytes(len(data), 4)
        plainText += data
        return plainText

def mainDecrypt(options, args):
    if len(args) < 1:
        args.append(sys.stdin.read())
    i = 0
    while i < len(args):
        data = args[i]
        data = base64.b64decode(data)
        mc = RahComm(options)
        data = mc.decrypt(data)
        data = str(data, "utf-8")
        print(data)
        i = i + 1


def mainEncrypt(options, args):
    if len(args) < 1:
        args.append(sys.stdin.read())
    i = 0
    while i < len(args):
        data = args[i]
        data = data.encode('utf-8')
        mc = RahComm(options)
        data = mc.encrypt(data)
        data = base64.b64encode(data)
        data = str(data, "utf-8")
        print(data)
        i = i + 1

if __name__ == '__main__':
    try:
        options, args = getopt.getopt(sys.argv[1:], 'hde', ['help', 'decrypt', 'encrypt', 'keysize=', 'pubkey=', 'prikey=', 'modulue=', 'aeskey='])
    except Exception as e:
        help()
    else:
        if ArrayUtil.array_key_exists('h', options) or ArrayUtil.array_key_exists('help', options):
            help()
            exit(0)
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
        elif ArrayUtil.array_key_exists('d', options) or ArrayUtil.array_key_exists('decrypt', options):
            if not ArrayUtil.array_key_exists('keysize', options):
                print("解密时必须指定keysize.")
                help()
                exit(1)
            if not ArrayUtil.array_key_exists('prikey', options):
                print("解密时必须指定私钥.")
                help()
                exit(1)
            if not ArrayUtil.array_key_exists('modulue', options):
                print("解密时必须指定模数.")
                help()
                exit(1)
            if not len(ArrayUtil.array_value("modulue", options)) * 4 == int(ArrayUtil.array_value("keysize", options)):
                print("模数和keysize不对称, 可能有一个是错误的.")
                help()
                exit(1)

            mainDecrypt(options, args)
        else:
            print("eithor encrypt or decrypt")
            help()
```
