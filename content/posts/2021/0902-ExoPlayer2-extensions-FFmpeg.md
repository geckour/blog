---
title: "ExoPlayer2 の FFmpeg extension をビルドしたときのメモ"
date: 2021-09-02T23:49:00+09:00
draft: false
tags: [Android, ExoPlayer]
---

こんにちは。急に秋が来て戸惑っている geckour です。  
今回はたまーに必要に駆られて作業するたびに詳細を忘れてつまづく ExoPlayer の FFmpeg extension のビルドについてメモしておこうと思います。

# 手順

基本は[公式ドキュメント](https://github.com/google/ExoPlayer/tree/release-v2/extensions/ffmpeg)に従います。

その上で、NDK は最新バージョンだと駄目でした。  
現状 `21.0.6113669` で動かしてうまく行っています。

また、CMake をインストールしていない場合、うまく行かない可能性があるので入れておいたほうが良いかもしれません。 (SDK Manager 経由で入れられます)  
さらに、Ninja というビルドツールに内部で依存しているようなのですが、依存をうまく解決できないので[パッケージマネージャ経由など](https://github.com/ninja-build/ninja/wiki/Pre-built-Ninja-packages)でインストールしてください。

ビルドした extensions の追加方法は[ここ](https://github.com/google/ExoPlayer/blob/02f7aafe67b4893134a31ea55a3ba0f0535df145/README.md#locally) に書いてあります。

```gradle
implementation project(':exoplayer-library-core')
implementation project(':exoplayer-extension-ffmpeg')
```

こんな感じで依存を入れれば OK です。

ExoPlayer への導入方法も一応言及しておくと、以下の要領です。

```kotlin
val renderersFactory = DefaultRenderersFactory(context)
    .setExtensionRendererMode(DefaultRenderersFactory.EXTENSION_RENDERER_MODE_PREFER)

SimpleExoPlayer.Builder(context, renderersFactory)
    ...
    .build()
```
