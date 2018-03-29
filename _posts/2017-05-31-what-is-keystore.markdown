---
layout: post
category: android
title:  "剖析keystore文件"
tags: [keystore,cygwin,openssl]
---

Keytool是一个Java数据证书的管理工具, Keytool将密钥（key）和证书（certificates）存在一个称为keystore的文件中。这句话可能比较官方，那么keystore里到底存放什么呢，它怎么用来rsa加密呢，这里基于剖析的方式从keystore文件中抽取公钥和私钥，并用来加密/解密数据。

<!-- more -->

## keystore文件是什么

keystore文件是由keytool工具生成的，keytool是一个java数字证书管理工具。
keystool会将密钥和公钥/证书存放在keystore文件中， 当然管理keystore文件还需要密码，但密码只是用来打开keystore文件的，和密钥和公钥/证书没有关系。

<!-- more -->

在keystore里，包含以下内容(这篇文章关注的)：

1. 非对称的密钥

2. 非对称的公钥(公钥在证书里)

3. 证书信息


## keystore文件提取私钥


1.转换keystore到pkcs12证书格式

```
keytool.exe -importkeystore -srckeystore demokey.keystore -deststoretype PKCS12 -destkeystore new.keystore 2>&1 | iconv -f gbk
```

2.从pkcs12证书中抽取private key

```
openssl.exe pkcs12 -in new.keystore -out new_private_key.pem -nodes -nocerts
```

3.删除new_private_key.pem文件中不必要的信息,只留私钥

new_private_key.pem是存储在keystore文件中的私钥

4.由private key生成 public key

```
openssl.exe rsa -in new_private_key.pem  -pubout -out new_public_key.pem
```

## keystore文件提取证书和公钥

1.使用keytool提取证书cert

```
keytool.exe -exportcert -keystore demokey.keystore -alias demokey.keystore -file cert  2>&1 | iconv -f gbk
```

2.抽取证书中的public key

```
openssl x509 -inform DER -in cert -pubkey -out pubkey.pub -noout > rsa_public_key.pem
```

3.验证rsa_public_key.pem 与 new_public_key.pem 相同

```
md5sum new_public_key.pem rsa_public_key.pem
```

4.获取证书的相应指纹

```
sha256sum cert
```

## 私钥/公钥和加密/解密


1.私钥加密，公钥解密

```
openssl rsautl -inkey rsa_private_key.pem -sign -in data -out encryptdata
openssl rsautl -inkey rsa_public_key.pem -in encryptdata -verify -pubin

```

2.公钥加密，私钥解密

```
openssl rsautl -encrypt -pubin -inkey new_public_key.pem -in data -out encryptdata
openssl rsautl -decrypt -inkey new_private_key.pem -in encryptdata
```
