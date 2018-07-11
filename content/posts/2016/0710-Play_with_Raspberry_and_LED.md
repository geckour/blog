---
title: "Raspberry Pi 3でLEDと戯れる"
date: 2016-07-10T01:39:15+09:00
draft: false
tags: ["RPi"]
---
まずは今回の回路図から。  
![Lkati.png](/images/Lkati.png)  
ブレッドボード上の左側の塊がLピカ/Lチカ/Lカチ全てに必要な回路で、右側はLカチに必要な回路。  

# Lピカ

LEDを単純に光らせる。以下のサンプルコードではコマンドライン引数で光らせる時間[s]を指定できるようにしている。

`Lpika.py` :
```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import sys
import RPi.GPIO as GPIO
import time

GPIO.setmode(GPIO.BOARD)
GPIO.setup(11, GPIO.OUT)

argvs = sys.argv
argc = len(argvs)

l = 0
if (argc &gt; 1):
    try:
        l = float(argvs[1])
    except ValueError:
        print &quot;数値を入力してください&quot;

if (l &gt; 0):
    do = True
    while do:
        GPIO.output(11, True)
        time.sleep(l)
        do = False

GPIO.cleanup()

```

# Lチカ

LEDを点滅させる。以下のサンプルコードでは0.4秒間隔で10回点滅したら終了する。

`Ltika.py` :
```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import RPi.GPIO as GPIO
import time

GPIO.setmode(GPIO.BOARD)
GPIO.setup(11, GPIO.OUT)

i = 0
while i &lt; 10:
    GPIO.output(11, True)
    time.sleep(0.2)
    GPIO.output(11, False)
    time.sleep(0.2)
    i += 1

GPIO.cleanup()

```

# Lカチ

スイッチのオンオフでLEDを制御する。ここでは敢えて一旦スイッチの情報をRaspberry Piに取り込んでからそれをLEDに伝搬させている。

`Lkati.py` :
```python
#!/usr/bin/env python
# -*- coding: UTF-8 -*-

import RPi.GPIO as GPIO
import sys, time

GPIO.setmode(GPIO.BOARD)
GPIO.setup(12, GPIO.IN)
GPIO.setup(11, GPIO.OUT)

def overwrite(s):
    sys.stdout.write(&quot;\r&#092;&#048;33[K&quot; + s)
    sys.stdout.flush()

try:
    while True:
        if GPIO.input(12):
            GPIO.output(11, True)
            overwrite(&quot;On&quot;)
        else:
            GPIO.output(11, False)
            overwrite(&quot;Off&quot;)
        time.sleep(0.01)

except KeyboardInterrupt:
    pass

GPIO.cleanup()

```
