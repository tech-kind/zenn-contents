---
title: "C#でROSっぽい書き方をしてみる"
emoji: "🌠"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["csharp", "ros", "ros2"]
published: true
---

# はじめに

ROS（ROS2も含む）って魅力的なプラットフォームですよね。
オープンソースでありながら、ロボット開発において重要な役割を果たしており、
自動運転システムなどでも採用されていたりします。

ROSには、
* 処理をノードとして分割する
* ノード同士はトピックを介してデータのやり取りを行う

といった特徴があります。

これらの特徴は、
* 再利用性・生産性の向上
* 障害分離（障害発生時に特定のプロセスの切り離しや再起動がしやすい）

といったメリットをもたらし、これらのメリットはロボット開発のみならず、
あらゆるシステム開発において享受されるべきものだと考えられます。

そこで今回は、C#でROSっぽいコードを書いてみたいと思います。
（あくまでも、ROSっぽいコードです。）

# 実装

## 全体像

今回扱うソースコードはGitHubに挙げていますので、合わせてご確認ください。

https://github.com/tech-kind/interprocess_sample

開発はVisual Studio 2022を使用し、バージョンは.Net6になります。
3つのプロジェクトが入っており、
1. InterProcessProvider（ROSっぽい書き方をするためのクラスの定義など）
2. SamplePublisher（トピックをPublishする側のサンプルアプリ）
3. SampleSubscription（トピックをSubscribeする側のサンプルアプリ）

となっております。

また、開発するにあたりMessagePipeを利用させていただきました。

https://github.com/Cysharp/MessagePipe

## クラス定義

まず、NodeやPublisher、Subscriptionの定義をしていきます。

それぞれの定義は以下をご覧ください。

:::details Nodeクラス

```csharp:InterProcessProvider/Node.cs
public abstract class Node : IDisposable
{
    /// <summary>
    /// Create a new instance.
    /// </summary>
    /// <param name="serviceProvider"></param>
    public Node(ServiceProvider serviceProvider)
    {
        _serviceProvider = serviceProvider;
    }

    private readonly ServiceProvider _serviceProvider;

    #region dispose

    public void Dispose()
    {

    }

    #endregion

    #region message
    /// <summary>
    /// Creates a message publisher for the specified type.
    /// </summary>
    /// <typeparam name="TMessage"></typeparam>
    /// <returns></returns>
    protected Publisher<TMessage> CreatePublisher<TMessage>(string topicName)
        where TMessage : notnull
    {
        var pub = _serviceProvider.GetRequiredService<IDistributedPublisher<string, TMessage>>();
        return new Publisher<TMessage>(topicName, pub);
    }

    /// <summary>
    /// Creates a message subscriber for the specified type.
    /// </summary>
    /// <typeparam name="TMessage"></typeparam>
    /// <returns></returns>
    protected Subscription<TMessage> CreateSubscription<TMessage>(string topicName)
        where TMessage : notnull
    {
        var sub = _serviceProvider.GetRequiredService<IDistributedSubscriber<string, TMessage>>();
        return new Subscription<TMessage>(topicName, sub);
    }
    #endregion
}
```

:::

:::details Publsiherクラス

```csharp:InterProcessProvider/Publisher.cs
public class Publisher<TMessage>
{
    private readonly string _topicName = "";
    private readonly IDistributedPublisher<string, TMessage> _publisher;

    public Publisher(string topicName, IDistributedPublisher<string, TMessage> publisher)
    {
        _topicName = topicName;
        _publisher = publisher;
    }

    public void Publish(TMessage message)
    {
        _publisher.PublishAsync(_topicName, message);
    }

}
```

:::

:::details Subscriptionクラス

```csharp:InterProcessProvider/Subscription.cs
public class Subscription<TMessage>
{
    private readonly string _topicName = "";
    private readonly IDistributedSubscriber<string, TMessage> _subscription;

    public Subscription(string topicName, IDistributedSubscriber<string, TMessage> subscription)
    {
        _topicName = topicName;
        _subscription = subscription;
    }

    public ValueTask<IAsyncDisposable> Subscribe(IMessageHandler<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters)
    {
        return _subscription.SubscribeAsync(_topicName, handler, filters);
    }

    public ValueTask<IAsyncDisposable> Subscribe(Action<TMessage> handler, params MessageHandlerFilter<TMessage>[] filters)
    {
        return _subscription.SubscribeAsync(_topicName, handler, filters);
    }

    public ValueTask<IAsyncDisposable> Subscribe(Action<TMessage> handler, Func<TMessage, bool> predicate, params MessageHandlerFilter<TMessage>[] filters)
    {
        return _subscription.SubscribeAsync(_topicName, handler, predicate, filters);
    }
}
```

:::

プロセス間で通信を行いたいので、MessagePipeの`IDistributedPublisher`と`IDistributedSubscriber`を使用しました。
外側からトピック名を指定して通信を行いたかったので、それぞれをラッパーするようなクラスとして`Publisher`と`Subscription`を定義しています。
`Node`は抽象クラスとして定義し、実際に個々の処理を行うノードを作成する際にこのクラスを継承して作成します。
`Node`を継承することで、継承先のクラス内で`Publisher`や`Subscription`を生成することができます。

## Pub/Subのサンプル

実際に上のクラスを利用して、Pub/Subのサンプルを行ってみたいと思います。

まずは、`Publisher`を利用したノードを作ってみます。

```csharp:SamplePublisher/SamplePublisherNode.cs
public class SamplePublisherNode : Node
{
    private readonly Publisher<int> _publisher;
    private int _count = 0;

    public SamplePublisherNode(ServiceProvider serviceProvider)
        : base(serviceProvider)
    {
        _publisher = CreatePublisher<int>("/api/test");

        var timer = new System.Timers.Timer(100);
        timer.Elapsed += OnTimer;

        // タイマーを開始する
        timer.Start();
    }

    private void OnTimer(object? sender, EventArgs e)
    {
        _count++;
        Console.WriteLine($"[publish] count={_count}");
        _publisher.Publish(_count);
    }
}
```

`Node`を継承して、`Publisher`を生成できるようにしておきます。
`Publisher`の生成は、`CreatePublisher`関数を使用します。
上記の例では、トピック名を`"/api/test"`、送信するメッセージの型を`int`にしています。
100ミリ秒ごとに実行されるタイマーを用意し、カウンターを1ずつ増やしながら送信します。

次に、`Subscription`を利用したノードを作ってみます。

```csharp:SamplePublisher/SamplePublisherNode.cs
public class SampleSubscriptionNode : Node
{
    private readonly Subscription<int> _subscription;

    public SampleSubscriptionNode(ServiceProvider serviceProvider)
        : base(serviceProvider)
    {
        _subscription = CreateSubscription<int>("/api/test");
        _subscription.Subscribe(CountCallback);
    }

    private void CountCallback(int count)
    {
        Console.WriteLine($"[subscribe] count={count}");
    }
}
```

こちらも`Node`を継承することで、`Subscription`を生成できるようにしておきます。
`Subscription`の生成は、`CreateSubscription`関数を使用します。
`Publisher`側とトピック名を合わせます。ここでトピック名を合わせておくことで、同じトピック名同士が通信しあうことになります。
受信したカウンターをコンソールに出力するだけの簡単なノードです。

最後に、Pub/Subのサンプルそれぞれの`Program.cs`を見ておきましょう。

```csharp:SamplePublisher/Program.cs
var provider = InterprocessProvider.init("127.0.0.1", 3125);
var publisherNode = new SamplePublisherNode(provider);
InterprocessProvider.Spin();
```

```csharp:SampleSubscription/Program.cs
var provider = InterprocessProvider.init("127.0.0.1", 3125);
var subscriptionNode = new SampleSubscriptionNode(provider);
InterprocessProvider.Spin();
```

まず、`init`関数でIPアドレスとポート番号を指定します。
その後、ノードを生成した後に、プログラムが終了しないように`Spin`関数を実行しておきます。

以上です。
たった、3行！素晴らしい！
そして、かなりROSっぽいのではないでしょうか。

動作確認してみましょう。
ビルドして、`SamplePublsiher.exe`と`SampleSubscription.exe`の両方を立ち上げてコンソールを確認してみます。

問題なく通信できていることがわかります。

![](/images/2202091147_internalprocess/exe_result.png)

# まとめ

この仕組みを利用すれば、アプリ間でデータを共有するのも容易になりますし、
複数のアプリをまたいでログを収集するといったこともできそうです。
いろんなことに応用できそうでワクワクしてきました。

こうしたほうが良いのではとかありましたら教えてください！
最後まで読んでいただきありがとうございました。
