---
title: "Flutter x Firebaseでメール通知機能を実装してみる"  
emoji: "📩"   
type: "tech"  
topics: ["Flutter", "Firebase", "SendGrid"] # タグ。["markdown", "rust", "aws"]のように指定する
published: true 
---
この記事は、[Flutter 大学アドベントカレンダー 2022](https://qiita.com/advent-calendar/2022/flutteruniv) 12日目の記事です。

# はじめに
こんにちは、Masakiといいます。  
先日、実務でメール通知機能を実装する機会があったため、その時の学びをまとめておこうと思います。  

とてもシンプルに実装出来るのでぜひご覧ください💡

# 今回つくるもの
こちらの簡単なサンプルアプリを使って話を進めていきたいと思います。
https://github.com/masa-tokyo/sendgrid_demo_app

![](/images/sendgrid-demo-app/demo_app.png =250x)
*メールアドレスを入力するテキストフォーム*
入力して送信ボタンを押すと、以下のようなメールが送信されます。
![](/images/articles/sendgrid-demo-app/email_page.png)


# 実装方法
## ①メール配信システムの選定
まずは、連携して実際にメールを送ってくれるメール配信システムを選定します。

こちらの記事にもあるように、様々なタイプのサービスがあるようです。
https://www.aspicjapan.org/asu/article/928

実務で利用する際にはその要件によって採用するものが変わってくると思いますが、今回は12,000通/月までの無料枠がある（2022年12月時点)SendGridを利用します。

アカウントは以下を参考に作成してください。  
https://sendgrid.kke.co.jp/blog/?p=183


## ②画面作成

以下のようにテキストフォームとボタンを作成します。

```dart
class MyHomePage extends StatefulWidget {
  const MyHomePage({
    super.key,
  });

  @override
  State<MyHomePage> createState() => _MyHomePageState();
}

class _MyHomePageState extends State<MyHomePage> {
  final _controller = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('Sendgrid Demo App'),
      ),
      body: Center(
        child: Column(
          mainAxisAlignment: MainAxisAlignment.center,
          children: <Widget>[
            Padding(
              padding: const EdgeInsets.symmetric(horizontal: 36),
              child: TextFormField(
                controller: _controller,
                decoration: const InputDecoration(
                  hintText: 'xxxxx@gmail.com',
                ),
              ),
            ),
            const SizedBox(height: 24),
            ElevatedButton(
                onPressed: () {
                  // 「⑤ボタン押下時の処理」で追記
                },
                child: const Text('送信')),
          ],
        ),
      ),
    );
  }
}
```

## ③Firebaseプロジェクト作成

![](/images/articles/sendgrid-demo-app/add_new_project.png)
Firebaseコンソールから以下のことを行いましょう。

- 新規プロジェクトの作成
- アプリ（iOS/Android）の追加
- firestoreの初期設定

こちらの詳細は、Flutter大学メンバーのとっくさんの記事で分かりやすく説明してくださっています🙌  
https://tokku-engineer.tech/create-pj-flutterfirebase-ios-android-initial/

Firebaseのプロジェクトを作成したら、以下のコマンドでパッケージを追加しましょう。  
```
flutter pub add firebase_core
``` 

``` 
flutter pub add cloud_firestore
``` 


main関数内でも以下のように初期化をしておきます。

```dart
void main() async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp();
  runApp(const MyApp());
}
```

## ④拡張機能の追加


Firebaseコンソールより「Trigger Email」という拡張機能を追加します。
![](/images/articles/sendgrid-demo-app/add_extension.png)

![](/images/articles/sendgrid-demo-app/config_extension.png)

| 項目                         | 説明                                                                                                                                                                                                                                                                               |
|----------------------------|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Cloud Functions Location   | firestoreのregionを東京にしているならここもasia-northeast1で良いかと思います。                                                                                                                                                                                                                           |
| SMTP connection URI        | 下記で詳細説明します。                                                                                                                                                                                                                                                                      |
| SMTP password              | 下記で詳細説明します。                                                                                                                                                                                                                                                                      |
| Email documents collection | このコレクションへのドキュメントの追加をトリガーにメールが送信されます。                                                                                                                                                                                                                                             |
| Default FROM address       | 送信者としてメールに表示されるアドレス。<br>ここに記載するアドレスは、SendGrid側で独自ドメインとして設定することが推奨されています。この設定をしない状態だと、迷惑メールフォルダに入ってしまったり、「このメールにはご注意ください」というような運営側からの警告メッセージが文面上に表示されてしまうことがあります。<br>設定方法は[こちら](https://sendgrid.kke.co.jp/docs/Tutorials/D_Improve_Deliverability/using_whitelabel.html)をご覧ください。 |

**SMTP connection URI**  
ここには実際にメール送信を行ってくれるメールサービスで作成したURIを記載します。  
今回はSendGridを用いるため、SendGridのユーザーページから作成します。

ユーザーページ左のメニュー欄から「Email API > Integration Guide > SMTP Relay」を選択して作成してください。  
![](/images/articles/sendgrid-demo-app/sendgrid_console.png)

ここに表示されている情報を`smtps://{Username}@{Server}:{Port}`という形式で記載します。

**SMTP password**  
「Create an API key」からキーを作成後に表示される「Password」欄の値を入力します。  
ここで追加したAPIキーは「Settings > API Keys」から編集可能です。

## ⑤ボタン押下時の処理

ElevatedButtonのonPressedの処理を記載します。  
先ほど設定したemailsコレクションにデータを追加します。
```dart
onPressed: () async {
  try {
    await FirebaseFirestore
        .instance
        .collection('emails')
        .add({
      'to': _controller.text,
      'message': {
      'subject': 'Hello from Flutter!',
      'html':
        'このメールは${_controller.text}宛てにSendGridDemoAppから送信されました。'
        '<br>'
        'html形式で文章を作成することが出来ます。'
      }
        });
      // ドキュメント追加後にテキストフォームを空にする
      setState(() {
      _controller.clear();
      });
  } on Exception catch (e) {
      debugPrint(e.toString());
  }
},

```

これで無事完成です🎉  
https://github.com/masa-tokyo/sendgrid_demo_app

# 最後に
お読みいただきありがとうございました。  
最後にちょっとした宣伝なのですが、実際の業務でSendGridを用いた事例を勉強会で紹介します。具体的なイメージが湧いて実装の参考になれば幸いです。  
お時間合えばぜひご参加ください↓   
https://docodoor.connpass.com/event/263871/  

※ イベント終了後追記   
上記リンク先の概要欄にYouTubeライブのアーカイブが残っているのでぜひご覧ください！

# 参考
https://tokku-engineer.tech/create-pj-flutterfirebase-ios-android-initial/
https://zenn.dev/ryo_kawamata/articles/firebase-trigger-email