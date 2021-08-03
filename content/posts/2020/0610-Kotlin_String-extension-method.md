---
title: "Kotlin の String extension method の罠"
date: 2020-06-10T03:04:00+09:00
draft: false
tags: ["Kotlin"]
---

Kotlin の (やりすぎ説はあるが) 便利なところといえば、Collections や String の拡張関数が真っ先に浮かぶ筆者ですが、先日いつも通り便利に拡張関数を利用しながら Android 開発をしていたときに罠 (？) を踏み抜いたので記録に残しておこうかと思います。

# String/CharSequence の拡張関数

結論から言うと、僕は `String` や `CharSequence` の拡張関数が自身を加工した結果として自身と同じ型の値を返す場合、その型は継承状態を保持していると思いこんでいましたが、そんなことはなかったという話です。

何を言っているのかというと、  
例えば `Spanned` は `CharSequence` を継承していて、 `CharSequence.trimEnd()` は `CharSequence` を返しますが、その返却された値を必ずしも `Spanned` にキャストできるとは限らない、ということです。

"必ずしも" ということはできる場合とできない場合があるのですが、そのサンプルがこちらです。

```kotlin
// OK
SpannableStringBuilder("abc\n\n\n").trimEnd() as Spanned
SpannableStringBuilder("abc").trimEnd() as Spanned
```

```kotlin
// NG
SpannableStringBuilder("\n\n\n").trimEnd() as Spanned
SpannableStringBuilder("").trimEnd() as Spanned
```

なぜこのような違いが生まれるのかというと、`trimEnd()` の内部では **呼び出し元の文字列がすべて空白文字である** 場合と **呼び出し元の文字列が空文字列である** 場合には空文字の `String` インスタンスを新規生成して返却し、そうでない場合には元の文字列に対して `subSequence()` を呼び出した結果を返却するからです。

# JetBrains の見解

[YouTrack にてこの問題を報告してみた](https://youtrack.jetbrains.com/issue/KT-39370) のですが、「[それはそういう仕様だよ](https://youtrack.jetbrains.com/issue/KT-39370#focus=streamItem-27-4179513.0-0)」と言われてしまいました。

当たり前といえば当たり前なのですが、先述の通りなまじうっかりキャストが成功するケースもあるので、もしも今までキャストができるものだと思いこんでいた僕のような方がいれば、それはたまたまであるということを留意いただければと思います。

# おわりに

~~別に `subSequence(0,0)` で返せば良くない？~~

それでは！
