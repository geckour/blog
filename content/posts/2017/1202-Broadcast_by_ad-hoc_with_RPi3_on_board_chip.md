---
title: "Raspberry Pi 3 model BのオンボードWi-Fiチップを使ってad-hocでbroadcast通信する"
date: 2017-12-02T12:37:00+09:00
draft: false
tags: ["RPi"]
---
ad-hocモードに設定する際、妙にハマったのでメモ。

まずは時刻が正確に同期されていることを確認する。
Proxy環境下などでNTPが使えない場合は以下の手順で同期する。

`$ nano ~/.bashrc`

`.bashrc`>  
```sh
http_proxy=http://your.proxy.example.com:port
https_proxy=http://your.proxy.example.com:port
ftp_proxy=http://your.proxy.example.com:port

alias setdate='sudo date --set @"$(wget -q https://ntp-a1.nict.go.jp/cgi-bin/jst -O - | sed -n 4p | cut -d. -f1)"'
```

```sh
$ source ~/.bashrc
$ setdate  # 時刻を同期
```

```sh
$ sudo -E rpi-update  # ファームウェアなどを更新
$ sudo reboot
```

`$ sudo nano /etc/network/interfaces.d/static_ad-hoc_wlan0`

`static_ad-hoc_wlan0`>  
```sh
auto wlan0
allow-hotplug wlan0
iface wlan0 inet static

address 192.168.0.x
netmask 255.255.255.0

wireless-channel 4  # 空いてるチャンネルを選ぶ
wireless-mode ad-hoc
wireless-essid hoge  # 何でもよい
wireless-key f0f0f0f0f0  # 16進10桁 or 26桁
```

```sh
$ sudo rfkill unblock all;sudo ip addr flush dev wlan0;sudo systemctl restart networking
$ sudo reboot  # 設定がうまく反映されてない場合は再起動する
```

ここまででad-hocの設定は終了。  
ついでにPythonでのbroadcastのサンプルコードも載せときます。

送信用  
`send.py`>  
```py
from socket import *

message = b"Hoge Fuga"

client = socket(AF_INET, SOCK_DGRAM)
client.setsockopt(SOL_SOCKET, SO_BROADCAST, 1)

client.sendto(message, ("192.168.0.255", 10101))  # ポートは任意に決定
print "message \"%s\" has send." % message
```

受信用  
`receive.py`>  
```py
from socket import *


server = socket(AF_INET, SOCK_DGRAM)
server.bind(("", 10101))  # ポートは任意に決定

try:
    while True:
        message = server.recvfrom(10000)
        if message:
            print(message)
except KeyboardInterrupt:
    print ""
```
