---
title: "Android Wear の Watch Face を作ってみる (Wear OS 1.x & 2.0 対応)"
date: 2018-06-04T12:23:44+09:00
draft: false
tags: ["Android", "Android Wear"]
---
# はじめに
Android Wear が世に出て随分経ちますが、先日はじめて Wear アプリの開発に触れてみたので記事にしてみたいと思います。  
サンプルコードは[こちら](https://github.com/geckour/Nocturne)に上がっています。

# 早速作ってみる
## テンプレートからプロジェクト生成
Android Studio にバンドルされている Android Wear のテンプレートからプロジェクトを生成します。

Android Studio を起動し、`File` -> `New` -> `New Project` でプロジェクト作成ウィザードを起動。  
プロジェクトの設定を適切にした後、`Next`を押し、対象 API を選択する画面で`Wear`にチェックを入れます。  
次の画面で`Watch Face`を選択し、次へ進みます。  
ここで、`Analog`か`Digital`、どちらの`Style`にするか聞かれますが、そんなに生成物に大きく影響はしないので、迷ったときは自分が作りたいものがどちらにより近いかで決めると良いと思います。  
なお、この記事では`Digital`を選択した体で話を進めます。

こうして、書くべきコードが予め網羅的に書いてあるプロジェクトができるので、あとは自分好みに修正するだけで済みます。  
そう、ただ作るだけなら Watch Face の開発は非常に簡単なのです。

## ちょいと弄ってみる
まずは、このまま`Run`してみましょう。  
といっても、プロジェクト生成直後はいつもの`Ctrl+R`では実行できないはずです。  
これは、Watch Face は Launch Activity を持たない (`WatchFaceService` を拡張して利用する) ためです。

`Run`ボタンの左側にある`module`を選ぶ小窓の中の`Edit Configurations…`を選択し、`General`タブの中にある`Launch Options`内の`Launch`を`Nothing`にすれば無事`Run`できるようになるはずです。

### 実は使えるレイアウトXMLファイル
[公式ドキュメント](https://developer.android.com/training/wearables/watch-faces/drawing)を見ていると、`Canvas`を使ってゴリゴリ描画していく方法しか載っていないのですが、少し工夫することで普通にレイアウトXMLも使えます。  
`ConstraintLayout`も普通に使えます✨  
ただし、**どうやらData Bindingは使えないっぽい**ので注意してください。

#### レイアウトXMLファイル作成
まずは、適当なレイアウトXMLファイルを作りましょう。

`face.xml` :
```xml
<?xml version="1.0" encoding="utf-8"?>

<android.support.constraint.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
                                             xmlns:app="http://schemas.android.com/apk/res-auto"
                                             xmlns:tools="http://schemas.android.com/tools"
                                             android:layout_width="match_parent"
                                             android:layout_height="match_parent">

    <android.support.constraint.Guideline
        android:id="@+id/guide_box_left"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintGuide_percent="0.1464"/>

    <android.support.constraint.Guideline
        android:id="@+id/guide_box_right"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="vertical"
        app:layout_constraintGuide_percent="0.8536"/>

    <android.support.constraint.Guideline
        android:id="@+id/guide_box_top"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintGuide_percent="0.1464"/>

    <android.support.constraint.Guideline
        android:id="@+id/guide_box_bottom"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintGuide_percent="0.8536"/>

    <android.support.constraint.Guideline
        android:id="@+id/guide_time_secondary"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:orientation="horizontal"
        app:layout_constraintGuide_percent="0.7518"/>

    <android.support.v7.widget.AppCompatTextView
        android:id="@+id/time_primary"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="44sp"
        app:layout_constraintBottom_toBottomOf="@id/guide_box_bottom"
        app:layout_constraintEnd_toEndOf="@id/guide_box_right"
        app:layout_constraintStart_toStartOf="@id/guide_box_left"
        app:layout_constraintTop_toTopOf="@id/guide_box_top"
        tools:text="15:25"/>

    <android.support.v7.widget.AppCompatTextView
        android:id="@+id/time_secondary"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_marginBottom="5dp"
        android:textSize="16sp"
        app:layout_constraintBottom_toTopOf="@id/date"
        app:layout_constraintEnd_toEndOf="@id/time_primary"
        tools:text="07"/>

    <android.support.v7.widget.AppCompatTextView
        android:id="@+id/date"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:textSize="12sp"
        app:layout_constraintBottom_toTopOf="@id/guide_box_bottom"
        app:layout_constraintEnd_toEndOf="@id/guide_box_right"
        app:layout_constraintStart_toStartOf="@id/guide_box_left"
        tools:text="18/05/31 (金)"/>

</android.support.constraint.ConstraintLayout>
```

`android.support.constraint.Guideline`を利用して、Round タイプの Wear におけるセーフエリアを定義しています。

#### レイアウトの`Inflate`
Watch Face における描画処理はほとんど`WatchFaceService.Engine`の中で行われます。  
以下にレイアウトの`Inflate`に関する部分と情報の更新のみに簡略化した`WatchFaceService.Engine`の定義例を示します。

```kotlin
inner class Engine : CanvasWatchFaceService.Engine() {

    private var isAmbient = false

    private lateinit var layout: View

    private lateinit var spec: Size

    override fun onCreate(holder: SurfaceHolder) {
        super.onCreate(holder)

        layout = getSystemService(LayoutInflater::class.java)
                .inflate(R.layout.face, null)

        val displaySize = Point().apply {
            getSystemService(WindowManager::class.java).defaultDisplay
                    .getSize(this)
        }
        spec = Size(
                View.MeasureSpec.makeMeasureSpec(displaySize.x, View.MeasureSpec.EXACTLY),
                View.MeasureSpec.makeMeasureSpec(displaySize.y, View.MeasureSpec.EXACTLY)
        )
    }

    override fun onAmbientModeChanged(inAmbientMode: Boolean) {
        super.onAmbientModeChanged(inAmbientMode)

        isAmbient = inAmbientMode
    }

    override fun onDraw(canvas: Canvas, bounds: Rect) {
        val now = Calendar.getInstance()
        layout.apply {
            measure(spec.width, spec.height)
            layout(0, 0, measuredWidth, measuredHeight)

            findViewById<TextView>(R.id.time_primary).text = now.getPrimaryTimeString()
            findViewById<TextView>(R.id.time_secondary).apply {
                if (isAmbient) visibility = View.GONE
                else {
                    visibility = View.VISIBLE
                    text = now.getTimeSecondString()
                }
            }
            findViewById<TextView>(R.id.date).text = now.getDateString()

            canvas.drawColor(Color.BLACK)
            draw(canvas)
        }
    }

    private fun Calendar.getPrimaryTimeString(): String =
            "%02d:%02d".format(get(Calendar.HOUR_OF_DAY), get(Calendar.MINUTE))

    private fun Calendar.getTimeSecondString(): String =
            "%02d".format(get(Calendar.SECOND))

    private fun Calendar.getDateString(): String =
            "%02d/%02d/%02d(%s)".format(get(Calendar.YEAR) % 100, get(Calendar.MONTH), get(Calendar.DATE), getDayString())

    private fun Calendar.getDayString(): String? =
            when (get(Calendar.DAY_OF_WEEK)) {
                Calendar.MONDAY -> "Mon"
                Calendar.TUESDAY -> "Tue"
                Calendar.WEDNESDAY -> "Wed"
                Calendar.THURSDAY -> "Thu"
                Calendar.FRIDAY -> "Fri"
                Calendar.SATURDAY -> "Sat"
                Calendar.SUNDAY -> "Sun"
                else -> null
            }
}
```
ポイントは、`onCreate`内でディスプレイサイズとその`MeasureSpec`を取得、レイアウトを`Inflate`し、  
`onDraw`で生成済みのレイアウト(`View`)に対して`View#measure()`及び`View#layout()`を呼んでいることです。  
`measure()`で`onCreate`内で取得した`MeasureSpec`を渡した後に`layout()`を呼ぶことで、レイアウトのrootビューがデバイスの画面にフィットしてくれます。

レイアウトに対するデータの流し込みは、`Data Binding`が使えないのでゴリゴリやりましょう…。

#### ゴリッと描画もしたいよね
とはいえ、レイアウトXMLファイルで表現するのが難しいようなデザインを実現するために、Canvasを使いたい場合も当然あると思います。  

もちろん、そんな時は`onDraw`の引数の`Canvas`に対して描画していけば反映されます。  
`Canvas`に対する描画が生成したレイアウトに対する`View.draw(Canvas)`を呼ぶ前か呼んだ後かによって、どちらが手前に来るのかが変わるので注意してください。

以下に秒を円で表示するサンプルを載せておきます。

なお、このタイミングで言うことかという感じはしますが、**Ambientモードの時は画面更新が1回/1分程度に制限される**ので注意してください。

```kotlin
override fun onDraw(canvas: Canvas, bounds: Rect) {
    // 省略

    if (isAmbient.not()) {
        // Round Chinな画面に対応
        val longerSideLength = max(measuredWidth, measuredHeight)
        val circleRect = RectF(
                longerSideLength * 0.04f,
                longerSideLength * 0.04f,
                longerSideLength * 0.96f,
                longerSideLength * 0.96f)

        val paint = Paint().apply {
            strokeWidth = 8f
            style = Paint.Style.STROKE
            color = Color.WHITE
            isAntiAlias = true
        }

        val minute = now.get(Calendar.MINUTE)
        val second = now.get(Calendar.SECOND)
        val milli = now.get(Calendar.MILLISECOND)
        val secondF: Float = second + milli * 0.001f
        val isOdd = minute % 2 == 1

        if (isOdd && secondF < INTERACTIVE_UPDATE_RATE_MS * 0.001) {
            canvas.drawCircle(circleRect.centerX(), circleRect.centerY(), circleRect.width() / 2, paint)
        } else {
            var startAngle = -90f
            var sweepAngle = secondF * 360 / 60

            if (isOdd) {
                sweepAngle = 360f - sweepAngle
                startAngle = -90f - sweepAngle
            }

            canvas.drawArc(circleRect, startAngle, sweepAngle, false, paint)
        }
    }
}
```

## リリース
さて、いい感じの Watch Face もできたことだし、早速リリースだ！  
と思った時に、壁にぶつかると思います。少なくとも僕はぶつかった。

### Wear OS 1.x と 2.0 への対応
Watch Face を Standalone アプリとしてリリースし、かつ configuration Activity を mobile 側に作る場合、1.x と2.0 それぞれへのAPKをビルドする必要があります。  
なお、Standalone アプリで mobile 側のアプリが存在しない場合は 2.0 のみ対応のアプリになるでしょうし、Standalone ではなく Bundled アプリとしてリリースするのであれば 2.0 向けの対応は必要ないでしょう。

公式のドキュメントを読んでいてもイマイチ勘所が掴めなかったので、四苦八苦して正解っぽいところまでたどり着いた結果を記しておきます。

#### `build.gradle`の変更
基本方針は 1.x と 2.0 それぞれのビルドフレーバーを設定し、ビルド時に使い分けるという感じです。

まず、Wear 側の`build.gradle`に以下を追記します。

`build.gradle (Wear)`:
```gradle
android {
    // 省略

    flavorDimensions 'distribute'

    productFlavors {
        wear1 { dimension 'distribute' }
        wear2 {
            dimension 'distribute'
            minSdkVersion 25
        }
    }
}
```
なお、ここで指定している`flavorDimensions`は識別できれば何でも良いです。

次に、Mobile 側の`build.gradle`に以下を追記します。
`build.gradle (Mobile)`:
```gradle
android {
    flavorDimensions 'distribute'

    productFlavors {
        wear1 { dimension 'distribute' }
        wear2 { dimension 'distribute' }
    }
}
```
署名付きAPKのビルド時

- Wear 側のモジュールをビルドする時は `wear2`
- Mobile 側のモジュールをビルドする時は `wear1`

を使います。

なお、Wear 側の`versionCode`と Mobile 側の`versionCode`に同じものを使うことはできないので注意してください。  
バージョン + フレーバー識別子 のように付けることを推奨されているようです。

- バージョン: 13, フーレバー: 1 -> `versionCode 131`
- バージョン: 13, フーレバー: 0 -> `versionCode 130`

こうして出来上がったAPKを、Multiple APK としてリリースします。  
といっても、`リリースを作成`の中で APK を登録する際に、 wear2 フレーバーの Wear 側モジュール APK と、 wear1 フレーバーの Mobile 側モジュール APK の2つをアップロードするだけです。

# おわりに
ただ作るだけなら優秀なテンプレートのおかげで簡単にできる Android Wear の Watch Face。  
リリースで躓いて眠らせるなんてもったいない！ので皆さんも是非素敵な Face をリリースしてください！

~~Wear 0S 2.0で直接インストールした Watch Face の Mobile 側アプリインストールへの導線がなさすぎて辛い~~  
それでは！
