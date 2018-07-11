---
title: "Raspberry Pi 3で気圧(と温度)の取得"
date: 2016-07-12T21:08:00+09:00
draft: false
tags: ["RPi", "Web"]
images: ["/images/temp_humi_press.png"]
---
今回使うセンサは<a href="http://akizukidenshi.com/catalog/g/gK-08338/" target="_blank">こちら</a>(AE-LPS25H)。  
今回の回路図は<a href="https://blog.geckour.com/2016/0712-get_temperature_and_humidity_withRPi3" target="_blank">前回</a>と同じです  
![circuit](/images/temp_humi_press.png)  
今回は右側のブロックを使用します。

I2Cデバイスの設定方法は<a href="https://blog.geckour.com/2016/0712-get_temperature_and_humidity_withRPi3" target="_blank">前回</a>を参照してください。  
LPS25Hのアドレスは`0x5c`です。

今回のサンプルコードはこちら

`get_lps25h.py`  
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
    sys.stdout.write("\r\033[K" + s)
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
    sys.stdout.write("Content-Type: application/javascript; charset=utf-8\n\n")
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
        print "\n"
```
このコードはCGIとして実行するとJSONを返します。
