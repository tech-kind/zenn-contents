---
title: "外付けドライブにUbuntu20.04 LTSをインストールする方法"
emoji: "💻"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["Linux", "Ubuntu", "MicrosoftSurface", "SurfaceBook2"]
published: true
---

# はじめに

最近、ROSなどの勉強で、Ubuntuを使いたい機会があったので、外付けSSDにUbuntu20.04をインストールしました。

インストールには下記Qiita記事を参考にさせていただきました。

https://qiita.com/koba-jon/items/019a3b4eac4f60ca89c9

ただ、この記事にも記載がありますが、Microsoft Surfaceでは、記事の手順通りに進めても、Ubuntuが起動しません。

では、「Microsoft SurfaceでUbuntuを起動できないのか？」というと、そういうわけではないので、解決策ややっておくとよいことをここに載せておきます。

# 1. BitLockerの無効化

まず、BitLockerの無効化です。

Ubuntuをインストールすると、Windows起動時に、毎回BitLockerの回復を求められます。

BitLockerの回復には暗号化キーの入力が必要で、これを毎回入力するのはかなり大変です。

簡単に解決するには、BitLockerを無効化してしまうことです。

Windowsでファイルエクスプローラーを起動して、デバイスとドライブから「Windowsのドライブ」を右クリックし、「BitLockerを無効にする」をクリックします。

あとは、無効になるまでしばらく待っていただくだけです。

![](https://storage.googleapis.com/zenn-user-upload/ubr8516i3xkinydf1ope6fx00kfo)

::: message
BitLockerが無効化されていると、画像のように、「BitLockerを有効にする」と表示されます。
有効化されている間は、「BitLockerを無効にする」と表示されているはずなので、文言に従って有効化/無効化してください。
:::

# 2. セキュアブートの変更

2つ目に、UEFIのセキュアブートの設定の変更になります。

この設定は、Ubuntuを起動するのに絶対必要になるので、この項目が一番大切です。

Microsoftの公式サイトでもSurface UEFIのセキュリティに関することが記載されています。

https://docs.microsoft.com/ja-jp/surface/manage-surface-uefi-settings#uefi-%E3%82%BB%E3%82%AD%E3%83%A5%E3%83%AA%E3%83%86%E3%82%A3-%E3%83%9A%E3%83%BC%E3%82%B8uefi-security-page

UEFI設定画面を表示するには、
1. Surfaceをシャットダウンし、10秒待って電源が切れるのを確認します。
2. 音量を上げボタンを長押しし、同時に電源ボタンを長押しします。
3. MicrosoftまたはSufaceロゴが画面に表示されたら、UEFI設定画面が表示されるまで音量を上げ続けます。

画面が表示されたら、「Security→Secure Boot」から「Change configuration」ボタンを押して設定を調整します。

「Change configuration」ボタンを押すと、セキュアブートの設定変更を行う画面が表示されるので、
「Microsoft & 3rd-party CA」を選択して、OKを押してください。

これで、サードパーティ製のOSをSurfaceから起動できるようになります。

# 3. Windows時刻の自動同期

最後に、時刻の自動同期になります。

SurfaceでUbuntuを起動できるようになると、Ubuntuを起動するたびにWindowsの時刻がUTC（協定世界時）に変更されてしまいます。

Ubuntu側の時間を日本時間に変更していたとしても、同様の現象になります。

これを回避するには、システム起動時にWindows Timeサービスを開始するようにしてあげる必要があります。

まずは、サービスアプリを起動し、「Windows Time」というサービスを探します。

![](https://storage.googleapis.com/zenn-user-upload/3axxlwkcmiyz7x4ebd6csnkpmfmc)

見つかったら、Windows Timeを右クリックしてプロパティを選択します。

そこで、スタートアップの種類を確認し、「手動」などになっている場合は、「自動（遅延開始）」に変更し、適用してください。

次に、タスクスケジューラアプリを起動します。

タスクスケジューラライブラリから「Microsoft >> Windows >> Time Synchronization」と手繰っていき、「時刻同期」を選択します。

![](https://storage.googleapis.com/zenn-user-upload/5190024t93ptmssjvsrw3eft4c5v)

選択したら、画面右の選択した項目から、「無効化」を選択し、時刻同期のトリガーを無効化します。

::: message
トリガーを無効化していると、画像のように、「有効化」と表示されます。
有効化されている間は、「無効化」と表示されているはずなので、文言に従って有効化/無効化してください。
:::

これで、システム起動時に、Windows Timeサービスが起動し、時刻同期がされるようになります。

# 最後に

Ubuntuをしばらく触ってみましたが、仮想環境とは違うので、動作も快適でいいですね。

いいROSの勉強ができそうです。

2つのOSを扱えるなんて何かかっこいい！笑
