---
title: "Androidアプリ開発者の Unity + ARCore 入門"
date: 2019-01-10T13:47:52+09:00
draft: false
---
今回は、普段は Android アプリを主に開発している筆者が Unity を使って ARCore を弄ってみようとした奮闘記です。  
なお、すべての情報は記事執筆時点のものであり、閲覧時には参考とならない場合がありますので注意してください。  
また、以下の情報は全て macOS を前提としています。

# Unity Hub
Unity を使うには、素の Unity をインストールする方法と、 Unity Hub という公式の Unity 専用パッケージマネージャのようなものを介して Unity をインストールする方法の2種類が代表的です。  
今回は、特に大きな理由はありませんが、後々の新バージョンへの追随が楽そうだということで Unity Hub を使用します。

## インストール
[こちら](https://unity3d.com/jp/get-unity/download)より、「Unity Hub をダウンロード」というボタンをクリックします。  
インストーラがダウンロードされるので、指示に従ってインストールしてください。

![ダウンロードページ](/images/ss_download_unity-hub.png)

## Unity 本体のインストール
Unity Hub を起動すると、以下の画面が表示されます。

![起動画面](/images/ss_unity-hub_startup.png)

「Download」ボタンを押し、特に理由がない場合は最新の安定版をインストールします。

![リリース一覧画面](/images/ss_list_release-unity.png)

このとき、 ARCore を利用するために必要なので、オプションで「**Android Build Support**」を忘れずに選択してください（本体インストール後に追加インストールもできますが、タイミングによっては多少ややこしいことになります）。

![オプション一覧画面](/images/ss_list_option-unity.png)

これで Unity のインストールは終了です。

## ARCore プラグイン
Unity のインストールには時間がかかるので、その間に [Unity 用 ARCore プラグイン](https://github.com/google-ar/arcore-unity-sdk/releases)をダウンロードしておきましょう。  
なお、こちらのプラグインは Unity にインストールして使う、という類のものではなく、必要になったときに都度ロードして利用するのでわかりやすい場所への移動をおすすめします。

# プロジェクト作成
Unity 本体のインストールが終わっていることを確認し、新規プロジェクトを作ります。

![新規ボタン](/images/ss_new-project-unity.png)

プロジェクト名を適当につけ、テンプレートとして3Dを選択、親ディレクトリに適当な場所を選択し、「Create Project」ボタンを押します。

![プロジェクト新規作成画面](/images/ss_unity_new_project_setting.png)

![開かれた新規プロジェクト](/images/ss_unity_new_project_opened.png)

成功すれば上のような画面が表示されます。

## ARCore の設定
### ビルドタイプの変更
ARCore を使った Android アプリを生成するために、ビルドタイプに Android を指定する必要があります。  
[File] -> [Build Settings…] をクリックし、[Android] を選択して [Switch Platform] ボタンを押します。

![ビルド設定画面を開く](/images/ss_unity_open_build_settings.png)

![ビルドタイプをAndroidに切り替え](/images/ss_unity_build_settings_android.png)

### プレイヤー設定の変更
ここで、Android ビルド関連の各種設定も弄っておきます。  
開いている Build Settings 画面内の [Player Settings…] ボタンを押します。

![プレイヤー設定画面を開く](/images/ss_unity_open_player_settings.png)

Build Settings 画面はもう必要ないので閉じます。  
メインウィンドウの [Inspector] カラムにプレイヤー設定が表示されているはずです。

![プレイヤー設定画面](/images/ss_unity_player_settings.png)

変更すべき点は

- Other Settings
  - Rendering
    - Multithreaded Rendering -> Off
    - Identification
      - Package Name -> お好きなものに
      - Version -> お好きなものに
      - Minimum API Level -> 7.0 / 24 以上に
      - Target API Level -> 現時点の最新に
- XR Settings
    - ARCore Supported -> On

といったところでしょうか。

![その他設定の変更](/images/ss_unity_player_settings_other.png)

![XR設定の変更](/images/ss_unity_player_settings_xr.png)

### パスの設定
Android ビルドをするにあたって、各種パス設定も必要なので行います。  
なお、 Unity にて ARCore を扱う際には NDK もあったほうが良いようですので、そちらも合わせてダウンロードしておきます（特定バージョンのNDKを求められるので既にNDKを持っている方も確認してください）。

[Unity] -> [Preferences…] より Unity の設定画面を開きます。

![Unity の設定画面を開く](/images/ss_unity_open_preference.png)

[External Tools] タブを選択し、各種パスを設定します。
NDK については [Download] ボタンを押せば対応しているバージョンのものが手に入るので楽です。

![Android の各種パスを設定](/images/ss_unity_change_android_path.png)

Android SDK のパスがわからない場合は Android Studio を起動して設定を確認すればわかるかと思います。

## ARCore の導入
いよいよプロジェクトに ARCore を導入します。  
[先ほどダウンロードしておいた](#arcore-プラグイン)プラグインを現在のプロジェクトに読み込ませます。

[Assets] -> [Import Package] -> [Custom Package…] より、ダウンロードしておいた ARCore のパッケージを選択します。

![ARCore のパッケージをインポート](/images/ss_unity_import_custom_package.png)

パッケージ内からどれをインポートするか聞かれますが、とりあえず全て選択し、 [Import] ボタンを押します。

![ARCore のパッケージ構成](/images/ss_unity_import_arcore.png)

ここまでで一通りの前準備は終わりました。

# サンプルを動かす
ARCore のインポートが終わるとサンプルもロードされるので、そちらを動かしてみましょう。

メインウィンドウの [Project] カラム内を  
[Assets] -> [GoogleARCore] -> [Examples] -> [HelloAR] -> [Scenes]  
とたどります。  
HelloARシーンが見つかるので、ダブルクリックして展開します。

![Projectカラム](/images/ss_unity_helloar_column_project.png)

**おめでとうございます！**  
これで、Android 端末を接続した上で Play ボタン(▶) を押すと、ドロイド君(Andy) が平面に置き放題な AR アプリデモが実行できるはずです。

# 実際の AR アプリ開発
実際に独自の AR アプリを開発していきたい、となった時、イチから自前で ARCore の実装をする事もできるとは思います。  
しかし、サンプルである HelloAR は ARCore の最低限の実装を備えているので、これを改変していくのが近道であり妥当ではないかと思います。  
その上で、Unity 開発固有のやり方を覚えていくことになると思いますが、

- Prefab の扱い方
- Game Object と Script の紐付き方

を押さえると、比較的すんなり開発が進むのではないかと思います。  
それでは、良き ARCore ライフを！
