---
title: "Android watch face 再入門"
date: 2022-11-06T11:10:48+09:00
draft: false
tags: ["Android", "Kotlin"]
---

こんにちは。  
今回は、Pixel Watch が発売されたことをきっかけに watch face の開発に再入門してみたのでその概要をお伝えしようと思います。

ビルド関連については[以前の記事](https://blog.geckour.com/posts/2018/0604-create-watch-face/#%E3%83%AA%E3%83%AA%E3%83%BC%E3%82%B9)を参照してください。

サンプルコードは[こちら](https://github.com/geckour/Nocturne)です。

# Service の作成

[以前紹介した](https://blog.geckour.com/posts/2018/0604-create-watch-face/#%E3%83%AA%E3%83%AA%E3%83%BC%E3%82%B9)時とは Service の作成方法が大きく変わっています。

以前は `CanvasWatchFaceService` を継承していましたが、今はそれがなくなって `WatchFaceService` を継承する方法に変わったようです。

特に難しいところはないと思うので、詳細は[サンプル](https://github.com/geckour/Nocturne/blob/master/wear/src/main/java/com/geckour/nocturne/NocturneFaceService.kt)を見てください。

# Canvas への描画

`Renderer.CanvasRenderer2#render()` を override して `Canvas` に描画していきます。

描画時点での時刻情報は `zonedDateTime` に格納されていて、ambient mode を含む現在の表示モードの状態については `renderParameters.drawMode` を参照すればわかります。  
また、画面サイズは `bounds` に格納されています。

`Canvas` への描画なので、描画は先にしたものから順番に下から上へ上書きされていくことに注意してください。

小ネタとして、ミリ秒を含む秒は `zonedDateTime.toInstant().toEpochMilli() % 60000 * 0.001f` のようにすることで取得することができます。

ここであまりに複雑な描画処理などの重い処理を行うと、処理の完了が 1 フレーム分の時間に間に合わずカクカクした描画になってしまうので、最適化を頑張りましょう。  
また、電池持ちを良くするために、ambient mode など画面がアクティブでない時の処理を間引いたり UI のコントラストを抑えたりするなどの対策もしたいところです。  
なお、ambient mode ではフレームレートが 1 fpm 程度に制限されます。

ちなみに、この記事を書いている時点では watch face の描画を Jetpack Compose でできるようにする予定はないようです。

# Watch face の設定項目の追加

Watch face は設定画面を持つことができます。  
そして、その画面で設定した値の反映をどのようにするかを悩む必要はなく、専用のフレームワーク (`UserStyle`) が提供されています。

設定画面では `EditorSession#createOnWatchEditorSession()` を呼び出すことで `EditorSession` が取得できるので、こちらを介して値を更新します。

```kotlin
editorSession.userStyle.value = editorSession.userStyle.value
    .toMutableUserStyle()
    .apply {
        set(
            // ...
        )
    }
    .toUserStyle()
```

Render 側では、`init()` などで以下のようにして値を取り出します。

```kotlin
CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate).launch {
    currentUserStyleRepository.userStyle.collect { userStyle ->
        val hoge = (userStyle[hogeSetting] as UserStyleSetting.BooleanUserStyleSetting.BooleanOption).value
    }
}
```

ここで、watch face がどのような `UserStyle` を持っているのかの指定が必要なことに気づくかと思うのですが、そちらは `WatchFaceService#createUserStyleSchema()` を override して定義します。  
シンプルなので詳しくは[サンプル](https://github.com/geckour/Nocturne/blob/master/wear/src/main/java/com/geckour/nocturne/NocturneFaceService.kt)を参照してください。

# Complication の構成・描画

さて、watch face には外部アプリが (内部からも可能) 提供してくれるデータを表示するための `complication` という概念が存在します。

心拍数や歩数、その他様々なデータなどを表示できる非常に便利なものですが、公式ドキュメントが十分でないため実装はかなり手探りとなりました。  
以下に順を追って説明していきます。

## どのように complecation を表示させたいかを定義する

`WatchFaceService#createComplicationSlotsManager()` を override し、各種設定をしていきます。

- `ComplicationDrawable` によって表示スタイルを定義した Drawable リソースを作成
- データソースの種類を指定
- 描画領域を指定

以上を `ComplicationSlotsManager` のコンストラクタに渡してフィニッシュです。

<sub>`WatchFaceService`:</sub>

```kotlin
override fun createComplicationSlotsManager(currentUserStyleRepository: CurrentUserStyleRepository): ComplicationSlotsManager {
    val defaultCanvasComplicationFactory =
        CanvasComplicationFactory { watchState, invalidateCallback ->
            CanvasComplicationDrawable(
                checkNotNull(ComplicationDrawable.getDrawable(this, R.drawable.complication)), // Complication 描画時に用いられるスタイル定義
                watchState,
                invalidateCallback
            )
        }
    val dataSourcePolicy = DefaultComplicationDataSourcePolicy()

    return ComplicationSlotsManager(
        listOf(
            ComplicationSlot.createRoundRectComplicationSlotBuilder(
                id = 0, // お好きな id を指定する
                defaultCanvasComplicationFactory,
                listOf(
                    ComplicationType.SMALL_IMAGE,
                    ComplicationType.SHORT_TEXT,
                    ComplicationType.RANGED_VALUE,
                    ComplicationType.NO_DATA,
                    ComplicationType.EMPTY,
                    ComplicationType.NOT_CONFIGURED,
                ), // データとしてユーザが選択可能なタイプをフィルタする
                dataSourcePolicy,
                ComplicationSlotBounds(RectF(0.4f, 0.125f, 0.6f, 0.325f)) // ここで設定した bounds に complecation slot は描画される (画面左上を [0f, 0f]、右下を [1f, 1f] とした時の割合で設定する)
            ).build(),
            // ...
        ),
        currentUserStyleRepository
    )
}
```

<sub>`R.drawable.complication`:</sub>

```xml
<?xml version="1.0" encoding="utf-8"?>
<androidx.wear.watchface.complications.rendering.ComplicationDrawable xmlns:app="http://schemas.android.com/apk/res-auto"
    app:borderStyle=...
    app:highlightColor=...
    app:iconColor=...
    app:rangedValuePrimaryColor=...
    app:rangedValueRingWidth=...
    app:rangedValueSecondaryColor=...
    app:textColor=...
    app:textTypeface=...
    app:titleColor=...
    app:titleTypeface=...>

    <ambient
        app:borderColor=...
        app:iconColor=...
        app:rangedValuePrimaryColor=...
        app:rangedValueSecondaryColor=...
        app:textColor=...
        app:titleColor=... />
</androidx.wear.watchface.complications.rendering.ComplicationDrawable>
```

## どの complecation slot に何のデータを表示するかを選べるようにする

データの入れ物は定義できたので、次にユーザがどの入れ物に何のデータを入れるかを指定できるようにします。

Watch face の設定画面に項目を追加すると良いと思います。  
データの選択・complecation slot への設定自体は `EditorSession#openComplicationDataSourceChooser()` を呼び出すだけでよく簡単です。

ただ、設定されたデータのアイコンを描画したいとなるとかなりまどろっこしい処理をしなければならないので覚悟してください。  
以下にアイコン描画をする例を示します (Jetpack Compose を使用しています) 。

```kotlin
private var editorSession: EditorSession? = null

override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)

    lifecycleScope.launchWhenCreated {
        editorSession = EditorSession.createOnWatchEditorSession(this@HogeActivity)
    }

    setContent {
        val editorSession: EditorSession? by remember { mutableStateOf(editorSession) }
        val coroutineScope = rememberCoroutineScope()

        val previewData = editorSession?.complicationsPreviewData?.collectAsState() // Complecation slot それぞれのプレビュー描画用ダミーデータ
        val dataSourceInfo = editorSession?.complicationsDataSourceInfo?.collectAsState() // 互換性確保のためのプレビュー描画用ダミーデータ
        val slotState = editorSession?.complicationSlotsState?.collectAsState() // プレビュー描画時の描画領域を指定するために使用

        val complicationData1 = previewData?.value?.get(1) ?: dataSourceInfo?.value?.get(1)?.fallbackPreviewData
        val actualBounds1 = slotState?.value?.get(1)?.bounds

        ComplicationButton(
            coroutineScope = coroutineScope,
            id = 1,
            complicationData = complicationData1,
            actualBounds = actualBounds1
        )
    }
}

@Composable
fun ComplicationButton(
    modifier: Modifier = Modifier,
    coroutineScope: CoroutineScope,
    id: Int,
    complicationData: ComplicationData?,
    actualBounds: Rect?
) {
    actualBounds ?: return

    Button(
        modifier = Modifier.fillMaxSize(),
        onClick = {
            coroutineScope.launch {
                editorSession?.openComplicationDataSourceChooser(id)
            }
        }
    ) {
        when (complicationData?.type) {
            null,
            ComplicationType.NO_DATA,
            ComplicationType.NOT_CONFIGURED,
            ComplicationType.EMPTY -> {
                Icon(imageVector = Icons.Rounded.Add, "Add additional function")
            } // 表示するデータがない場合にデフォルトアイコン (ここでは追加ボタン) を表示する
            else -> {
                ComplicationDrawable(this@HogeActivity).run {
                    bounds = actualBounds
                    setComplicationData(complicationData, false)
                    Image(
                        modifier = Modifier.fillMaxSize(),
                        bitmap = current
                            .toBitmap(
                                width = actualBounds.width(),
                                height = actualBounds.height(),
                                config = Bitmap.Config.ARGB_8888
                            )
                            .asImageBitmap(),
                        contentDescription = "Additional function preview"
                    )
                }
            }
        }
    }
}
```

## Complecation slot を描画する

ここまで設定できたら、ようやく complecation slot を描画できるようになります。

`Renderer.CanvasRenderer2#render()` と `Renderer.CanvasRenderer2#renderHighlightLayer()` で、それぞれ

```kotlin
complicationSlotsManager.complicationSlots.forEach { (_, complication) ->
    if (complication.enabled) {
        complication.render(canvas, zonedDateTime, renderParameters)
    }
}
```

このようにしてあげれば良いです。

ただし、上記の設定ではユーザが complecation slot に有効なデータソースを設定していない場合、complecation が描画されるべき箇所には何も表示されないので注意してください。

# スマートフォンからの大きなデータの同期

スマートフォン側の設定アプリから画像などの大きなデータを watch face アプリに同期させたい場合 (背景画像を指定できるようにしたい場合など) 、`DataClient` 経由でデータをセットし、`WatchFaceService` で受け取るのが一般的かと思います。

しかし、`WatchFaceService#createWatchFace()` 内などで

```kotlin
Wearable.getDataClient(this)
    .addListener {
        // ...
    }
```

この様にしたとしても、リスナーの呼び出しを待つことができず結果を取得できません。

`createWatchFace()` は `suspend fun` なので、依存ライブラリに `org.jetbrains.kotlinx:kotlinx-coroutines-play-services` を追加し

```kotlin
Wearable.getDataClient(this)
    .addListener {
        // ...
    }
    .await()
```

この様にすることで無事データを取得することができます。

# 終わりに

記事としてはなるべくコンパクトになるようにまとめてみましたが、実際実装するとなると躓く場面は多々あるかと思います。  
そんな折には[サンプルアプリのコード](https://github.com/geckour/Nocturne)を眺めつつ、素敵な watch face を作っていただければ幸いです。

それでは！
