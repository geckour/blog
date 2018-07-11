---
title: "Raspberry Pi 3で温度/湿度の取得"
date: 2016-07-12T00:23:47+09:00
draft: false
tags: ["RPi", "Web"]
---
今回使うセンサは<a href="http://akizukidenshi.com/catalog/g/gM-08775/" target="_blank">こちら</a>(AE-HDC1000)。  

まずは回路図  
![temp_humi_press](/images/temp_humi_press.png)  
今回は左側のブロックを使用します(右側の気圧センサは次の記事で)。  

Raspberry Piに、今回使うインターフェイスであるI2Cの管理用パッケージを入れて、使用するアドレスを確認します
```bash
$ sudo apt-get install i2c-tools
$ sudo i2cdetect -y 1
```
```
     0  1  2  3  4  5  6  7  8  9  a  b  c  d  e  f
00:          -- -- -- -- -- -- -- -- -- -- -- -- --
10: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
20: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
30: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
40: 40 -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
50: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
60: -- -- -- -- -- -- -- -- -- -- -- -- -- -- -- --
70: -- -- -- -- -- -- -- --  
```
こんな感じでアドレスが取れたら次へ  

PythonからI2Cバスをコントロールするライブラリのインストール  
```bash
$ sudo apt-get install python-smbus
$ sudo pip install wiringpi2
```
ここで`error: command 'arm-linux-gnueabihf-gcc' failed with exit status 1`のようなエラーが出た時は、`sudo apt-get install python-dev`をしてからもう一度`sudo pip install wiringpi2`する。  

今回のサンプルコードはこちら  

`get_hdc1000.py` :
```python
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
    sys.stdout.write(&quot;\r&#092;&#048;33[K&quot; + s)
    sys.stdout.flush()


class HDC1000:
    def __init__(self):
        self.hdc1000 = sb.SMBus(HDC1000_CHANNEL)
        # self.fd = os.open(&quot;/dev/i2c-%d&quot; % HDC1000_CHANNEL, os.O_RDWR, 0)
        self.fd = i2c.setup(HDC1000_ADDRESS)
        self.fromcgi = &quot;REQUEST_METHOD&quot; in os.environ

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
                overwrite(&quot;測定中...&quot;)
            time.sleep(0.01)

        data = [((struct.unpack('4B', os.read(self.fd, 4)))[0] &lt;&lt; 8 |
                 (struct.unpack('4B', os.read(self.fd, 4)))[1]),
                ((struct.unpack('4B', os.read(self.fd, 4)))[2] &lt;&lt; 8 |
                 (struct.unpack('4B', os.read(self.fd, 4)))[3])]

        if data[0] &lt; 0 or data[1] &lt; 0:
            if not self.fromcgi:
                overwrite(&quot;エラー: 値が取得できません&quot;)
            return None
        else:
            return parse(data)

    def get_value(self):
        data = HDC1000.read_value(self)
        if data is not None:
            return {&quot;temp&quot;: data[0], &quot;humi&quot;: data[1]}
        else:
            return None

sensor = HDC1000()
sensor.config()

if sensor.fromcgi:
    sys.stdout.write(&quot;Content-Type: application/javascript; charset=utf-8\n\n&quot;)
    value = sensor.get_value()
    if value is not None:
        sys.stdout.write(json.dumps(value))
    else:
        sys.stdout.write(json.dumps({&quot;error&quot;: &quot;Can't read value&quot;}))
else:
    try:
        while True:
            value = sensor.read_value()
            if value is not None:
                overwrite(&quot;温度: &quot; + str(value[0]) + &quot;℃, 湿度: &quot; + str(value[1]) + &quot;%&quot;)
            else:
                overwrite(&quot;エラー: 有効な値が取得できません&quot;)
                break
            time.sleep(INTERVAL)
    except KeyboardInterrupt:
        pass

    os.close(sensor.fd)
    GPIO.cleanup()

```
ちなみにこのコード、CGIとして叩くとJSONが返ります  
