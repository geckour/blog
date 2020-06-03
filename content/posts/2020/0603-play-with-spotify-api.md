---
title: "Spotify API と戯れる"
date: 2020-06-03T23:55:00+09:00
draft: false
tags: ["Android", "API"]
---

ドキュメントもあって一見使いやすそうな [Spotify Web API](https://developer.spotify.com/documentation/web-api/) 。  
ですが、使ってみると案外ハマりどころがあったので簡単にまとめてみようと思います。

# Spotify Web API とは

Spotify が提供している、rate limit 制限内なら色んな情報をくれちゃう便利な API。

例えば、[Search API](https://developer.spotify.com/documentation/web-api/reference/search/search/) ではその名の通り Spotify が持つ巨大なライブラリを検索して結果を得られるのですが、そのエンティティは

<details>

```json
{
  "tracks": {
    "href": "https://api.spotify.com/v1/search?query=Perfume+TOKYO+GIRL&type=track&offset=0&limit=1",
    "items": [
      {
        "album": {
          "album_type": "album",
          "artists": [
            {
              "external_urls": {
                "spotify": "https://open.spotify.com/artist/2XMxWKPKCxoLkSdpCViCnr"
              },
              "href": "https://api.spotify.com/v1/artists/2XMxWKPKCxoLkSdpCViCnr",
              "id": "2XMxWKPKCxoLkSdpCViCnr",
              "name": "Perfume",
              "type": "artist",
              "uri": "spotify:artist:2XMxWKPKCxoLkSdpCViCnr"
            }
          ],
          "available_markets": ["JP"],
          "external_urls": {
            "spotify": "https://open.spotify.com/album/0e7AS0Cgn03lXq2orBHQG0"
          },
          "href": "https://api.spotify.com/v1/albums/0e7AS0Cgn03lXq2orBHQG0",
          "id": "0e7AS0Cgn03lXq2orBHQG0",
          "images": [
            {
              "height": 640,
              "url": "https://i.scdn.co/image/ab67616d0000b27341d029191747f109998f8023",
              "width": 640
            },
            {
              "height": 300,
              "url": "https://i.scdn.co/image/ab67616d00001e0241d029191747f109998f8023",
              "width": 300
            },
            {
              "height": 64,
              "url": "https://i.scdn.co/image/ab67616d0000485141d029191747f109998f8023",
              "width": 64
            }
          ],
          "name": "Future Pop",
          "release_date": "2018-08-15",
          "release_date_precision": "day",
          "total_tracks": 12,
          "type": "album",
          "uri": "spotify:album:0e7AS0Cgn03lXq2orBHQG0"
        },
        "artists": [
          {
            "external_urls": {
              "spotify": "https://open.spotify.com/artist/2XMxWKPKCxoLkSdpCViCnr"
            },
            "href": "https://api.spotify.com/v1/artists/2XMxWKPKCxoLkSdpCViCnr",
            "id": "2XMxWKPKCxoLkSdpCViCnr",
            "name": "Perfume",
            "type": "artist",
            "uri": "spotify:artist:2XMxWKPKCxoLkSdpCViCnr"
          }
        ],
        "available_markets": ["JP"],
        "disc_number": 1,
        "duration_ms": 267386,
        "explicit": false,
        "external_ids": {
          "isrc": "JPPO01803313"
        },
        "external_urls": {
          "spotify": "https://open.spotify.com/track/4GeM1lg9SDZXyBlPBlE62L"
        },
        "href": "https://api.spotify.com/v1/tracks/4GeM1lg9SDZXyBlPBlE62L",
        "id": "4GeM1lg9SDZXyBlPBlE62L",
        "is_local": false,
        "name": "TOKYO GIRL - Remastered",
        "popularity": 46,
        "preview_url": null,
        "track_number": 4,
        "type": "track",
        "uri": "spotify:track:4GeM1lg9SDZXyBlPBlE62L"
      }
    ],
    "limit": 1,
    "next": "https://api.spotify.com/v1/search?query=Perfume+TOKYO+GIRL&type=track&offset=1&limit=1",
    "offset": 0,
    "previous": null,
    "total": 18
  }
}
```

</details>

このように様々なデータが詰まっています。
さらに、そうして得た楽曲の [Spotify ID](https://developer.spotify.com/documentation/web-api/#spotify-uris-and-ids) を [Audio Features API](https://developer.spotify.com/documentation/web-api/reference/tracks/get-several-audio-features/) に投げると、これまた

<details>

```json
{
  "audio_features": [
    {
      "danceability": 0.728,
      "energy": 0.915,
      "key": 2,
      "loudness": -3.13,
      "mode": 1,
      "speechiness": 0.0339,
      "acousticness": 0.222,
      "instrumentalness": 0.153,
      "liveness": 0.0831,
      "valence": 0.659,
      "tempo": 127.97,
      "type": "audio_features",
      "id": "4GeM1lg9SDZXyBlPBlE62L",
      "uri": "spotify:track:4GeM1lg9SDZXyBlPBlE62L",
      "track_href": "https://api.spotify.com/v1/tracks/4GeM1lg9SDZXyBlPBlE62L",
      "analysis_url": "https://api.spotify.com/v1/audio-analysis/4GeM1lg9SDZXyBlPBlE62L",
      "duration_ms": 267387,
      "time_signature": 4
    }
  ]
}
```

</details>

このように詳細なデータを惜しげもなく提供してくれます。

詳しくは[公式ドキュメント](https://developer.spotify.com/documentation/web-api/)へ。

# 罠 1: Search API

とても素敵な [Spotify Web API](https://developer.spotify.com/documentation/web-api/) ですが、意外と癖が強い API でもあります。

その最たる例のひとつが [Search API](https://developer.spotify.com/documentation/web-api/reference/search/search/) の検索クエリで、日本語 (マルチバイト) を含むクエリをドキュメントの通りに投げるとうまく検索できないケースが多発します。

<sub>うまく行かないクエリの例</sub>

- `track:"恋" album:"恋" artist:"星野源"`

試行錯誤を繰り返し得た現在の結論は、**マルチバイト文字を含む場合は field filter を外す** というものです。

<sub>うまく行くクエリの例</sub>

- `"恋" "恋" "星野源"`
- `track:"TOKYO GIRL" album:"TOKYO GIRL" artist:"Perfume"`

絞り込み能力は落ちると思われますが、致し方ないかなと妥協しています。

# 罠? 2: Refresh Token

[Spotify Web API](https://developer.spotify.com/documentation/web-api/) を使うためには認証が必要となりますが、アプリケーションに対する認証フローである [Client Credentials Flow](https://developer.spotify.com/documentation/general/guides/authorization-guide/#client-credentials-flow) を使うと rate limit に容易に達してしまうため、リクエストが頻繁に発生する場合には必然的に [Authorization Code Flow](https://developer.spotify.com/documentation/general/guides/authorization-guide/#authorization-code-flow) を使うことになると思います。

その際、トークンの有効期限が切れた場合に refresh token を使って更新する必要があるのですが、refresh token は最初の access token リクエスト時か refresh token によって新しい access token を得たとき以外は `null` が返ってくるようで、デバッグ時は扱いに気をつけないと永遠に token を更新できない、という事態にハマるかもしれません。

また、ここでハマる人は少ないとは思いますが、refresh token の使い方がドキュメント上で探しづらくて個人的にハマりました ([ここに書いてあります](https://developer.spotify.com/documentation/general/guides/authorization-guide/#4-requesting-a-refreshed-access-token-spotify-returns-a-new-access-token-to-your-app)) 。

要点としては、access token をリクエストする際は `/api/token` に

- `code`
- `redirect_uri`
- `grant_type`

を投げますが、refresh token では同じく `/api/token` に

- `refresh_token`
- `grant_type`

を投げればよい、というところの様です。

# 罠? 3: いっぱい貰えるデータ

詳細なデータを惜しげもなくくれる [Spotify Web API](https://developer.spotify.com/documentation/web-api/) ですが、いかんせん色々くれすぎて本当に欲しいデータがどれなのか判断に困るケースもあったりします。

例えば、先程も例に出した [Search API](https://developer.spotify.com/documentation/web-api/reference/search/search/) のレスポンスには、

```json
{
    ...

    "external_urls": {
        "spotify": "https://open.spotify.com/track/4GeM1lg9SDZXyBlPBlE62L"
    },
    "href": "https://api.spotify.com/v1/tracks/4GeM1lg9SDZXyBlPBlE62L",
    "preview_url": null,

    ...
}
```

というフィールドがあるのですが、どちらをどう使うのが適切なのかがいまいち分かっていません。

[ドキュメント](https://developer.spotify.com/documentation/web-api/reference/object-model/#track-object-full) を見てみても、分かるようでわからない…  
結局の所は推理能力が求められるのかもしれません。

# まとめ

つらつらと書きましたが、 [罠 1](#罠-1-search-api) 以外は半ば無理やり絞り出したもので大した罠とは言えないかもしれません。

癖さえ掴めば色々な情報を手軽に手に入れられる、素敵な API 。

このような強力な API が公開されているのに、使わないのは損をしているとさえ言えるのではないでしょうか。

末筆ながら [Search API](https://developer.spotify.com/documentation/web-api/reference/search/search/) でハマる人が少しでも減ることを願っています。

それでは。
