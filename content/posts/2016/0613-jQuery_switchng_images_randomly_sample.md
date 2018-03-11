---
title: "[jQuery] 60秒ごとにランダムに画像を切り替えるサンプル"
date: 2016-06-13T23:24:00+09:00
draft: true
tags: ["Web", "JavaScript"]
---
```js
$(function(){
    var isFirstLoadImg = true;
    var bgImg = ["img/hoge.jpg", "img/huga.jpg", "img/piyo.jpg"];
    var index = Math.floor(Math.random() * bgImg.length);

    loadImg(index);
    function loadImg(i) {
        var image = new Image();

        image.onload = function(){
            if (isFirstLoadImg) {
                isFirstLoadImg = false;
                $("#bg").css("background-image","url(\"" + bgImg[i] + "\")");
                $("#bg").fadeIn(1000);
                setIndex();
                loadImg(index);
            }
        };
        image.src = bg[i];
    }

    function setIndex() {
        var index_current = index;
        while (index == index_current) {
            index = Math.floor(Math.random() * bgImg.length);
        }
    }

    function changeBg(i) {
        $("#bg").fadeOut(1000, function(){
            $("#bg").css("background-image","url(\"" + bgImg[i] + "\")");
        });
        $("#bg").fadeIn(1000);
    }

    setInterval(function(){
        changeBg(index);
        setIndex();
        loadImg(index);
    }, 60000);
});
```
