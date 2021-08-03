---
title: "Android と UTF-16 と Iterator"
date: 2021-08-04T01:26:15+09:00
draft: false
tags: ["Android", "Kotlin"]
---

こんにちは。  
今回は Android アプリ開発中に文字列を弄っていたらハマった話をしようかと思います。

# Android と UTF-16

さて、Android における Kotlin は JVM の上で動いていることはご存知かと思いますが、Java において `String` は UTF-16 でエンコードされていることをご存じの方は意外と少ないかもしれません。僕は知りませんでした。

そして、UTF-16 では、8 bit で表せるコード範囲に収まらなかった文字については "サロゲートペア" という仕組みを用いて 16 bit (8 bit の符号を 2 つ束ねた組) で表現されます。

# `String` と `CharSequence` と `forEach()`

`String` は `CharSequence` を継承したクラスです。  
そして、`CharSequence` は拡張関数によりあたかも `Iterable` を継承しているかのような関数群が提供されています。

つまり、`forEach()` や `map()` などが使えるわけですが、安易に使うと罠にハマるかもしれません。

`CharSequence#forEach()` の引数 (ラムダ式の `it` の型) は `Char` になります。これ自体は "`Char` の sequence" と言っている以上自然なことかと思います。  
しかし、`Char` とは 8 bit の符号を格納するためのものなので、先述の "サロゲートペア" を含む文字列で `forEach()` を使った場合にはペアが分解されて格納され、文字化けが発生してしまいます。

具体的には絵文字などが影響を受けやすいです。

# それでも `String` で `forEach()` したいじゃん

ここまでで、`String` で安直に `forEach()` を使うと文字化けが発生する可能性がある、という問題を紹介しました。  
ですが、やはり `String` でも `forEach()` なり `map()` なり Collection 操作をしたい場面はあるかと思います。

そこで便利なメソッドが `String#codePoints()` です。  
このメソッドは名前の通り `String` を code point (`Int`) のシーケンスに変換してくれます。

返り値は `IntStream` なのですが、Kotlin は `IntStream` を `List<Int>` に変換する `IntStream#toList()` を提供してくれているのでそちらを利用すると楽ができるかもしれません。

なお、code point とは 8 bit/16 bit で表されている単一の文字を `Int` の値として表現したもので、それによりサロゲートペアを利用した文字も 1 つの `Int` として表現してくれます。

ちなみに `List<Int>` になったは良いが文字列に戻すときに困るじゃん…？という話ですが、僕は以下のようにして解決しています。

```kotlin
val originalString = "hogefuga🙈🙉🙊"

originalString.codePoints()
    .toList()
    .map { codePoint -> String(intArrayOf(codePoint), 0, 1) + "!" }
    .joinToString("")
```

\> `h!o!g!e!f!u!g!a!🙈!🙉!🙊!`

# まとめ

今回は `String` で iterate したいときには `String#codePoints()` を使うのが便利っぽいよ、ということを紹介しました。  
少しでも皆さんの参考に慣れれば嬉しいです。

~~本来なら `CharSequence` を iterate する時に `Char` のリストとして扱われるのがおかしい気もしますが…~~
