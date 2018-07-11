---
title: "Retrofit を Coroutines と共に使ってみる"
date: 2018-07-10T23:11:59+09:00
draft: false
tags: ["Android"]
---
先日、Kotlin の`Coroutines`が Experimental でなくなったと聞いて、これは本格導入の機運だなと思ったのでひとまず Retrofit + RxKotlin を置き換えようと試みてみました。  
そこで、ざっくりとした導入方法と、Jake 謹製の Coroutines 用 Retrofit アダプタの実装についてつらつらと書いてみたいと思います。

# Retrofit で Coroutines
## Retrofit に Deferred を 扱うためのアダプタを設定する

`build.gradle (module)` :
```gradle
dependencies {
    implementation 'com.jakewharton.retrofit:retrofit2-kotlin-coroutines-experimental-adapter:x.y.z'
}
```

`RetrofitUtil.kt` :
```kotlin
object RetrofitUtil {

    lateinit var retrofit: Retrofit

    fun init(context: Context) {
        retrofit = Retrofit.Builder()
                .addCallAdapterFactory(CoroutineCallAdapterFactory()) // これで Retrofit で Deferred が使えるようになる
                .baseUrl("...")
                .client(createOkHttpClient(context))
                .build()
    }

    private fun createOkHttpClient(context: Context): OkHttpClient =
            OkHttpClient.Builder().build()
}
```

## Deferred で通信を受ける

`Fuga.kt` :
```kotlin
data class Fuga(
    val piyoList: List<Piyo>
) {
    data class Piyo(
        val id: Long,
        val name: String,
        val height: Float
    )
}
```

`HogeService.kt` :
```kotlin
interface HogeService {
    @GET("/fuga")
    fun getFuga(): Deferred<Fuga>
}
```

`HogeClient.kt` :
```kotlin
class HogeClient(private val retrofit: Retrofit = RetrofitUtil.retrofit) {

    suspend fun getPiyoList(): List<Piyo> =
            retrofit.create(HogeService::class.java) // Deferred<Fuga>
                    .await()  // Fuga
                    .piyo // Fuga.piyo (List<Piyo>)
}
```

## これだけ？

これだけです。

これだけで `Deferred` で通信結果を受けられるようになる。すごい。  
あなたも実装が気になってきたはずです。私は気になった。それでは見ていきましょう。

# Kotlin Coroutine (Experimental) Adapter の実装を覗いてみる
## めっちゃ薄い

このアダプタの実装は、[150行足らずの1つのクラス](https://github.com/JakeWharton/retrofit2-kotlin-coroutines-adapter/blob/master/src/main/java/com/jakewharton/retrofit2/adapter/kotlin/coroutines/experimental/CoroutineCallAdapterFactory.kt)にまとまっています。気持ちいいくらい薄いですね。  

## ほんのり解説
まず、Retrofit のアダプタなので`retrofit2.CallAdapter.Factory()`を継承し、その中でさらに`Deferred<T>`が欲しい時用の`CallAdapter`と`Deferred<retrofit2.Response<T>>`が欲しい時用の`CallAdapter`をそれぞれ定義しています。

要求された型が`Deferred`のジェネリクスでない場合は`null`で早期リターンします。
また、要求された型が`Deferred`であるがそのデータ型が指定されていない場合には例外を吐きます。

そこを抜けると後は`Deferred<T>`が欲しいのか`Deferred<Response<T>>`が欲しいのかの判定を行い、それぞれの`CallAdapter`に渡します。

いずれの`CallAdapter`も実装はほとんど同じです。  
まず`CompletableDeferred`のインスタンスを作り、この`Deferred`がキャンセルされた際に通信もキャンセルするために`CompletableDeferred#invokeOnCompletion`に`Call#cancel()`を記述します。

そして、`Call#enqueue`を実行し、コールバックとして、失敗時には`CompletableDeferred#completeExceptionally(Throwable)`を、成功時には`CompletableDeferred#complete(T)`を呼びます。

最後にこうして出来上がった`Deferred`を`return`して完了です。

# まとめ
非同期処理を完結に記述できる Coroutines 。  
薄くて使い勝手の良いライブラリのおかげで今すぐにでも Retrofit と組み合わせて使えますね。

リアクティブプログラミング的なことをしたい場合は同じく Coroutines の`Channel`を使って非同期処理をラップしてあげると良いかもしれません。  
それでは。
