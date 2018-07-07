---
title: "Android で再生中の音楽の metadata を MediaController を利用して取得する"
date: 2018-07-07T17:50:22+09:00
draft: false
tags: ["Android"]
---
`BroadcastIntent`を利用して他のアプリが再生している楽曲情報を取得する方法は検索するとたくさん出てきますが、対象とするアプリごとの実装が必要になるためいかんせんイケていない。  
というわけで昨今 Android で音楽再生する際にはほぼ間違いなく用いられる`MediaSession`を利用して、サクッとメタデータを抜き出す方法を紹介します。

# MediaSession とは
おそらく Android Auto の登場と共に、音楽再生アプリに依存しない再生コントロールAPIが必要となり、その実現の一環として導入されたクラスです。  
メディアを扱う際に必要となる様々な情報が集約されていて、実際の再生コントロールには`MediaController`が用いられます。

# MediaController を取得する
そして、`MediaController`を取得してしまえば`MediaController.getMetadata()`によってメタデータが手に入ってしまいます。
`MediaController`の取得には`MediaSessionManager`を用いますが、ここで`MediaSessionManager.addOnActiveSessionsChangedListener()`を用いると、登録した`MediaSessionManager.OnActiveSessionsChangedListener`の`onActiveSessionsChanged()`に返ってくる`MutableList<MediaController?>`の並び順の法則性が不明な上に、`onActiveSessionsChanged()`が呼ばれる条件がよく分からず実用に耐えなかったので、今回は`NotificationListenerService`と併用しました。

それでは [NowPlaying4Droid](https://github.com/geckour/NowPlaying4Droid) から抜粋したサンプルコードを見てみます。

`NotificationService.kt` :
```kotlin
override fun onNotificationPosted(sbn: StatusBarNotification?) {
    super.onNotificationPosted(sbn)

    if (sbn != null && sbn.packageName != packageName)
        fetchMetadata(sbn.packageName)
}

private fun fetchMetadata(packageName: String): MediaMetadata? =
        getSystemService(MediaSessionManager::class.java).let {
            val componentName =
                    ComponentName(this@NotificationService,
                            NotificationService::class.java)

            return@let it.getActiveSessions(componentName)
                    .firstOrNull { it.packageName == packageName }
                    ?.metadata
        }
```
これで、自身のアプリから以外の新しい通知が生成される度にメタデータの取得を試みるようになります。  
`NotificationListenerService`の導入方法については[こちら](/posts/2018/0330-introduce_notification_listener_service/)を参考にしてください。

# まとめ
やってみれば案外簡単に`MediaMetadata`を取得できてしまいました。  
とはいえ、この辺りの API はあまり使われないのかドキュメントが整っていない部分も多く、本当はもう少しスマートにできるんじゃないかという気がとてもします。  
それでは。
