---
title: "Raspberry Pi 3 model B & 部品類の購入からRaspberry Piセットアップまで"
date: 2016-07-09T20:53:28+09:00
draft: false
tags: ["RPi"]
images: ["/images/IMG_20160707_221512.jpg"]
---
ふと思い立ってRaspberry Piを触ってみることにした。

### 材料
今回購入したものはこちら:  
![summary](/images/IMG_20160707_221512.jpg)

- Raspberry Pi 3 model B(element14)
- microSDカード(16GB, TOSHIBA THN-M301R0160A4)
- USB Aオス-microBオスケーブル
- ブレッドボード
- ジャンパコード
  - オス-オス: たくさん
  - オス-メス
    - 赤: 5
    - 黒: 5
    - 白: 10
- 抵抗
  - 100Ω: 5
  - 300Ω: 5
  - 1kΩ:  5
- LED照光式押しボタンスイッチ(モーメンタリ)
- 赤外光LED: 10
- フォトカプラ
- 青色LED: 2
- 押しボタンスイッチ(モーメンタリ)
- トグルスイッチ
- デジタル温湿度センサモジュール(AE-HDC1000)
- デジタル気圧センサモジュール(AE-LPS25H)
- デジタル照度センサモジュール(TSL2561)

計約1万円でした。

### セットアップ手順

#### OSインストール
まずはRaspberry Pi用のOSインストーラである<a href="https://www.raspberrypi.org/downloads/noobs/" target="_blank">NOOBS</a>をダウンロード。
特にこだわりがない限り、NOOBS LITEではなく標準のNOOBSを使用することを推奨。
ダウンロードしたNOOBS_xxx.zipを解凍し、SDカードをFAT32でフォーマットしたらコピーする。

1. SDカード
2. キーボード/マウス
3. HDMI(or 何らかの映像端子)
4. 電源
の順に接続する。電源を接続すると勝手に起動する。

ここまでは良かったのだが、起動すると以下の画面が出た。  
![Error on burning IMG](/images/IMG_20160708_185248.jpg)  
Closeを押すとこの画面に。  
![Error on burning IMG](/images/IMG_20160708_230002.jpg)  
そしてここから1時間以上放置しても画面が変わらない。
気を取り直してフォーマッタをMacのディスクユーティリティに変えて再度フォーマットしてみた(それまではSD Associationの<a href="https://www.sdcard.org/jp/downloads/formatter_4/" target="_blank">SDFormatter</a>を使っていた)が駄目だったので、結局Raspbianのimgを直接焼いた。
Macでの焼き方は<a href="https://www.raspberrypi.org/documentation/installation/installing-images/mac.md" target="_blank">ここ</a>を参照。

#### 各種設定

##### ログイン
ようやく無事起動にこぎつけたので、デフォルトユーザでログインする。
```sh
user: pi
password: raspberry
```

##### 基本設定
何はともあれ`$ sudo raspi-config`。

- `1 Expand Filesystem`を実行し、
- `9 Advanced Options`の`A4 SSH`で有効化されていることを確認したら、クライアントからSSH接続する。
その後、引き続き`raspi-config`を実行し
- `2 Change User Password`でログインパスワードの更新、
- `5 Internationalisation Options`の`I2 Change Timezone`でタイムゾーンの変更(Asia/Tokyo)と`I4 Change Wi-fi Country`でWi-Fi使用地域の設定(JP Japan)
- (必要なら)`9 Advanced Options`の`A2 Hostname`でネットワーク上の名前の変更
- `A5 SPI`と`A6 I2C`で各インターフェイスの有効化

ここで一旦Raspberry Piを再起動する(`sudo shutdown -r now`など)。

続いて、rootのパスワードを変更する。  
`sudo passwd root`

(必要であれば)公開鍵を設置、設定を変更する  
`scp ~/.ssh/id_rsa.pub pi@192.168.x.x:~`

##### Wi-Fi
そしてWi-Fiの設定をする。( **※以下は失敗例** )

`sudo nano /etc/wpa_supplicant/wpa_supplicant.conf`でWi-Fi設定ファイルの編集:
```sh
country=JP
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1

//以下を追加
network={
    ssid="ssid_of_your_device"
    psk="password_of_your_ssid"
}
```

これで`$ sudo ifdown wlan0`と`$ sudo ifup wlan0`でサービスの再起動をすればWi-Fiアクセスポイントに繋がる。
が、ここで問題が。

Wi-Fi側のIPアドレスを指定してSSH接続している場合、Ethernetケーブルを抜くとパイプが壊れ通信できなくなってしまう。  
クライアント -> Raspberry PiはWi-Fi経由、Raspberry Pi -> クライアントはEthernet経由で通信が行われるのが原因らしい。

( **※ここから対応例** )  
面倒くさくなったのでGUIから設定することにした。  
Ethernetケーブルを抜いてから右上のネットワークボタンを押して、接続したいSSIDを選択してpskを入力するだけ。簡単。

##### その他(NTP)
(必要であれば)続いてNTPの設定を行う。

`sudo nano /etc/ntp.conf`
```bash
...

# You do need to talk to an NTP server or two (or three).
#server ntp.your-provider.example

# pool.ntp.org maps to about 1000 low-stratum NTP servers.  Your server will
# pick a different set every time it starts up.  Please consider joining the
# pool: <http://www.pool.ntp.org/join.html>
#server 0.debian.pool.ntp.org iburst #コメントアウト
#server 1.debian.pool.ntp.org iburst #コメントアウト
#server 2.debian.pool.ntp.org iburst #コメントアウト
#server 3.debian.pool.ntp.org iburst #コメントアウト
server x.x.x.x iburst #追加

...
```

保存したら`sudo service ntp restart`。  
少し間を置いて`ntpq -p`を実行し、`remote`欄のアドレスの前に`*`が表示されていたら接続できている。


大方設定終わったので次はLチカでもしようかな
