---
title: "Palette APIの導入と通知背景色への適用"
date: 2018-03-17T11:54:43+09:00
draft: false
tag: ["Android"]
---
今回は、昔からあったらしいPalette APIというものを最近知って感動したのでその導入方法と、ついでにAndroid SDK 26から追加された通知背景色の変更APIにPalette APIを適用した例を。

## Palette API
Android Support Library v7に存在する、画像を与えるとその画像の特徴的な色を抜き出してくれるライブラリです。  
抜き出す色は

- Vibrant
- Vibrant Dark
- Vibrant Light
- Muted
- Muted Dark
- Muted Light

の6つの種類 (property) が定義されていて、この中から好みのものを選んで使います。  
ただし、後述しますが必ずこの6種類が全て取得されるわけではなく、指定したpropertyが空の場合もあります。

### 使い方
非常にシンプル・簡単です。  
画像さえ用意できればあとはAPIに投げるだけ。
```kotlin
val bitmap: Bitmap = ...
val color = Palette.from(bitmap).generate().getLightMutedColor(Color.WHITE)
```
ここで、`getLightMutedColor`の引数に`Color.WHITE`を指定していますが、これがそのpropertyが空であった場合に代わりに用いられる色となります。  
お分かりとは思いますが、`Palette.from(bitmap).generate()`によってPaletteのインスタンスを取得して、使いたいpropertyを`Palette#getHogeProperty(fallbackColor)`とすれば`Int`で色が取得できます。

## 通知の背景色変更
Android Oreo (8.0)から追加された`NotificationBuilder#setColorized(colorize: Boolean)`を使うと通知に独自の背景色を設定することができます。  
ここで、いくつか注意すべき点があり、`setColorized`による通知の背景色変更の有効化には

- 通知の表示に用いる`Service`をForegroundServiceとして起動する
- 通知の`Style`に`Notification.MediaStyle`か`Notification.DecoratedMediaCustomViewStyle`を適用した上で、`NotificationBuilder#setMediaSession(token: MediaSession.Token)`により`MediaSession`を設定する

のいずれかが必要となります。  
つまり、`MediaSession`を使わないような通知の場合、`startForegroundService`などによって`Service`を起動します。  
なお、`NotificationChannel`を生成する際の`importance`の値は関係ない模様です。
これを踏まえた上で、
```kotlin
val notificationBuilder: NotificationBuilder = ...

val bitmap: Bitmap = ...
val color = Palette.from(bitmap).generate().getLightMutedColor(Color.WHITE)
notificationBuilder.setColor(color)

if (Build.VERSION.SDK_INT >= 26) notificationBuilder.setColorized(true)
```
とすれば、`NotificationBuilder#setColor`によって指定した色が通知の背景色として適用されます。  
…おや？背景色の指定に`setColor`の色が使われるということは…  

そう。背景色を指定すると、アクセントカラーは指定できなくなります。  
将来的に別々に指定できるAPIが開放されることを淡く期待しつつ、今回はここまで。
