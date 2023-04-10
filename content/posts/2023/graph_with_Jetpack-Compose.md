---
title: "Jetpack Compose と Canvas でグラフを作ってみた"
date: 2023-04-10T10:28:53+09:00
draft: false
tags: ["Android", "Jetpack Compose", "Kotlin"]
---

こんにちは。  
今回は面倒くさそうだなーと思って手を付けずに放置していたグラフ表示を Jetpack Compose でやってみた紹介です。

結論を先に書くとめちゃくちゃ楽だし楽しいです。

なお今回の成果物は [このあたり](https://github.com/geckour/HomeAPI/blob/main/app/src/main/java/com/geckour/homeapi/ui/main/Dialog.kt) にあります。

<img src="/movies/screen-20230410-131621-2.gif" alt="Sample GIF animation" style="width: 40%; margin: auto; display: block;">

# 今回のグラフ

まず前提を整理しておきます。

グラフにしたいデータは気温、湿度、気圧などを定期的に記録したものです。  
折れ線グラフが適しているかな、と思ったのでその線で検討を進めました。

また、データの件数ですが、おおよそ 10 分毎に記録しているので 1 日分で 144 件、1 週間分で 1004 件、1 ヶ月分で 30240 件程となります。

# 折れ線グラフの表現方法

さて、どうやって折れ線グラフを描画するかですが、LazyRow なんかも検討してみましたが

- データ件数と画面幅的にギュウギュウになってしまう
- パフォーマンスがあまり良くなさそう
- 今回時間軸 (横軸) でのスクロールは考えていないのでリサイクルによる恩恵もなさそう

これらから Canvas にゴリゴリ書いていく方針にしました。

## Canvas で折れ線グラフ

Canvas に折れ線を描く事自体は `Path` と `Canvas#drawPath()` を使えば簡単なのですが、問題はどのように `Path` を生成する (と効率的) かということです。  
今回描きたい折れ線は計 4 本 (気温、湿度、気圧、土壌湿度) だったので、それぞれで使い回せる形にしました。

```kotlin
fun getPlot(
    canvasSize: Size,
    padding: RectF,
    value: Float,
    maxValue: Float,
    minValue: Float,
    time: Date,
    start: Date,
    end: Date
): PointF {
    val paddingLeft = padding.left
    val paddingBottom = padding.bottom
    val width = canvasSize.width - (paddingLeft + padding.right)
    val height = canvasSize.height - (padding.top + paddingBottom)
    val timeRange = end.time - start.time
    val timeOffset = time.time - start.time
    val valueRange = maxValue - minValue
    val valueOffset = maxValue - value

    return PointF(paddingLeft + width * timeOffset / timeRange, paddingBottom + height * valueOffset / valueRange)
}
```

上記関数を各データごとに呼び出して `Path` を形成していきます。

ポイントは、表示する時間レンジを `start` `end` として、また対象の値に対する最大最小レンジを `maxValue` `minValue` として渡し、`Canvas` の大きさ等から計算することで再利用性を高めていることです。  
いちいち `paddingHoge` `width` `height` `timeRange` 等を計算することで無駄が発生していますが、そのおかげで何も考えずに値を突っ込むことができているとも言えます。

この辺はパフォーマンスと相談すると良さそうですが、今回のケースではそれほど問題なさそうだったのでこのまま行くことにしました。

ちなみに `maxValue` や `minValue` を実際のデータの最大値・最小値ではなく任意の値に変えることで、グラフの表示範囲を自在に変えることができます。

# インタラクティブなグラフ

さて、ここまででひとまず折れ線グラフを表示することはできるようになりましたが、このままではただ線が出ているだけで値がよくわかりません。  
そこでグラフをタップしたら最近傍の値を取得してポップアップ表示できるようにしようと思います。

Canvas にタップのインタラクションをつけるには、他の Composable と同じく `modifier` に `pointerInteropFilter` を指定した `Modifier` を渡してあげます。

```kotlin
data class Log(
    val temperature: Float,
    val humidity: Float,
    val pressure: Float,
    val time: Date,
)

@Composable
fun Graph(logs: List<Log>) {
    var selectedLog by remember { mutableStateOf<Log?>(null) }

    var width by remember { mutableStateOf(0f) }

    val start = logs.minByOrNull { it.time } ?: return
    val end = logs.maxByOrNull { it.time } ?: return

    Canvas(
        modifier = Modifier
            .pointerInteropFilter { event ->
                return@pointerInteropFilter when (event.action) {
                    MotionEvent.ACTION_DOWN,
                    MotionEvent.ACTION_POINTER_DOWN,
                    MotionEvent.ACTION_MOVE -> {
                        val pointedTime = (start.time + (end.time - start.time) * event.x / width).toLong()
                        selectedLog = logs.minBy { abs(it.date.time - pointedTime) }
                    }
                    MotionEvent.ACTION_UP,
                    MotionEvent.ACTION_POINTER_UP -> {
                        selectedLog = null
                        true
                    }
                    else -> false
                }
            },
        onDraw = {
            width = size.width // Canvas 自体の横幅。雑だがこうして取得するのが楽だった

            // ...

            selectedLog.time?.let {
                val top = getPlot(size, graphPadding, 1f, 0f, 1f, it, start, end)
                val bottom = getPlot(size, graphPadding, 0f, 0f, 1f, it, start, end)
                drawLine(Color.White, Offset(top.x, top.y), Offset(bottom.x, bottom.y))
            }
        }
    )
}
```

`pointerInteropFilter` 内で更新した `selectedLog` を用いて `onDraw` で縦線を引いています (なおその際に [先程紹介した](#canvas-で折れ線グラフ) `getPlot()` を利用しています) 。  
また、`MotionEvent.ACTION_UP` と `MotionEvent.ACTION_POINTER_UP` で `selectedLog` に `null` を代入することで指を離したときに線が消えるようにしました。

もちろんこの `selectedLog` を使って値の詳細を別途表示したりすることもできます。

# グラフのデータ更新

最新データを取得し直したりデータの範囲を変えたりしたいときも、データを `State` で保持したり `StateFlow` を `collectAsState()` して保持したりしておくことで Jetpack Compose の recompose により非常にスムーズにグラフを更新することが可能です。

今回の僕のケースではグラフを `AlertDialog` 内に描画しているのですが、表示範囲を変えるボタンを押すと裏側で通信が走り、通信が終わると新しいデータがセットされてシームレスにグラフが更新されるという流れが実現できています。

Jetpack Compose による宣言的 UI は色々と考えることが減らせて便利ですね。

# まとめ

ライブラリの制約に縛られたりすることなく自由自在に、しかも簡単にグラフを作れるのは本当に楽しいです。  
みなさんもぜひこの快感を体感してください！

それでは。
