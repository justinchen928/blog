---
title: "ssh key pair登入Server"
date: 2019-12-29T08:05:10+08:00
author: justinchen
type: post
slug: login-remote-server-with-ssh-key-pair
share_img: https://miro.medium.com/v2/resize:fit:1400/format:webp/1*608C4EoqJ5WDMPOqZOX8_Q.png
categories:
  - ssh
tags: ["ssh"]
---

![Cover](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*608C4EoqJ5WDMPOqZOX8_Q.png)

最近在嘗試用 VS code的Remote-ssh，但每次都需要輸入密碼很麻煩，想說直接用key登入，就不用輸入密碼。

<!--more-->
### Client端

首先要產生ssh key pair：

    ssh-keygen

* -t 使用特定演算法 [ dsa | ecdsa | ed25519 | rsa | rsa1 ]

* -e 指定key的長度，單位是 bits

之後會讓你輸入key要存在哪邊，這邊可以輸入想要儲存的地方和名稱，再來會詢問要不要輸入 passphrase ，輸入的話之後用這個key都要輸入。當然我們用key是懶癌末期了不想輸入，所以直接按Enter跳過。

最後會產生一個private key YOUR_KEY_NAME 和一個public key YOUR_KEY_NAME.pub

private key 就是我們登入server用的key，所以public key我們要丟到server的 ~/.ssh/authorized_keys。

    ssh-copy-id -i ~/.ssh/YOUR_KEY_NAME.pub USER_NAME@SERVER_HOST

需要先輸入密碼後會幫你把public key加到 authorized_keys 裡。

### Server端

先確定兩點

* ~/.ssh 目錄權限是 700 ，只能夠被你 讀、寫、以及執行。

* authorized_keys 是 600 ，只能夠被你 讀、寫。

再來查看server ssh的設定：

    sudo vi /etc/ssh/sshd_config

確認以下兩個設定：

    PubkeyAuthentication    yes                    #公開金鑰認證登入方式
    AuthorizedKeysFile      .ssh/authorized_keys   #公開金鑰檔案位置

甚至把密碼登入方式關掉：

    PasswordAuthentication  no

最後重新啟動ssh服務。

理論上這些做完就應該可以直接使用private key登入server

    ssh -i ~/.ssh/YOUR_KEY_NAME USER_NAME@SERVER_HOST

### Debug

但事情通常不會這麼簡單，Google得到的方法都做了，他還是叫我用密碼登入。最後找到了ssh debug的方式，在server端輸入：

    sudo $(which sshd) -p 6666 -d

* -p 使用6666 port

* -d debug模式

然後在client端用ssh在連一次server，但這次加上 -p 6666 和 -v

    ssh -p -v 6666 -i ~/.ssh/YOUR_KEY_NAME USER_NAME@SERVER_HOST

可以在client端和server端看到ssh分別做了什麼事，最後發現在server端有個訊息

    Authentication refused: bad ownership or modes for directory /SSH_PARENT_PATH

說我的根目錄權限有問題，確認後才發現根目錄的權限是 775 ，表示除了我自己擁有讀寫執行的權限外，跟我同個group得使用者也一樣能對我的根目錄操作讀寫執行，應該是檔案權限太鬆，改成 755 就可以了。

### 參考

* [ssh-keygen — Generate a New SSH Key](https://www.ssh.com/ssh/keygen)

* [sshd](https://www.ssh.com/ssh/sshd)

* [ssh-copy-id](https://www.ssh.com/ssh/copy-id)

* [sshd_config](https://www.ssh.com/ssh/sshd_config)

* [ssh-client-does-not-offer-public-key-to-ssh-server](https://unix.stackexchange.com/questions/275161/ssh-client-does-not-offer-public-key-to-ssh-server)
