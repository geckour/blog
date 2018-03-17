---
title: "AndroidアプリにWidgetを導入してみる"
date: 2018-03-17T13:14:42+09:00
draft: false
tags: ["Android"]
---
今までいくつかAndroidアプリを作ってきたものの、ウィジェットを作ったことがないことに気がついたので今回はAndroidのWidgetに入門してみたいと思います。

## Widget
API level 1から存在するらしい~~古の~~技術です。  
あまり更新されてないので、結構今の開発感覚で使おうとするとつらみを感じたりします。  
今回は扱いませんが、リストを持つようなウィジェットを作るには相当の努力が必要です。

### レイアウト作成
まずはレイアウトファイルを作成します。  
`ConstraintLayout`は使えないので、`RelativeLayout`を使うなどして頑張って作りましょう。  
<a href="https://github.com/geckour/NowPlaying4GPM" target="_blank">NowPlaying4GPM</a>で使用しているレイアウトファイルをサンプルとして載せておきます。

`widget_share.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/widget_share_root"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:clickable="true"
    android:focusable="true">

    <LinearLayout
        android:id="@+id/widget_wrap_text"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentStart="true"
        android:layout_centerVertical="true"
        android:layout_marginEnd="8dp"
        android:layout_marginStart="8dp"
        android:layout_toStartOf="@+id/widget_button_setting"
        android:orientation="vertical">

        <TextView
            android:id="@+id/widget_desc_share"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:ellipsize="end"
            android:lines="1"
            android:maxLines="1"
            android:shadowColor="@color/colorShadowTextLight"
            android:shadowRadius="3"
            android:text="@string/widget_desc"
            android:textColor="@color/colorPrimary"
            android:textSize="13sp"/>

        <TextView
            android:id="@+id/widget_summary_share"
            android:layout_width="match_parent"
            android:layout_height="wrap_content"
            android:layout_marginTop="4dp"
            android:ellipsize="end"
            android:lines="1"
            android:maxLines="1"
            android:shadowColor="@color/colorShadowTextDark"
            android:shadowRadius="3"
            android:text="@string/dialog_message_alert_no_metadata"
            android:textColor="@android:color/white"
            android:textSize="13sp"
            tools:text="#NowPlaying Title - Artist (Album)"/>
    </LinearLayout>

    <ImageView
        android:id="@+id/widget_button_setting"
        android:layout_width="56dp"
        android:layout_height="56dp"
        android:layout_alignParentEnd="true"
        android:layout_centerVertical="true"
        android:clickable="true"
        android:focusable="true"
        android:padding="12dp"
        android:src="@drawable/ic_settings_black_24px"
        android:tint="@color/colorAccent"/>
</RelativeLayout>
```

お気づきかもしれませんが、Data Bindingを導入していません。  
後述しますが、ウィジェットにビューを反映する際に用いなければならない`RemoteView`の仕様上、Data Bindingを使う利点がほぼ皆無となるからです。つらい。

### レシーバの作成・登録
#### レシーバ作成
ウィジェットの生成・更新を実現するために、`AppWidgetProvider`を継承してレシーバを作ります。  
適宜メソッドをオーバーライドすることで、ウィジェット生成時の処理や`PendingIntent`受信 (ウィジェット経由のアクション発火) 時の処理を定義することができます。  
ここでも、<a href="https://github.com/geckour/NowPlaying4GPM" target="_blank">NowPlaying4GPM</a>で使用しているレシーバを簡略化して載せておきます。

`ShareWidgetProvider.kt` :
```kotlin
import android.app.PendingIntent
import android.appwidget.AppWidgetManager
import android.appwidget.AppWidgetProvider
import android.content.ComponentName
import android.content.Context
import android.content.Intent
import android.widget.RemoteViews
import com.geckour.nowplaying4gpm.R
import com.geckour.nowplaying4gpm.activity.SettingsActivity
import com.geckour.nowplaying4gpm.activity.SharingActivity

class ShareWidgetProvider : AppWidgetProvider() {

    enum class Action {
        SHARE,
        OPEN_SETTING
    }

    companion object {
        fun getPendingIntent(context: Context, action: Action): PendingIntent =
                PendingIntent.getBroadcast(
                        context,
                        0,
                        Intent(context, ShareWidgetProvider::class.java).apply { setAction(action.name) },
                        PendingIntent.FLAG_UPDATE_CURRENT)
    }

    override fun onEnabled(context: Context?) {
        super.onEnabled(context)

        if (context == null) return

        val summary = ...
        updateWidget(context, summary)
    }

    override fun onReceive(context: Context?, intent: Intent?) {
        super.onReceive(context, intent)

        if (context == null || intent == null) return

        when (intent.action) {
            Action.SHARE.name -> {
                val summary: String = ...
                val artworkUri: Uri = ...
                context.startActivity(SharingActivity.getIntent(context, summary, artworkUri))
            }

            Action.OPEN_SETTING.name -> context.startActivity(SettingsActivity.getIntent(context))
        }
    }

    private fun updateWidget(context: Context, summary: String?) =
            AppWidgetManager.getInstance(context).apply {
                val ids =
                        getAppWidgetIds(ComponentName(context, ShareWidgetProvider::class.java))
                                .firstOrNull() ?: return@apply

                updateAppWidget(
                        ids,
                        RemoteViews(context.packageName, R.layout.widget_share).apply {
                            setTextViewText(R.id.widget_summary_share, summary
                                    ?: context.getString(R.string.dialog_message_alert_no_metadata))

                            setOnClickPendingIntent(
                                    R.id.widget_share_root,
                                    ShareWidgetProvider.getPendingIntent(context,
                                            ShareWidgetProvider.Action.SHARE))
                            setOnClickPendingIntent(
                                    R.id.widget_button_setting,
                                    ShareWidgetProvider.getPendingIntent(context,
                                            ShareWidgetProvider.Action.OPEN_SETTING))
                        }
                )
            }
}
```
`updateWidget`の実装を見るとわかりますが、`AppWidgetManager#updateAppWidget`を呼んでウィジェットを更新する際には、RemoteViewのインスタンスを指定する必要があります。  
情報の更新は`setTextViewText`などの様な限られたインタフェースしか提供されていないため、先述したようにData Bindingのうまみがほぼ皆無となります。  
`AppWidgetProvider`に`RemoteView`のインスタンスをメンバでもたせて、`setHoge`を呼んで行うのがよいのかもしれません。  
今回は`AppWidgetProvider`自体のインスタンスがどう保持されているかわからなかったので、この様に毎回生成する実装としました。  

また、`onReceive`で受け取る`Intent`からはおそらく`Action`のみしか取得できず、`Bundle` (data) は`null`となっているので注意してください。

#### レシーバ登録
作成したレシーバをマニフェストファイルに登録します。  
まずは`res/xml`以下に`AppWidgetProviderInfo`オブジェクトを生成するためのXMLファイルを作成します。

`widget_provider` :
```xml
<?xml version="1.0" encoding="utf-8"?>
<appwidget-provider xmlns:android="http://schemas.android.com/apk/res/android"
    android:minWidth="180dp"
    android:minHeight="40dp"
    android:minResizeWidth="40dp"
    android:minResizeHeight="40dp"
    android:initialLayout="@layout/widget_share"
    android:resizeMode="horizontal"
    android:updatePeriodMillis="0" />
```
きちんと確かめたわけではないのですが、

- `minWidth`, `minHeight`: ウィジェット生成時の最小サイズ
- `minResizeWidth`, `minResizeHeight`: ユーザがウィジェットをリサイズする際の最小サイズ
- `initialLayout`: ウィジェット生成時に用いるレイアウトファイル
- `resizeMode`: ユーザがウィジェットをリサイズする際のリサイズ可能な方向

だと思っています。  
こうして作った`AppWidgetProviderInfo`オブジェクトを`AndroidManifest.xml`に登録します。
```xml
<manifest package="com.geckour.nowplaying4gpm"
          xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:name=".App"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:theme="@style/AppTheme">

        <!-- 中略 -->

        <receiver android:name=".receiver.ShareWidgetProvider">
            <meta-data
                android:name="android.appwidget.provider"
                android:resource="@xml/widget_provider"/>

            <intent-filter>
                <action android:name="android.appwidget.action.APPWIDGET_UPDATE"/>
            </intent-filter>
        </receiver>

    </application>
</manifest>
```
これでウィジェットの準備は完了です。  

ウィジェットの表示内容の更新は、上記`ShareWidgetProvider.kt`内の`updateWidget`と同様の処理を更新したいタイミングで行えば可能です。  
やはり`RemoteView`のインスタンスをどこかに保持しておきたいものの、どこに持つかが悩ましいところ。  
  
今回は以上です。
