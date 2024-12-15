---
title: "Flutterfire CLI で 環境分けをする"  
emoji: "🔨"   
type: "tech"  
topics: ["Flutter", "AI", "Gemini"]
published: true
---

# はじめに
少し前に Andrea さんのこちらの記事で、Flutterfire CLI を使って環境構築する、と言う記事が出ていたので試してみました。

https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/

これまでは、以下記事の村松さんのものがとてもお手軽で使わせてもらっていたのですが、`--flavor`を使ったことが無く、知見を広げる意味でも試してみました。

https://zenn.dev/altiveinc/articles/separating-environments-in-flutter-old-edition

# flutter_flavorizr 導入

## 前提

このパッケージの利用については、既存プロジェクトにおいて過去に行っていた設定を想定外に上書きしてしまう可能性があるため、新規プロジェクトでの利用が推奨されています。

iOS環境構築には以下が必要になります：
- [Ruby](https://www.ruby-lang.org/en/documentation/installation/)
- [Gem](https://rubygems.org/pages/download)
- [Xcodeproj](https://github.com/CocoaPods/Xcodeproj)

尚、今回の実行環境は、Flutterバージョン3.27.0を利用しています。

※ [こちらの記載](https://pub.dev/packages/flutter_flavorizr#prerequisites:~:text=If%20your%20app%20uses%20a%20Flutter%20plugin%20and%20you%20plan%20to%20create%20flavors%20for%20iOS%20and%20macOS%2C%20you%20need%20to%20make%20sure%20there%27s%20an%20existing%20Podfile%20file%20under%20the%20ios/macos%20folder.)では iosディレクトリ配下に`Podfile`が必要とありますが、今回の実行環境であるFlutter 3.27.0 （flutter create時には`Podfile`が作成されない）では不要です。

## 初期設定

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

プロジェクトルートに `flavorizr.yaml`というファイルを作成し、以下のように記載します：

```yaml
flavors:
  dev:
    app:
      name: "Flavor Sample App Dev"
    android:
      applicationId: "com.example.flavor_sample_app.dev"
    ios:
      bundleId: "com.example.flavor_sample_app.dev"
  prod:
    app:
      name: "Flavor Sample App"
    android:
      applicationId: "com.example.flavor_sample_app"
    ios:
      bundleId: "com.example.flavor_sample_app"
```

ここでは、`dev` と `prod` の2つの環境を用意しています。


その後、以下コマンドを実行します：
```shell
dart run flutter_flavorizr
```


# 感想

// TODO: comment after actually trying it out
実際にやってみて、
pros
- 開発環境もリリースモードで作れる

cons
- 割と手間？

# 代替案
// TODO: alternative solution if any
二つを組み合わせて以下のようにするのが良いかもしれません


# 終わりに


# 参考
// TODO: add references