---
title: "WorkManager を導入してみる"
date: 2019-02-19T14:15:06+09:00
draft: false
tags: ["Android"]
---
`WorkManager`が世に出てしばらく経ちました。  
この辺で、頭の整理がてらその実装方法をまとめておきたいと思います。  
本記事の内容は大体[公式ドキュメント](https://developer.android.com/topic/libraries/architecture/workmanager/)に載っていて、なおかつそちらのほうが詳しいかと思いますのでご了承ください。

# WorkManager
バックグラウンドで実行したい処理をうまく扱うためのAPI。  
基本的にはUIと切り離せるような処理をお任せするような用途で用います。  
今までは`AlarmManager`や`JobScheduler`をAPIバージョンによって使い分けたりしていましたが、`WorkManager`の登場によってそれらを意識せず統合的に扱うことができるようになりました。  
また、`AlarmManager`や`JobScheduler`では実現が難しかったことも`WorkManager`では簡単にできるような機能も追加されています。

# 導入
## ライブラリの導入
まずは依存ライブラリを定義してあげましょう。

`[module]/build.gradle`:
```gradle
dependencies {
    def work_version = "x.y.z"
    implementation "android.arch.work:work-runtime-ktx:$work_version"
}
```

`[module]/build.gradle.kts`:
```kotlin
dependencies {
    val work_version = "x.y.z"
    implementation("android.arch.work:work-runtime-ktx:$work_version")
}
```
最新のバージョンなど詳しくは[こちら](https://developer.android.com/jetpack/androidx/releases/work)へ

## Workerをつくる
それでは早速ハリボテの`Worker`を作ってみます。

`SampleWorker.kt`:
```kotlin
import android.content.Context
import androidx.work.Result
import androidx.work.Worker
import androidx.work.WorkerParameters

class SampleWorker(
    context: Context,
    workerParameters: WorkerParameters
) : Worker(context, workerParameters) {

    override fun doWork(): Result {
        return Result.success()
    }
}
```
はい。たったこれだけです。  
ここから`doWork()`の中にやりたい処理をガリガリ書いていきます。  
が、あまり重たい処理を書いてしまうとタイムアウトが発生して`Worker`がコケるので、いくつかの処理ブロックに分けて複数の`Worker`を作り、チェーンする必要があります([後述](#複数のworkerを繋げる))。  
`doWork()`の返り値である`Result`には、

- `Result.success()`
- `Result.failure()`
- `Result.retry()`

が定義されており、使い分けることで挙動を制御できます。

## Workerを起動する
作ったWorkerを起動してみましょう。  
とその前に、起動するための`static`メソッドをWorker側に定義してしまいましょう。

`SampleWorker.kt`:
```kotlin
import android.content.Context
import androidx.work.ExistingWorkPolicy
import androidx.work.Result
import androidx.work.ExistingWorkPolicy
import androidx.work.OneTimeWorkRequest
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.Worker
import androidx.work.WorkerParameters
import java.util.UUID

class SampleWorker(
    context: Context,
    workerParameters: WorkerParameters
) : Worker(context, workerParameters) {

    override fun doWork(): Result {
        return Result.success()
    }

    companion object {
        private const val NAME = "SampleWorker"

        fun start(): UUID {
            val work = getWork()
            WorkManager.getInstance()
                .beginUniqueWork(NAME, ExistingWorkPolicy.APPEND, work)
                .enqueue()

            return work.getId() // 後々このUUIDを使います
        }

        fun getWork(): OneTimeWorkRequest =
            OneTimeWorkRequestBuilder<SampleWorker>().build()
    }
}
```
これで`SampleWorker.start()`とするだけで起動できるようになりました。

## 起動したWorker(WorkRequest)をキャンセルする
動いている`WorkRequest`をキャンセルするには、その`WorkRequest`のIDが必要になります。  
また、キャンセルはベストエフォートで実行され、即時実行は保証されないので注意が必要です。

```kotlin
val workId: UUID = SimpleWorker.start()
WorkManager.getInstance().cancelWorkById(workId)
```

## 複数のWorkerを繋げる
[先述](#workerをつくる)した複数の`Worker`をチェーンする方法です。  
非常に簡潔に、わかりやすく記述することができます。

```kotlin
WorkManager.getInstance()
    .beginWith(SampleWorkerA.getWork())
    .then(SampleWorkerB.getWork())
    .then(SampleWorkerC.getWork())
    .enqueue()
```

また、複数のチェーンを束ねて一つの`Worker`に繋げることも可能です。
```kotlin
// チェーン1本目
val chain1 = WorkManager.getInstance()
    .beginWith(SampleWorkerA.getWork())
    .then(SampleWorkerB.getWork())
// チェーン2本目
val chain2 = WorkManager.getInstance()
    .beginWith(SampleWorkerC.getWork())
    .then(SampleWorkerD.getWork())
// 束ねる
val chain3 = WorkManager.getInstance()
    .combine(listOf(chain1, chain2))
    .then(SampleWorkerE.getWork())
// エンキュー
chain3.enqueue()
```

## 入力と出力
`Worker`には入出力が存在し、入力は自身への引数として、出力はチェーンした際の次の`Worker`への引数として利用できます。

`SampleWorker.kt`:
```kotlin
import android.content.Context
import androidx.work.Data
import androidx.work.ExistingWorkPolicy
import androidx.work.Result
import androidx.work.ExistingWorkPolicy
import androidx.work.OneTimeWorkRequest
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.Worker
import androidx.work.WorkerParameters
import java.util.UUID

class SampleWorker(
    context: Context,
    workerParameters: WorkerParameters
) : Worker(context, workerParameters) {

    override fun doWork(): Result {
        val hoge = inputData.getString(KEY_HOGE) ?: return Result.failure()
        val output = mapOf(KEY_HOGE_HOGE, "$hoge$hoge").toWorkData() // 出力データの生成
        return Result.success(output)
    }

    companion object {
        private const val NAME = "SampleWorker"
        private const val KEY_HOGE = "keyHoge"
        const val KEY_HOGE_HOGE = "keyHogeHoge"

        fun start(): UUID {
            val work = getWork()
            WorkManager.getInstance()
                .beginUniqueWork(NAME, ExistingWorkPolicy.APPEND, work)
                .enqueue()

            return work.getId()
        }

        fun getWork(): OneTimeWorkRequest =
            OneTimeWorkRequestBuilder<SampleWorker>()
                .setInputData(
                    Data.Builder()
                        .putString(KEY_HOGE, "fuga") // Stringの入力を指定
                        .build()
                )
                .build()
    }
}
```
入出力の処理は`doWork()`の中で行います。

## Workerの実行に条件を設ける
`Worker`には`Constraints`というものを設定できます。  
これは、`Worker`の実行条件を定義するもので、

- トリガーとなる content URI の指定
- 充電状態の要求
- 端末の待機状態の要求
- 十分なストレージ残容量の要求
- 特定の通信状態の要求
- 十分な電池残量の要求

が定義できます。詳しくは[公式リファレンス](https://developer.android.com/reference/kotlin/androidx/work/Constraints.Builder)をご参照ください。

`SampleWorker.kt`:
```kotlin
import android.content.Context
import androidx.work.Constraints
import androidx.work.ExistingWorkPolicy
import androidx.work.Result
import androidx.work.ExistingWorkPolicy
import androidx.work.OneTimeWorkRequest
import androidx.work.OneTimeWorkRequestBuilder
import androidx.work.WorkManager
import androidx.work.Worker
import androidx.work.WorkerParameters
import java.util.UUID

class SampleWorker(
    context: Context,
    workerParameters: WorkerParameters
) : Worker(context, workerParameters) {

    override fun doWork(): Result {
        return Result.success()
    }

    companion object {
        private const val NAME = "SampleWorker"

        fun start(): UUID {
            val work = getWork()
            WorkManager.getInstance()
                .beginUniqueWork(NAME, ExistingWorkPolicy.APPEND, work)
                .enqueue()

            return work.getId()
        }

        fun getWork(): OneTimeWorkRequest =
            OneTimeWorkRequestBuilder<SampleWorker>()
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED) // ネットワーク接続を要求
                        .build()
                )
                .build()
    }
}
```

# まとめ
以上、簡単にWorkManagerを紹介してみました。  
本記事執筆時点ではまだ安定版はリリースされていませんが、非常に強力なAPIを、簡潔に分かりやすく扱うことができるということが、この記事を通して伝わっていれば幸いです。  
詳しいことは[公式ドキュメント](https://developer.android.com/topic/libraries/architecture/workmanager/)を読もうな！（台無し）
