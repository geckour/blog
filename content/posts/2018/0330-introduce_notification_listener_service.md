---
title: "NotificationListenerServiceを導入する"
date: 2018-03-30T14:38:33+09:00
draft: false
tags: ["Android"]
---
設定して権限を与えると、表示されている通知の内容を覗き放題になるNotificationListenerService。  
あまり使いたくはなかったのですが、使わざるを得ない状況となったので嵌ったポイントなどを残しておきます。

# NotificationListenerService
`NotificationListenerService`を継承したクラスを作り、`AndroidManifest.xml`に適切な設定を書いて、システム設定で権限を与えると現在表示されている通知をパッケージに関係なく取得可能になります。  
権限付与時に影響範囲を絞る事ができないため、ユーザにとってはできる限り避けたい機能を提供するAPIだと言えるかもしれません。

## 既知のバグ (仕様？)
`NotificationListenerService`を継承したクラスに設定から権限を付与して、色々やっているうちに通知内容が取得できなくなることがあります。  
その際、既に色々なところで言われていますが、権限の振り直しやあやしいところのデバッグなど、何をしても治らなくなることがあり、そうなると継承しているクラスの名前を変えるしかなくなってしまいます。  

アプリのアップデート時にもこのバグが確認されているそうで、注意が必要です。

## ワークフロー
### `NotificationListenerService`を継承したクラスの定義
Google Play Musicアプリの通知を監視するシンプルなサンプルを載せておきます。

`GPMNotificationListenerService.kt` :
```kotlin
import android.app.Notification
import android.graphics.Bitmap
import android.graphics.drawable.BitmapDrawable
import android.service.notification.NotificationListenerService
import android.service.notification.StatusBarNotification
import timber.log.Timber

class GPMNotificationListenerService : NotificationListenerService() {

    companion object {
        private const val PACKAGE_NAME_GPM: String = "com.google.android.music"
    }

    override fun onNotificationPosted(sbn: StatusBarNotification?) {
        super.onNotificationPosted(sbn)
        if (sbn == null) return

        if (sbn.packageName == PACKAGE_NAME_GPM)
            onUpdate(sbn.notification)
    }

    private fun onUpdate(notification: Notification) {
        notification.extras.apply {
            val track: String? = if (containsKey(Notification.EXTRA_TITLE)) getString(Notification.EXTRA_TITLE) else null
            val artist: String? = if (containsKey(Notification.EXTRA_TEXT)) getString(Notification.EXTRA_TEXT) else null
            val album: String? = if (containsKey(Notification.EXTRA_INFO_TEXT)) getString(Notification.EXTRA_SUB_TEXT) else null
            val artwork: Bitmap = (notification.getLargeIcon().loadDrawable(this@GPMNotificationListenerService) as BitmapDrawable).bitmap
            Timber.d("track: $track, artist: $artist, album: $album, artwork: $artwork")
        }
    }
}
```

### 定義したクラスを`AndroidManifest.xml`に登録
必ず指定しなければいけないパラメータがあります。ミニマムな定義を以下に示します。

`AndroidManifest.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    package="com.example.geckour.notificationlistenerserviceexsample">

    <application
        android:allowBackup="true"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/AppTheme">

        ...

        <service android:name=".GPMNotificationListenerService"
            android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
            <intent-filter>
                <action android:name="android.service.notification.NotificationListenerService" />
            </intent-filter>
        </service>

        ...

    </application>
</manifest>
```

### 定義したクラスへ権限を付与
定義したクラスに権限を付与しなくても即アプリがクラッシュするということはありません。  
しかし、権限がないと、定義したクラスが自動的に実行されることはなく、通知状況が変化しても何も起こりません。  
権限の確認と設定画面の表示に関するサンプルコードを以下に示します。

`SampleActivity.kt` :
```kotlin
class SampleActivity: Activity() {

    enum class RequestCode {
        GRANT_NOTIFICATION_LISTENER
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        requestNotificationListenerPermission()
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        when (requestCode) {
            RequestCode.GRANT_NOTIFICATION_LISTENER.ordinal -> {
                requestNotificationListenerPermission()
            }
        }
    }

    private fun requestNotificationListenerPermission(onGranted: () -> Unit = {}) {
        if (NotificationManagerCompat.getEnabledListenerPackages(this).contains(packageName).not()) {
            AlertDialog.Builder(this)
                    .setTitle(R.string.dialog_title_alert_grant_notification_listener)
                    .setMessage(R.string.dialog_message_alert_grant_notification_listener)
                    .setCancelable(false)
                    .setPositiveButton(R.string.dialog_button_ok) { dialog, _ ->
                        startActivityForResult(
                                Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS"),
                                RequestCode.GRANT_NOTIFICATION_LISTENER.ordinal)
                        dialog.dismiss()
                    }.show()
        } else onGranted()
    }
}
```

### おしまい
これで好きなだけ通知と戯れることができます。  
用法・用量を正しく守って素敵なAndroidライフを。
