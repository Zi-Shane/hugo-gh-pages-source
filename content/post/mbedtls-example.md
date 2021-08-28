+++
title = "mbedtls使用筆記"
date = "2021-08-27"
author = "Zi-Shane"
tags = ["mbedtls"]
description = "在Ubuntu上使用mbedtls的API的方式"
+++

目前論文在做MCU的開發，需要使用到mbedtls來做ECDSA跟ECDH，想說能不能在PC上的版本先測試過，再改MbedOS版本的程式碼就好，所以才研究怎麼讓mbedtls在Ubuntu的環境下也能使用。

如果在MbedOS環境想要使用mbedtls的話，不需要額外安裝其他套件，可以直接call mbetls的API

可以參考[這個程式](
https://github.com/Zi-Shane/mbed-os-5-test/tree/mbedtls-test)，
搭配官方API文件 - [ecdsa.h File Reference](https://tls.mbed.org/api/ecdsa_8h.html#aca644ee02921388fdc42eb06377f4b62)

----------

## 下載 mbedtls 原始碼

在Ubuntu下使用的話要自己下載原始碼來編譯安裝到系統中

mbedtls Github repo: https://github.com/ARMmbed/mbedtls

直接下指令來下載

```git
git clone https://github.com/ARMmbed/mbedtls.git
```

目前的 Release 為 v2.27.0

:::warning
v3.0.0 開始mbedtls大幅修改，無法使用此文章的程式碼
:::

![](https://i.imgur.com/UWlsDrC.png)

checkout 到 release 版本的 tag

```git
git checkout tags/v2.27.0
```

----------

## 安裝 mbedtls library

:::info
測試過 v2.26.0 會報錯告知需要安裝 Python
:::

說明中有提到可以用 GUN make, CMake 或 Visual Studio 編譯，但我是使用 Ubuntu 所以這裡我會使用 GNU make 來編譯

另外編譯過程會用到 python 3，如果沒裝的話可以先安裝，然後直接切換目錄到 mbedtls，執行 make 和 make install即可。


```zsh
cd mbedtls
make

# make install會將
# library複製到/usr/local/lib
# include複製到/usr/local/include/mbedtls 
make install
```

基本上到這裡就完成了，在`mbedtls/programs`資料夾下有[範例程式](https://github.com/ARMmbed/mbedtls/tree/development/programs)可以參考
想了解程式碼的話，可以參考[官方API文件](https://tls.mbed.org/api/modules.html)。

---

## 範例程式

在`programs/`底下有一些範例程式可以參考，在下`make`的同時也會把這些範例編好，可以直接使用，舉`pkey/pk_sign`和`pkey/pk_verify`當例子

- `pkey/pk_sign` : 使用private key簽章123.txt這個檔案

```bash
# usage: mbedtls_pk_sign <key_file> <filename>
programs/pkey/pk_sign ../prikey.pem ../123.txt

  . Seeding the random number generator...
  . Reading private key from '../ECDSA_Keypairs/server/Copy_to_Keys/D4142434445464748.pem'
  . Generating the SHA-256 signature
  . Done (created "../123.txt.sig")
```

成功後會產生`123.txt.sig`這個檔案

- `pkey/pk_verify` : 使用public key認證123.txt.sig簽章是否正確

```bash
# usage: mbedtls_pk_verify <key_file> <filename>
programs/pkey/pk_verify ../pubkey.pem ../123.txt

  . Reading public key from '../pubkey.pem'
  . Verifying the SHA-256 signature
  . OK (the signature is valid)
```

需要將x509轉換到publickey可以使用openssl
```bash
openssl x509 -pubkey -noout -in  ECDSA_Keypairs/server/Copy_to_Certificates/D4142434445464748.pem > pubkey.pem
```

----------

## 編譯程式

編譯的時候gcc指令需要指定Library位置，會被放在`/usr/local/lib`
```bash
gcc -o test test.c -lmbedtls -lmbedcrypto -lmbedx509
```

需要AES-GCM、ECDSA或ECDH的程式可以到我Github上下載。[連結](https://github.com/Zi-Shane/mbedtls-example)

---

參考資料：

- 關於public key和private key的格式
https://tls.mbed.org/kb/cryptography/asn1-key-structures-in-der-and-pem
- library
https://www.cs.dartmouth.edu/~campbell/cs50/buildlib.html

---
