---
title: "Flutterfire CLI で 環境分けをする"  
emoji: "🔨"   
type: "tech"  
topics: ["Flutter", "AI", "Gemini"]
published: true
---

## はじめに
少し前に Andrea さんのこちらの記事で、Flutterfire CLI を使って環境構築する、と言う記事が出ていたので試してみました。

https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/

これまでは、以下記事の村松さんのものがとてもお手軽で使わせてもらっていたのですが、`--flavor`を使ったことが無く、知見を広げる意味でも試してみました。

https://zenn.dev/altiveinc/articles/separating-environments-in-flutter-old-edition

## それぞれの手順

まずは Flutter プロジェクトを作成しましょう。

```shell
flutter create flavor_sample_app -e
```

ちなみに、`-e` オプションをつけることでお馴染みのカウンターアプリよりもシンプルな空のプロジェクトが出来ます。

`pubspec.yaml` に `flutter_flavorizr` を追加します：

```yaml
dev_dependencies:
  flutter_flavorizr: ^2.2.3
 ```

前提として、iOS環境にはいかが必要になります：
- [Ruby](https://www.ruby-lang.org/en/documentation/installation/)
- [Gem](https://rubygems.org/pages/download)
- [Xcodeproj](https://github.com/CocoaPods/Xcodeproj)

※ Flutterバージョン3.27.0以降でプロジェクトを作成している場合、iosディレクトリ配下に`Podfile`が作られないため、自身で用意しておきましょう。


## 感想

// TODO: comment after actually trying it out
実際にやってみて、
pros
- 開発環境もリリースモードで作れる

cons
- 割と手間？

## 代替案
// TODO: alternative solution if any
二つを組み合わせて以下のようにするのが良いかもしれません

## 参考
// TODO: consider referring to his course or not

## 終わりに