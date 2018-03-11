---
title: "Raspberry Pi 3のセンシング値をサーバから取得・ロギング"
date: 2016-07-15T03:27:00+09:00
draft: false
tags: ["RPi", "CentOS7", "JavaScript"]
---
前回まででセンサから値を取れるようになったので、今回はそのデータを利用してロギングしたいと思います。

まずはRaspberry Pi上でCGIスクリプトを実行できるようにする準備から。
```sh
$ sudo apt-get install apache2 #Apache(Webサーバ)のインストール 自動起動設定も勝手にやってくれます
$ sudo ln -s /etc/apache2/mods-available/cgi.load /etc/apache2/mods-enabled/cgi.load #CGIの有効化1
$ sudo nano /etc/apache2/sites-available/000-default.conf #Apacheの設定変更 (CGIの有効化2)
```

`000-default.conf`>  
```sh
<VirtualHost *:80>
        # The ServerName directive sets the request scheme, hostname and port that
        # the server uses to identify itself. This is used when creating
        # redirection URLs. In the context of virtual hosts, the ServerName
        # specifies what hostname must appear in the request's Host: header to
        # match this virtual host. For the default virtual host (this file) this
        # value is not decisive as it is used as a last resort host regardless.
        # However, you must set it for any further virtual host explicitly.
        ServerName hoge.fuga.piyo #コメントアウトを解除、適当なサーバ名を登録

        ServerAdmin webmaster@localhost
        DocumentRoot /var/www/html

        # Available loglevels: trace8, ..., trace1, debug, info, notice, warn,
        # error, crit, alert, emerg.
        # It is also possible to configure the loglevel for particular
        # modules, e.g.
        #LogLevel info ssl:warn
        LogLevel debug

        ErrorLog ${APACHE_LOG_DIR}/error.log
        CustomLog ${APACHE_LOG_DIR}/access.log combined

        # For most configuration files from conf-available/, which are
        # enabled or disabled at a global level, it is possible to
        # include a line for only one particular virtual host. For example the
        # following line enables the CGI configuration for this host only
        # after it has been globally disabled with "a2disconf".
        Include conf-available/serve-cgi-bin.conf #コメントアウトを解除
</VirtualHost>

# vim: syntax=apache ts=4 sw=4 sts=4 sr noet
```

```sh
$ sudo nano /etc/apache2/mods-available/mime.conf #CGIの有効化3
```

`mime.conf`>  
```sh
AddHandler cgi-script .cgi .py #219行目あたり コメントアウトを解除、追記
```

```sh
$ sudo gpasswd -a www-data gpio #Webサーバの実行ユーザがGPIO周りを触れる用にする (ユーザ www-data のグループ gpio への追加)
$ sudo apachectl graceful #Apacheを再起動
```
これで準備は完了です。

Raspberry Pi用のサンプルコードがこちら (前回までに出てきたものと同じです)  

`get_hdc1000.py 温湿度取得用`>  
```py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import RPi.GPIO as GPIO
import smbus as sb
import wiringpi as wp
import os
import sys
import time
import struct
import json

HDC1000_RDY_PIN = 27
HDC1000_ADDRESS = 0x40
HDC1000_CHANNEL = 1
INTERVAL = 1

CONFIG_POINTER = 0x02
CONFIG_MSB = 0x10
CONFIG_LSB = 0x00

TEMPERATURE_POINTER = 0x00
HUMIDITY_POINTER = 0x01

GPIO.setmode(GPIO.BOARD)
GPIO.setup(13, GPIO.IN)

i2c = wp.I2C()


def overwrite(s):
    sys.stdout.write("r\033[K" + s)
    sys.stdout.flush()


class HDC1000:
    def __init__(self):
        self.hdc1000 = sb.SMBus(HDC1000_CHANNEL)
        # self.fd = os.open("/dev/i2c-%d" % HDC1000_CHANNEL, os.O_RDWR, 0)
        self.fd = i2c.setup(HDC1000_ADDRESS)
        self.fromcgi = "REQUEST_METHOD" in os.environ

    def config(self):
        setting = [CONFIG_POINTER, CONFIG_MSB, CONFIG_LSB]
        self.hdc1000.write_i2c_block_data(HDC1000_ADDRESS, setting[0], setting[1:])

    def read_value(self):
        def parse(mix_val):
            mix_val[0] = mix_val[0] * 165.0 / float(0xffff) - 40.0
            mix_val[1] = mix_val[1] * 100.0 / float(0xffff)
            return mix_val

        self.hdc1000.write_i2c_block_data(HDC1000_ADDRESS, TEMPERATURE_POINTER, [])

        while GPIO.input(13):
            if not self.fromcgi:
                overwrite("測定中...")
            time.sleep(0.01)

        data = [((struct.unpack('4B', os.read(self.fd, 4)))[0] << 8 |
                 (struct.unpack('4B', os.read(self.fd, 4)))[1]),
                ((struct.unpack('4B', os.read(self.fd, 4)))[2] << 8 |
                 (struct.unpack('4B', os.read(self.fd, 4)))[3])]

        if data[0] < 0 or data[1] < 0:
            if not self.fromcgi:
                overwrite("エラー: 値が取得できません")
            return None
        else:
            return parse(data)

    def get_value(self):
        data = HDC1000.read_value(self)
        if data is not None:
            return {"temp": data[0], "humi": data[1]}
        else:
            return None

sensor = HDC1000()
sensor.config()

if sensor.fromcgi:
    sys.stdout.write("Content-Type: application/javascript; charset=utf-8nn")
    value = sensor.get_value()
    if value is not None:
        sys.stdout.write(json.dumps(value))
    else:
        sys.stdout.write(json.dumps({"error": "Can't read value"}))
else:
    try:
        while True:
            value = sensor.read_value()
            if value is not None:
                overwrite("温度: " + str(value[0]) + "℃, 湿度: " + str(value[1]) + "%")
            else:
                overwrite("エラー: 有効な値が取得できません")
                break
            time.sleep(INTERVAL)
    except KeyboardInterrupt:
        print "n"

    os.close(sensor.fd)
    GPIO.cleanup()
```

`get_lps25h.py 気圧取得用`>  
```py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import smbus as sb
import os
import sys
import time
import json

INTERVAL = 1

LPS25H_ADDRESS = 0x5c
LPS25H_CHANNEL = 1

WHO_AM_I = 0x0f
CTRL_REG1 = 0x20

PRESS_POUT_XL = 0x28
PRESS_OUT_L = 0x29
PRESS_OUT_H = 0x2a

TEMP_OUT_L = 0x2b
TEMP_OUT_H = 0x2c


def overwrite(s):
    sys.stdout.write("r\033[K" + s)
    sys.stdout.flush()


class LPS25H:
    def __init__(self):
        self.lps25h = sb.SMBus(LPS25H_CHANNEL)
        self.fromcgi = "REQUEST_METHOD" in os.environ
        self.esta = False

    def config(self):
        whoami = self.lps25h.read_byte_data(LPS25H_ADDRESS, WHO_AM_I)
        if whoami != 0xbd:
            if not self.fromcgi:
                overwrite("エラー: センサと通信できません")
            self.esta = False
            return
        else:
            self.esta = True

        self.lps25h.write_i2c_block_data(LPS25H_ADDRESS, CTRL_REG1, [0x90])
        if not self.fromcgi:
            overwrite("測定中...")
        time.sleep(0.5)

    def read_value(self):
        def conv(d):
            d[0] = float(d[0]) / float(0xfff)
            if d[1] & 0x8000:
                d[1] = -((d[1] - 1) ^ 0xffff)
            d[1] = float(d[1]) / 480.0 + 42.5

            return d

        if self.esta:
            data = [
                (self.lps25h.read_i2c_block_data(LPS25H_ADDRESS, PRESS_POUT_XL, 1)[0] |
                 self.lps25h.read_i2c_block_data(LPS25H_ADDRESS, PRESS_OUT_L, 1)[0] << 8 |
                 self.lps25h.read_i2c_block_data(LPS25H_ADDRESS, PRESS_OUT_H, 1)[0] << 16),
                (self.lps25h.read_i2c_block_data(LPS25H_ADDRESS, TEMP_OUT_L, 1)[0] |
                 self.lps25h.read_i2c_block_data(LPS25H_ADDRESS, TEMP_OUT_H, 1)[0] << 8)]

            if not (data[0] < 0 or data[1] < 0):
                return conv(data)
            else:
                return None
        else:
            return None

    def get_value(self):
        data = LPS25H.read_value(self)
        if data is not None:
            return {"press": data[0], "temp": data[1]}
        else:
            return None

sensor = LPS25H()
sensor.config()

if sensor.fromcgi:
    sys.stdout.write("Content-Type: application/javascript; charset=utf-8nn")
    value = sensor.get_value()
    if value is not None:
        sys.stdout.write(json.dumps(value))
    else:
        sys.stdout.write(json.dumps({"error": "Can't read value"}))

else:
    try:
        while True:
            value = sensor.read_value()
            if value is not None:
                overwrite("気圧: %fhPa, 温度: %f℃" % (value[0], value[1]))
            else:
                overwrite("エラー: 有効な値が取得できません")
                break
            time.sleep(INTERVAL)

    except KeyboardInterrupt:
        print "n"
```

続いてサーバサイドのサンプルコードがこちら

`log_atmos.py ログ保存用`>  
```py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import requests
import json
from time import time

r0 = requests.get("http://address.to.raspi.documentroot/path/to/cgi/get_hdc1000.py").json()
r1 = requests.get("http://address.to.raspi.documentroot/path/to/cgi/get_lps25h.py").json()

f = open("/path/to/log/atmos.log", "a")
f.write("%sn" % json.dumps({"time": time(), "hdc1000": r0, "lps25h": r1}))
f.close()

[/code]
[code lang="py" title="hold_atmos.py 値保持用"]
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import requests
import json
from time import time

r0 = requests.get("http://address.to.raspi.documentroot/path/to/cgi/get_hdc1000.py").json()
r1 = requests.get("http://address.to.raspi.documentroot/path/to/cgi/get_lps25h.py").json()

f = open("/path/to/var/atmos.txt", "w")
json.dump({"time": time(), "hdc1000": r0, "lps25h": r1}, f)
f.close()
```

これら二つのファイルを `cron` に登録して定期実行します。
```sh
$ sudo nano /etc/crontab
```

`crontab`>  
```sh
# 以下を追記
* * * * * root /path/to/log_hdc1000_lps25h_from_raspi01.py
* * * * * root for i in `seq 0 5 59`;do (sleep ${i} ; /path/to/hold_hdc1000_lps25h_from_raspi01.py) & done;
```
このサンプルでは、上の行は1分に1回、下の行は5秒に一回コマンドが実行されます。

ついでに、値を保持したファイルを読み込んでJSONを返すサンプルがこちら
このコードをJavaScriptから叩きます

`get_atmos_from_hold_file.py`>  
```py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import sys

f = open("/path/to/var/atmos.txt")
sys.stdout.write("Content-Type: application/javascript; charset=utf-8nn")
sys.stdout.write("callback(%s)" % f.readline())
```

JavaScriptはこちら

`atmos.js`>  
```js
$(function() {
    get();
    setInterval(function() {
        get();
    }, 5000)
})

function getReadableValue(value) {
    var l = Math.floor(value).toString();
    var s = Math.floor(value * 100).toString().slice(-2);
    return l + "." + s
}

function get() {
    $.ajax({
        url: "/path/to/get_atmos_from_hold_file.py",
        dataType: "jsonp",
        jsonp: false,
        jsonpCallback: "callback",
        timeout: 1000
    }).done(function(data) {
        $("#temp").text(getReadableValue(data.hdc1000.temp) + "[℃]")
        $("#humi").text(getReadableValue(data.hdc1000.humi) + "[%]")
        $("#press").text(getReadableValue(data.lps25h.press) + "[hPa]")
    });
}
```
