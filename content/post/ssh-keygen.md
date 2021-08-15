+++
title = "產生ssh keypair免輸密碼登入Linux Server（ed25519）"
date = "2021-01-22"
author = "Zi-Shane"
tags = ["ssh-keygen", "Linux"]
description = "產生新的ed25519 keypair來遠端登入server"
+++

## 產生key pair

```
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519
```
- `-t`可以指定產生的key pair用的演算法[dsa | ecdsa | ed25519 | rsa]
想知道不同演算法的差別可以參考這篇[[1]](https://medium.com/risan/upgrade-your-ssh-key-to-ed25519-c6e8d60d3c54)
- `-f`指定存放private key位置

要求設定passphrase可以直接`enter`跳過

![](https://i.imgur.com/C4SVsZw.png)

會產生public key(id_ed25519.pub)和private key(id_ed25519)

![](https://i.imgur.com/m62Cib1.png)

---

## 複製public key到遠端server

```
ssh-copy-id -i ~/.ssh/id_ed25519 <user-name>@<your-ip>
```

-----

## Reference:  
[Upgrade Your SSH Key to Ed25519](https://medium.com/risan/upgrade-your-ssh-key-to-ed25519-c6e8d60d3c54)
