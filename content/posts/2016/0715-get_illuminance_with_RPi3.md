---
title: "Raspberry Pi 3で照度の取得"
date: 2018-07-15T03:42:00+09:00
draft: false
---
今回使ったセンサは<a href="http://akizukidenshi.com/catalog/g/gM-08219/" target="_blank">こちら</a> (TSL2561)  
今回の回路図がこちら  
![circuit](/images/temp_humi_press_illumi.png)

今回のサンプルコードは、<a href="https://github.com/janheise/TSL2561/blob/master/TSL2561.py" target="_blank">こちら</a>のコードをsmbus向けに書きなおしたものです。

`get_tsl2561.py`>  
```py
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import smbus as sb
import sys
import os
import time
import json


def overwrite(s):
    sys.stdout.write("\r\033[K" + s)
    sys.stdout.flush()


class TSL2561:
    VISIBLE = 2  # channel 0 - channel 1
    INFRARED = 1  # channel 1
    FULLSPECTRUM = 0  # channel 0

    ADDR_LOW = 0x29
    ADDR_NORMAL = 0x39
    ADDR_HIGH = 0x49

    PACKAGE_CS = 0
    PACKAGE_T_FN_CL = 1

    COMMAND_BIT = 0x80
    WORD_BIT = 0x20

    CONTROL_POWERON = 0x03
    CONTROL_POWEROFF = 0x00
    LUX_CHSCALE_TINT0 = 322.0 / 11
    LUX_CHSCALE_TINT1 = 322.0 / 81

    REGISTER_CONTROL = 0x00
    REGISTER_TIMING = 0x01
    REGISTER_DATA = 0x9B
    REGISTER_ID = 0x0A
    REGISTER_CHAN0_LOW = 0x0C
    REGISTER_CHAN0_HIGH = 0x0D
    REGISTER_CHAN1_LOW = 0x0E
    REGISTER_CHAN1_HIGH = 0x0F

    INTEGRATIONTIME_13MS = 0x00  # 13.7ms
    INTEGRATIONTIME_101MS = 0x01  # 101ms
    INTEGRATIONTIME_402MS = 0x02  # 402ms

    GAIN_1X = 0x00
    GAIN_16X = 0x10

    address = ADDR_NORMAL
    i2cbus = 1
    package = PACKAGE_CS
    gain = GAIN_1X
    timing = INTEGRATIONTIME_402MS

    def __init__(self, address=ADDR_NORMAL, bus=i2cbus):
        self.address = address
        self.i2cbus = bus
        self.bus = sb.SMBus(self.i2cbus)
        self.fromcgi = "REQUEST_METHOD" in os.environ

    def find_sensor(self):
        self.bus.write_i2c_block_data(self.address, self.REGISTER_ID, [])
        read_results = self.bus.read_byte(self.address)

        if not self.fromcgi:
            print("state: %02x" % read_results)
        if read_results == 0x0A:
            return True
        return False

    def set_gain_and_timing(self):
        self.enable()
        self.bus.write_i2c_block_data(self.address, self.COMMAND_BIT | self.REGISTER_TIMING, [self.gain | self.timing])
        self.disable()

    def enable(self):
        self.bus.write_i2c_block_data(self.address, self.COMMAND_BIT | self.REGISTER_CONTROL, [self.CONTROL_POWERON])

    def disable(self):
        self.bus.write_i2c_block_data(self.address, self.COMMAND_BIT | self.REGISTER_CONTROL, [self.CONTROL_POWEROFF])

    def wait(self):
        if self.timing == self.INTEGRATIONTIME_13MS:
            time.sleep(0.14)
        if self.timing == self.INTEGRATIONTIME_101MS:
            time.sleep(0.102)
        if self.timing == self.INTEGRATIONTIME_402MS:
            time.sleep(0.403)

    def get_full_luminosity(self):
        self.set_gain_and_timing()
        self.enable()
        self.wait()

        read_results = [int, int]
        self.bus.write_i2c_block_data(self.address, self.COMMAND_BIT | self.WORD_BIT | self.REGISTER_CHAN0_LOW, [])
        read_results[0] = self.bus.read_i2c_block_data(self.address, 2)
        self.bus.write_i2c_block_data(self.address, self.COMMAND_BIT | self.WORD_BIT | self.REGISTER_CHAN1_LOW, [])
        read_results[1] = self.bus.read_i2c_block_data(self.address, 2)

        self.disable()

        full = [int, int]
        full[0] = read_results[0][1]
        full[0] <<= 8
        full[0] += read_results[0][0]
        full[1] = read_results[1][1]
        full[1] <<= 8
        full[1] += read_results[1][0]

        return float(full[0]), float(full[1])

    def get_luminosity(self, channel):
        x, y = self.get_full_luminosity()

        if channel == self.FULLSPECTRUM:
            return x
        if channel == self.INFRARED:
            return y
        if channel == self.VISIBLE:
            return x - y
        return 0

    def calculate_lux(self, ch0, ch1):
        ch_scale = 1
        if self.timing == self.INTEGRATIONTIME_13MS:
            ch_scale = self.LUX_CHSCALE_TINT0
        if self.timing == self.INTEGRATIONTIME_101MS:
            ch_scale = self.LUX_CHSCALE_TINT1

        if self.gain == self.GAIN_1X:
            ch_scale <<= 4

        ch0 *= ch_scale
        ch1 *= ch_scale

        ratio = 0
        if ch0 != 0:
            ratio = ch1 / ch0

        if ratio > 0:
            if self.package == self.PACKAGE_CS:
                if ratio <= 0.52:
                    return 0.0315 * ch0 - 0.0593 * ch0 * (ratio ** 1.4)
                elif ratio <= 0.65:
                    return 0.0229 * ch0 - 0.0291 * ch1
                elif ratio <= 0.80:
                    return 0.0157 * ch0 - 0.0180 * ch1
                elif ratio <= 1.30:
                    return 0.00338 * ch0 - 0.00260 * ch1
                else:
                    return 0
            else:
                if ratio <= 0.50:
                    return 0.0304 * ch0 - 0.062 * ch0 * (ratio ** 1.4)
                elif ratio <= 0.61:
                    return 0.0224 * ch0 - 0.031 * ch1
                elif ratio <= 0.80:
                    return 0.0128 * ch0 - 0.0153 * ch1
                elif ratio <= 1.30:
                    return 0.00146 * ch0 - 0.00112 * ch1
                else:
                    return 0

        return 0

sensor = TSL2561()
if not sensor.find_sensor():
    print "エラー: TSL2561が見つかりません\n"
    exit(1)


if sensor.fromcgi:
    broadband, ir = sensor.get_full_luminosity()
    lux = sensor.calculate_lux(broadband, ir)
    print "Content-Type: application/javascript; charset=utf-8\n\n"
    print json.dumps({"mix": broadband, "ir": ir, "lux": lux})
else:
    try:
        while True:
            broadband, ir = sensor.get_full_luminosity()
            lux = sensor.calculate_lux(broadband, ir)
            overwrite("mix: 0x%04x: %d, ir: 0x%04x: %d, lux: %f" % (broadband, broadband, ir, ir, lux))
            time.sleep(1)
    except KeyboardInterrupt:
        sensor.disable()
        print "\n"
```
