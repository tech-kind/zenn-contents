---
title: "GraphQLでGitHubのリポジトリ一覧を取得する"
emoji: "🍀"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "github", "csharp", "python"]
published: false
---

# はじめに

仕事でGraphQLを使う機会があったので、そのときに作成したサンプルプログラムを
紹介しようという記事になります。

今回は、GitHubのGraphQL APIを利用して自分のリポジトリ一覧を取得してみます。

# GraphQL vs REST

GraphQLはWeb APIの一種ですが、他の代表的なAPIとしてRESTful API（以下、REST）があります。
GraphQLとRESTの違いは、他の記事でもよく紹介されているので、以下のような記事を参考にされるとよいと思います。

https://qiita.com/NagaokaKenichi/items/a4991eee26e2f988c6ec

ここでも、違いを簡単に説明します。

単一アプリケーションのRESTとGraphQLについて考えてみましょう。
RESTは情報取得に必要なエンドポイントがいくつかあり、その中から適切なAPIを選択して、
利用することができます。このとき、レスポンスとして返ってくる情報量は常に一定です。

対して、GraphQLはエンドポイントが一つしかありません。
リクエストを投げる際に指定するクエリによって、レスポンスとして返ってくる情報量も変化するような仕組みになっています。

GraphQLの利点は、

1. 自分にとって必要なデータだけを取得できる（そのようにクエリを記述できる）
2. 必要な情報を取得するのに何回もAPIを叩く必要がない

といったことが、挙げられます。

GraphQLは様々なプログラム言語をサポートしています。
今回はC#とPythonを使ってみます。

# 準備

## アクセストークンの取得

プログラムを書く前に、まずはGitHubのアクセストークンを取得しておきます。
GraphQL APIを叩く際の認証キーとして必要になります。

以下のURLにアクセスすると、新しいトークンを生成できます。

https://github.com/settings/tokens

「Generate new token」ボタンを押すと、スコープを選択する画面が表示されます。
これは生成するアクセストークンにどこまでの権限を持たせるかの設定になります。

今回はリポジトリへのアクセス権限さえあればよいので「repo」にチェックを入れて、トークンに好きな名前をつけておきます。
あとは、「Generate token」ボタンを押せば、新しいトークンが生成されます。
生成されたトークン文字列はコピーしておきましょう。

## 環境変数への追加

生成されたトークン文字列はローカルのPCに保存しておく必要がありますが、
セキュリティの観点からメモ帳やプログラムのコード上に直接書くのは良くないので、今回はPCの環境変数に追加しておきます。

ユーザ環境変数に「GitHubKey」変数を追加し、値は先ほどコピーした文字列を入れておきます。

![](https://storage.googleapis.com/zenn-user-upload/7aea05ad07f771f52139436e.jpg)

Windowsで環境変数の編集をする際は、「Rapid Environment Editor」のようなツールを使うと編集が楽になります。

https://www.rapidee.com/ja/about

# GraphQLクライアントの実装

## C#の場合

## Pythonの場合