---
title: "BottomSheetBehavior の state と　OnBackPressedDispatcher を連携する"
date: 2021-09-14T00:41:29+09:00
draft: false
tags: [Android]
---

どうもこんにちは、geckour です。  
今回は、タイトルの通り BottomSheetBehavior の state によって OnBackPressedDispatcher の制御をしようとして若干手間取ったのでまとめておきます。

# 実装

やりたいことは `BottomSheetBehavior#state` が `BottomSheetBehavior.STATE_EXPANDED` の場合のみバックキーの制御を奪い、ボトムシートを閉じるというものです。  
が、実装を見てもらえればそれが全てだと思うので以下に示します。

```kotlin
class HogeFragment() : Fragment() {

    override fun onAttach(context: Context) {
        super.onAttach(context)

        val behavior = BottomSheetBehavior<View>.from(
            // The view that has been added an attribute of app:layout_behavior="com.google.android.material.bottomsheet.BottomSheetBehavior"
            (requireActivity() as HogeActivity).binding.bottomSheet
        )
        val bottomSheetCallback = object : BottomSheetBehavior.BottomSheetCallback() {
            override fun onSlide(v: View, dy: Float) {
                binding.sheet.progress = dy
            }

            override fun onStateChanged(v: View, state: Int) {
                onBackPressedCallback.isEnabled = state == BottomSheetBehavior.STATE_EXPANDED
            }
        }
        behavior.addBottomSheetCallback(bottomSheetCallback)

        onBackPressedCallback =
            object : OnBackPressedCallback(behavior.state == BottomSheetBehavior.STATE_EXPANDED) {
                override fun handleOnBackPressed() {
                    viewModel.toggleSheetState.value = Unit
                }
            }

        requireActivity().onBackPressedDispatcher.addCallback(this, onBackPressedCallback)
    }
}
```

# (一応) 解説

軽く解説もしておきます。

まず、 `OnBackPressedDispatcher` では `Activity#onBackPressed()` のように `super.onBackPressed()` を呼んでデフォルトのバックキーの制御を行うということができません。  
そこで、`OnBackPressedDispatcher#isEnabled` の値を変化させることによってコールバックの実行を制御できる仕組みを利用します。

具体的には、 `OnBackPressedDispatcher` を以下のようにラムダ式を使って宣言するのではなく、インスタンスとして保持します。  
その上で、`OnBackPressedDispatcher#isEnabled` は外部から `set` できるので、`BottomSheetBehavior.BottomSheetCallback` を利用して `state` を見て値を制御してあげます。  
`OnBackPressedDispatcher#isEnabled` は外部から `set` できる、ここが一番大事で、それさえ押さえておけば色々応用が効くと思います。
