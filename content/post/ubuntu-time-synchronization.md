+++
title = "Ubuntu與網路時間同步"
date = "2020-09-19"
author = "Zi-Shane"
tags = ["Linux"]
description = "最近使用kubectl遇到時間不同步的問題，導致憑證時間對不上的錯誤，這邊紀錄一下如何設定將ubuntu與NTP Server同步"
+++
## systemd-timesyncd的狀態出現Timed out無法自動和server同步時間
```
$ systemctl status systemd-timesyncd.service

● systemd-timesyncd.service - Network Time Synchronization
   Loaded: loaded (/lib/systemd/system/systemd-timesyncd.service; enabled; vendor preset: enabled)
   Active: active (running) since Thu 2020-09-03 22:46:43 CST; 3h 14min ago
     Docs: man:systemd-timesyncd.service(8)
 Main PID: 761 (systemd-timesyn)
   Status: "Idle."
    Tasks: 2 (limit: 4915)
   CGroup: /system.slice/systemd-timesyncd.service
#  This file is part of systemd.
           └─761 /lib/systemd/systemd-timesyncd
Sep 04 01:13:07 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.94.4:123 (ntp.ubuntu.com).Sep 04 01:13:17 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.91.157:123 (ntp.ubuntu.com).Sep 04 01:13:27 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from [2001:67c:1560:8003::c8]:123 (ntp.ubuntu.com).Sep 04 01:13:37 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from [2001:67c:1560:8003::c7]:123 (ntp.ubuntu.com).Sep 04 01:47:56 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.89.199:123 (ntp.ubuntu.com).Sep 04 01:48:06 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.91.157:123 (ntp.ubuntu.com).Sep 04 01:48:16 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.94.4:123 (ntp.ubuntu.com).Sep 04 01:48:27 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from 91.189.89.198:123 (ntp.ubuntu.com).Sep 04 01:48:37 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from [2001:67c:1560:8003::c7]:123 (ntp.ubuntu.com).Sep 04 01:48:47 dmlab-server systemd-timesyncd[761]: Timed out waiting for reply from [2001:67c:1560:8003::c8]:123 (ntp.ubuntu.com).
```

-----
## 解決方法
### 1. 用`timedatectl`指令檢查目前的時間和timezone
```
                      Local time: Thu 2020-09-03 17:35:02 CST
                  Universal time: Thu 2020-09-03 09:35:02 UTC
                        RTC time: Thu 2020-09-03 09:35:02    
                       Time zone: Asia/Taipei (CST, +0800)
       System clock synchronized: no
systemd-timesyncd.service active: yes
                 RTC in local TZ: no
```
需要更改timezone可以用以下指令查詢和修改，台灣可以直接設`Asia/Taipei`
```
# list all timezone
timedatectl list-timezones

# change timezone
sudo timedatectl set-timezone Asia/Taipei
```

### 2. 指定新的 NTP Server
修改`/etc/systemd/timesyncd.conf`的NTP設定
```
sudo vim /etc/systemd/timesyncd.conf
```
以下台灣ntp server可以選一個
- time.stdtime.gov.tw
- tock.stdtime.gov.tw
- watch.stdtime.gov.tw
- clock.stdtime.gov.tw
- tick.stdtime.gov.tw
```
# /etc/systemd/timesyncd.conf

[Time]
NTP=watch.stdtime.gov.tw      # 指定一個新的server
#NTP=
#FallbackNTP=ntp.ubuntu.com
#RootDistanceMaxSec=5
#PollIntervalMinSec=32
#PollIntervalMaxSec=2048
```

### 3. 重啟 timesync 服務
```
sudo systemctl restart systemd-timesyncd
```
最後可以用`timedatectl`或`date`指令驗證
![](https://i.imgur.com/tKAgRXH.png)
