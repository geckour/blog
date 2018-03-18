---
title: "Androidアプリにアプリ内決済 (In-app Billing) APIを導入する"
date: 2018-03-18T13:50:57+09:00
draft: false
tags: ["Android"]
---
随分前にIn-app Billingを個人アプリに適当に入れてみたことがあったのですが、その時はエラーハンドリングを一切しないまま放置したのでそろそろきちんとやってみよう、という記事です。

## In-app Billing API
言わずもがな、アプリ内課金を実現するAPIです。  
基本的に[ドキュメント](https://developer.android.com/google/play/billing/billing_integrate.html)を順番に追っていけば実装できますが、要点だけ知りたい方のためにまとめておこうと思います。

### AIDLファイルの追加
[ここ](https://developer.android.com/google/play/billing/billing_integrate.html#billing-add-aidl)を見てIn-app Billng APIのインタフェースを生やすためのAIDLファイルをプロジェクトに追加してください。  
注意すべき点は、

- Android Studioに内蔵されているSDKマネージャを使用している場合、`1.`の手順はすっ飛ばす
- `2.b`の実行後、`src/main/aidl`に進んでから`2.c`を実行する
- 自身のAndroid SDKの設置場所はSDKマネージャを開けば確認可能

といったところです。

### マニフェストの更新
Google Playに課金処理を移譲するための権限をマニフェストに追記します。  
`<uses-permission android:name="com.android.vending.BILLING" />`

### プロジェクトのビルド
AIDLファイルからIn-app Billing APIのインタフェースを生成します。

### 課金処理
ここまでで課金処理を実装するための準備は終了です。  
実際の課金処理のサンプルを、[NowPlaying4GPM](https://github.com/geckour/NowPlaying4GPM)の実装を基に紹介します。  

In-app Billing APIに対する操作はすべて、AIDLファイルをビルドすることで生成される`IInAppBillingService`を介して行います。  
今回は、基本的な操作を一つのクラスにまとめてみました。

`BillingApiClient.kt` :
```kotlin
import android.app.PendingIntent
import android.content.Context
import android.os.Bundle
import com.android.vending.billing.IInAppBillingService
import com.geckour.nowplaying4gpm.api.model.SkuDetail
import com.google.gson.Gson

class BillingApiClient(private val service: IInAppBillingService) {

    enum class ResponseCode(val code: Int) {
        RESPONSE_OK(0)
    }

    companion object {
        const val API_VERSION = 3
        const val BILLING_TYPE = "inapp"
        const val BUNDLE_KEY_RESPONSE_CODE = "RESPONSE_CODE"
        const val BUNDLE_KEY_SKU_DETAIL_LIST = "DETAILS_LIST"
        const val BUNDLE_KEY_PURCHASE_ITEM_LIST = "INAPP_PURCHASE_ITEM_LIST"
        const val BUNDLE_KEY_PURCHASE_DATA_LIST = "INAPP_PURCHASE_DATA_LIST"
        const val BUNDLE_KEY_BUY_INTENT = "BUY_INTENT"
        const val BUNDLE_KEY_PURCHASE_DATA = "INAPP_PURCHASE_DATA"
        const val QUERY_KEY_SKU_DETAILS = "ITEM_ID_LIST"
    }

    suspend fun getPurchasedItems(context: Context): List<String> = async {
        service.getPurchases(
                API_VERSION,
                context.packageName,
                BILLING_TYPE,
                null
        ).getStringArrayList(BUNDLE_KEY_PURCHASE_ITEM_LIST)
    }.await()

    suspend fun getSkuDetails(context: Context, vararg skus: String): List<SkuDetail> =
            async {
                service.getSkuDetails(
                        API_VERSION,
                        context.packageName,
                        BILLING_TYPE,
                        Bundle().apply {
                            putStringArrayList(
                                    QUERY_KEY_SKU_DETAILS,
                                    ArrayList(skus.toList()))
                        }
                ).let {
                    if (it.getInt(BUNDLE_KEY_RESPONSE_CODE) == ResponseCode.RESPONSE_OK.code) {
                        it.getStringArrayList(BUNDLE_KEY_SKU_DETAIL_LIST).map {
                            Gson().fromJson(it, SkuDetail::class.java)
                        }
                    } else listOf()
                }
            }.await()

    fun getBuyIntent(context: Context, sku: String): PendingIntent? =
            service.getBuyIntent(API_VERSION, context.packageName, sku, BILLING_TYPE, null)?.let {
                if (it.containsKey(BUNDLE_KEY_RESPONSE_CODE)
                        && it.getInt(BUNDLE_KEY_RESPONSE_CODE) == 0)
                    it.getParcelable(BUNDLE_KEY_BUY_INTENT)
                else null
            }
}
```

課金処理を実際に呼び出す`Activity`のフローのサンプルを示します。

`SettingsActivity.kt` :
```kotlin
import android.Manifest
import android.app.Activity
import android.app.AlertDialog
import android.content.*
import android.content.pm.PackageManager
import android.os.Bundle
import android.os.IBinder
import com.android.vending.billing.IInAppBillingService
import com.geckour.nowplaying4gpm.BuildConfig
import com.geckour.nowplaying4gpm.R
import com.geckour.nowplaying4gpm.api.BillingApiClient
import com.geckour.nowplaying4gpm.api.model.PurchaseResult
import com.geckour.nowplaying4gpm.databinding.ActivitySettingsBinding
import com.google.gson.Gson
import timber.log.Timber

class SettingsActivity : Activity() {

    enum class IntentSenderRequestCode {
        BILLING
    }

    companion object {
        fun getIntent(context: Context): Intent =
                Intent(context, SettingsActivity::class.java)
    }

    private lateinit var binding: ActivitySettingsBinding
    private lateinit var serviceConnection: ServiceConnection
    private var billingService: IInAppBillingService? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = DataBindingUtil.setContentView(this, R.layout.activity_settings)

        binding.itemDonate?.apply {
            root.setOnClickListener {
                launch(UI) { startBillingTransaction(BuildConfig.SKU_KEY_DONATE) }
            }
        }

        serviceConnection = object : ServiceConnection {
            override fun onServiceDisconnected(name: ComponentName?) {
                billingService = null
            }

            override fun onServiceConnected(name: ComponentName?, service: IBinder?) {
                IInAppBillingService.Stub.asInterface(service).apply {
                    billingService = IInAppBillingService.Stub.asInterface(service)
                }
            }
        }
        bindService(
                Intent("com.android.vending.billing.InAppBillingService.BIND")
                        .apply { `package` = "com.android.vending" }, // 他アプリに課金リクエストを傍受されないためにパッケージを指定する
                serviceConnection,
                Context.BIND_AUTO_CREATE
        )
    }

    override fun onDestroy() {
        super.onDestroy()

        billingService?.apply { unbindService(serviceConnection) }
    }

    override fun onActivityResult(requestCode: Int, resultCode: Int, data: Intent?) {
        super.onActivityResult(requestCode, resultCode, data)

        when (requestCode) {
            IntentSenderRequestCode.BILLING.ordinal -> {
                when (resultCode) {
                    Activity.RESULT_OK -> {
                        data?.getStringExtra(BillingApiClient.BUNDLE_KEY_PURCHASE_DATA)?.apply {
                            var success = false
                            try {
                                val purchaseResult =
                                        Gson().fromJson(this, PurchaseResult::class.java)

                                success = purchaseResult.purchaseState == 0
                            } catch (e: Exception) {
                                Timber.e(e)
                            }

                            if (success.not()) {
                                showErrorDialog(
                                        R.string.dialog_title_alert_failure_purchase,
                                        R.string.dialog_message_alert_failure_purchase)
                            }
                        }
                    }

                    Activity.RESULT_CANCELED -> {
                        showErrorDialog(
                                R.string.dialog_title_alert_failure_purchase,
                                R.string.dialog_message_alert_on_cancel_purchase)
                    }
                }
            }
        }
    }

    private suspend fun startBillingTransaction(skuName: String) {
        billingService?.let {
            BillingApiClient(it).apply {
                val sku =
                        getSkuDetails(this@SettingsActivity, skuName).firstOrNull()
                                ?: run {
                                    showErrorDialog(R.string.dialog_title_alert_failure_purchase, R.string.dialog_message_alert_on_start_purchase)
                                    return
                                }

                if (getPurchasedItems(this@SettingsActivity).contains(sku.productId)) {
                    showErrorDialog(R.string.dialog_title_alert_failure_purchase, R.string.dialog_message_alert_already_purchase)
                    return
                }
            }

            startIntentSenderForResult(
                    BillingApiClient(it)
                            .getBuyIntent(this@SettingsActivity, skuName)
                            ?.intentSender,
                    IntentSenderRequestCode.BILLING.ordinal,
                    Intent(), 0, 0, 0)
        }
    }

    private fun showErrorDialog(titleResId: Int, messageResId: Int) =
            AlertDialog.Builder(this)
                    .setTitle(titleResId)
                    .setMessage(messageResId)
                    .setPositiveButton(R.string.dialog_button_ok) { dialog, _ -> dialog.dismiss() }
                    .show()
}
```
流れとしては、

- `onCreate`内で`ServiceConnection`を用いて`IInAppBillingService`のインスタンスを取得
- 課金処理を開始したいタイミングで`startBillingTransaction`を呼ぶ
  - `IInAppBillingService`のインスタンスが生成済み
    - 指定された`sku`の情報をClient経由で取得
    - 取得した`sku`情報のIDが購入済みアイテムリストに含まれているかClient経由で照会
      - 含まれていなければ購入処理をClient経由で得た`BuyIntent`を用いて`startIntentSenderForResult`によって開始
- 課金処理の結果が`onActivityResult`に返ってくる
  - `requestCode`をチェック
    - `RESULT_OK`
      - 購入処理が完遂された
      - レスポンス (JSON) をパース
      - `purchaseState == 0`なら購入成功
    - `RESULT_CANCELED`
      - 購入処理が途中でキャンセルされた

このようになっています。

使用しているData classはこちら

`SkuDetail.kt` :
```kotlin
import com.google.gson.annotations.SerializedName

data class SkuDetail(
        val productId: String,

        val type: String,

        val price: String,

        @SerializedName("price_amount_micros")
        val priceInMicros: String,

        @SerializedName("price_currency_code")
        val priceCode: String,

        val title: String,

        val description: String
)
```

`PurchaseResult.kt` :
```kotlin
data class PurchaseResult(
        val autoRenewing: Boolean,
        val orderId: String,
        val packageName: String,
        val productId: String,
        val purchaseTime: Long,
        val purchaseState: Int,
        val developerPayload: String?,
        val purchaseToken: String
)
```
