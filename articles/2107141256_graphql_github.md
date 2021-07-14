---
title: "GraphQLでGitHubのリポジトリ一覧を取得する"
emoji: "⚡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["graphql", "github", "csharp", "python"]
published: true
---

# はじめに

仕事でGraphQLを使う機会があったので、そのときに作成したサンプルプログラムを
紹介する記事になります。

今回は、GitHubのGraphQL APIを利用して自分のリポジトリ一覧を取得してみます。

# GraphQL vs RESTful

GraphQLはWeb APIの一種ですが、他の代表的なAPIとしてRESTful API（以下、REST）があります。
GraphQLとRESTの違いは、他の記事でもよく紹介されているので、以下のような記事を参考にされるとよいです。

https://qiita.com/NagaokaKenichi/items/a4991eee26e2f988c6ec

自分の理解のためにも、違いを簡単に説明します。

単一アプリケーションのRESTとGraphQLについて考えてみましょう。
RESTは情報取得に必要なエンドポイントがいくつかあり、その中から適切なAPIを選択して、
利用することができます。このとき、レスポンスとして返ってくる情報量は常に一定です。

対して、GraphQLはエンドポイントが一つしかありません。
リクエストを投げる際に指定するクエリによって、レスポンスとして返ってくる情報量も変化するような仕組みになっています。

GraphQLの利点は、

1. 自分にとって必要なデータだけを取得できる（そのようにクエリを記述できる）
2. 必要な情報を取得するのに何回もAPIを叩く必要がない

といったことが、挙げられます。

また、GraphQLは様々なプログラム言語をサポートしています。
この記事ではC#とPythonを使っていきます。

# 準備

## アクセストークンの取得

プログラムを書く前に、まずはGitHubのアクセストークンを取得しておきます。
GraphQL APIを叩く際の認証キーとして必要になります。

以下のURLにアクセスして、新しいトークンを生成しましょう。

https://github.com/settings/tokens

「Generate new token」ボタンを押すと、スコープを選択する画面が表示されます。
これは生成するアクセストークンにどこまでの権限を持たせるかの設定になります。

今回はリポジトリへのアクセス権限さえあればよいので「repo」にチェックを入れて、トークンに好きな名前をつけます。
あとは、「Generate token」ボタンを押せば、新しいトークンが生成されます。
生成されたトークン文字列はコピーしておきましょう。

## 環境変数への追加

生成されたトークン文字列はローカルのPCに保存しておく必要がありますが、
セキュリティの観点からメモ帳やプログラムのコード上に直接書くのは良くないので、PCの環境変数に追加しておきます。

ユーザ環境変数に「GitHubKey」変数を追加し、値は先ほどコピーした文字列を入れます。

![](https://storage.googleapis.com/zenn-user-upload/7aea05ad07f771f52139436e.jpg)

Windowsで環境変数の編集をする際は、「Rapid Environment Editor」のようなツールを使うと編集が楽になりますよ！

https://www.rapidee.com/ja/about

## クエリの確認

GitHubからリポジトリを取得するためには、どのようなクエリを記述すればよいか、予め確認しておきます。
GitHubのGraphQL APIは以下のサイトから利用することができます。

https://docs.github.com/en/graphql/overview/explorer

クエリを書いて、実行ボタンを押すと、レスポンスが横に出力されます。
![](https://storage.googleapis.com/zenn-user-upload/ab6268ed8599aeb7d514e104.jpg)

この記事では以下のクエリを使って実装を行っていきます。

``` graphql
query { 
  user(login: "tech-kind") {
      name
      url
      repositories(last: 20) {
          totalCount
          nodes {
              name
              description
              createdAt
              updatedAt
              url
          }
      }
  }
}
```

loginの後の名前（上で`tech-kind`となっている箇所）を変更することで、指定した名前に対応した情報が取得できます。
上のクエリでは、ユーザ名、ユーザのGitHub URLと総リポジトリ数に加えて、取得したリポジトリごとに以下の情報が取得できます。

1. リポジトリ名
2. リポジトリの説明
3. 作成日時
4. 更新日時
5. リポジトリのURL

# GraphQLクライアントの実装

作成したプログラムはGitHubでも公開しているので、合わせてご確認ください。

https://github.com/tech-kind/github_repo_collection

## C#の場合

### パッケージのインストール

.Net Core 3.1で作成しています。
GraphQLを使うにあたり、以下のパッケージをインストールします。

* GraphQL.Client（v3.2.4）
* GraphQL.Client.Serializer.Newtonsoft（v3.2.4）

Visual StudioでPackage Managerを使う場合には、以下のコマンドでインストールできます。

``` bash
PM> Install-Package GraphQL.Client -Version 3.2.4
PM> Install-Package GraphQL.Client.Serializer.Newtonsoft -Version 3.2.4
```

### レスポンス格納用のクラス定義

まずは、GraphQLから返ってくるレスポンスの情報を受け取るためのクラスを用意しておきます。
クラスはクエリの階層構造に従って、それぞれの階層ごとにクラスを定義をします。

:::details レスポンス格納用クラス群

``` csharp
public class ResponseType
{
    public UserType User { get; set; }
}

public class UserType
{
    /// <summary>
    /// ユーザー名
    /// </summary>
    public string Name { get; set; }

    /// <summary>
    /// ユーザーのGitHub URL
    /// </summary>
    public string Url { get; set; }

    public RepositoryType Repositories { get; set; }
}

public class RepositoryType
{
    /// <summary>
    /// 取得したリポジトリの総数
    /// </summary>
    public string TotalCount { get; set; }

    /// <summary>
    /// リポジトリの情報リスト
    /// </summary>
    public List<NodeType> Nodes { get; set; }
}

public class NodeType
{
    /// <summary>
    /// リポジトリ名
    /// </summary>
    public string Name { get; set; }

    /// <summary>
    /// リポジトリの説明
    /// </summary>
    public string Description { get; set; }

    /// <summary>
    /// 作成日時
    /// </summary>
    public string CreatedAt { get; set; }

    /// <summary>
    /// 更新日時
    /// </summary>
    public string UpdatedAt { get; set; }

    /// <summary>
    /// リポジトリURL
    /// </summary>
    public string Url { get; set; }
}
```

:::

各リポジトリの情報を格納する`NodeType`クラスは`List`型で実体を保持し、複数のリポジトリの結果を受け取れるようにしています。

### GraphQLの実行

結果をクラスで受け取れるようにできたら、実際に実行していきます。
まずは、GitHubのアクセストークンを環境変数から読み込みます。

``` csharp
// アクセストークンは環境変数から読み込む
var key = Environment.GetEnvironmentVariable("GitHubKey", EnvironmentVariableTarget.User);
```

環境変数の読み込みは`GetEnvironmentVariable`関数を使用します。
今回は、ユーザ環境変数の"GitHubKey"という変数名に含まれている値を取得します。

次に、エンドポイントの設定し、リクエストのヘッダー部に取得したアクセストークンを追加します。

``` csharp
// GitHubのGraphQLのエンドポイントの指定
var graphQLClient = new GraphQLHttpClient("https://api.github.com/graphql", new NewtonsoftJsonSerializer());
// アクセストークンをリクエストのヘッダに追加
graphQLClient.HttpClient.DefaultRequestHeaders.Add("Authorization", $"bearer {key}");
```

GitHubのGraphQLのエンドポイントは`"https://api.github.com/graphql"`になります。

最後に、クエリを記述しGraphQLを実行します。

``` csharp
// クエリに取得したい情報を記述する
var repositoriesRequest = new GraphQLRequest
{
    Query = @"
    query { 
        user(login: ""tech-kind"") {
            name
            url
            repositories(last: 20) {
                totalCount
                nodes {
                    name
                    description
                    createdAt
                    updatedAt
                    url
                }
            }
        }
    }"
};

var graphQLResponse = await graphQLClient.SendQueryAsync<ResponseType>(repositoriesRequest);
```

`SendQueryAsync`関数でGraphQLを実行できます。
型引数で`ResponseType`を指定していますが、ここで指定した型で関数の戻り値を受け取れます。
なので、戻り値で受け取った`graphQLResponse`変数の中にレスポンスの結果が入っており、簡単に結果を確認することができます。

結果をコンソールに出力してみると、しっかりリポジトリの情報が受け取れているのが確認できます。

![](https://storage.googleapis.com/zenn-user-upload/2c639b1f0b4d8c37ae216574.jpg)

## Pythonの場合

### パッケージのインストール

Pythonの場合は、`requests`モジュールが必要となります。
`requests`モジュールが存在しない場合は、以下のコマンドでインストールしておきます。

``` bash
$ python -m pip install requests
```

### モジュールのラッパークラスの定義

Pythonでは、まず上記の`requests`モジュールをラッパーするようなクラスを定義しておきます。

::: details requestsモジュールのラッパークラス

``` python
class BaseRequest(object):
  def __init__(self, url, access_token):
      self._url = url
      self._headers = {'Authorization': 'Bearer {}'.format(access_token)}

      self._rdata = requests.Response()

  def get(self):
      self._rdata = requests.get(self._url, headers=self._headers)

  def post(self, query):
      self._rdata = requests.post(self._url, json=query, headers=self._headers)

  def check_status(self):
      if self._rdata.status_code == 200:
          return True
      else:
          return False

  def get_json_data(self):
      return self._rdata.json()
```

:::

GraphQLを実行する際は、`post`関数を使用します。
引数で受け取ったクエリに基づき、レスポンスが返ってきます。

レスポンスの結果を受け取る際は、`get_json_data`関数を使用します。
先ほど、`post`関数実行時に受け取ったレスポンスは`self._rdata`というメンバ変数に格納されていますが、
`get_json_data`関数ではこのレスポンス情報をjson形式に変換したうえで結果を返しています。
Pythonの型としては、`dict`として扱うことができます。

### GraphQLの実行

まず、環境変数からアクセストークンを読み込む際は、`os`モジュールの`getenv`関数を使用します。

``` python
access_token: str = getenv("GitHubKey")
```

次に、エンドポイントを設定し、先ほど定義したラッパークラス`BaseRequest`クラスのインスタンスを作成します。

``` python
url: str = "https://api.github.com/graphql"
graphql_client = BaseRequest(url, access_token)
```

最後に、クエリを記述しGraphQLを実行します。

``` python
query = { 'query' : """
  query { 
    user(login: "tech-kind") {
      name
      url
      repositories(last: 20) {
        totalCount
        nodes {
          name
          description
          createdAt
          updatedAt
          url
        }
      }
    }
  }
  """
}

graphql_client.post(query)
```

結果をコンソールに出力してみましょう。C#と同様の結果が出力されるのが確認できます。
やりました！

![](https://storage.googleapis.com/zenn-user-upload/1c492db8e533706d6074ba5d.jpg)

# 最後に

GraphQL、扱いやすくていいですね！
RESTだと、API叩いた結果を使って、また別のAPIを叩くみたいなことがありましたが、
そういった煩わしさから解放されるだけでも魅力があります。
実際の仕事では、RESTを使うことのほうが多いですが、今後はGraphQLも使っていこうかな。
