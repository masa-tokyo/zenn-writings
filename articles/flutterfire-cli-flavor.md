---
title: "Flutterfire CLI で 環境分けをする"  
emoji: "🔨"   
type: "tech"  
topics: ["Flutter", "flavor", "cli"]
published: false
---

## はじめに
少し前に Andrea さんのこちらの記事で、Flutterfire CLI を使って環境構築する、と言う記事が出ていたので試してみました。

https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/

これまでは、以下記事の村松さんのものがとてもお手軽で使わせてもらっていたのですが、`--flavor`を使ったことが無く、知見を広げる意味でも試してみました。

https://zenn.dev/altiveinc/articles/separating-environments-in-flutter-old-edition

## flutter_flavorizr 導入

// TODO explain about flutter_flavorizr

### 前提

このパッケージの利用については、既存プロジェクトにおいて過去に行っていた設定を想定外に上書きしてしまう可能性があるため、新規プロジェクトでの利用が推奨されています。

iOS環境構築には以下が必要になります：
- [Ruby](https://www.ruby-lang.org/en/documentation/installation/)
- [Gem](https://rubygems.org/pages/download)
- [Xcodeproj](https://github.com/CocoaPods/Xcodeproj) (gem経由でのインストールが必要)

iOSのターゲットバージョンは12.0以上である必要があるようです（詳細は[こちら](https://github.com/AngeloAvv/flutter_flavorizr/blob/master/doc/troubleshooting/unable-to-load-contents-of-file-list/README.md#:~:text=Before%20running%20pod%20install%20again%2C%20we%20need%20to%20make%20sure%20that%20the%20XCode%20Workspace%20is%20targeting%20at%20least%20iOS%2012.0))。
※ [pub.dev上の記載](https://pub.dev/packages/flutter_flavorizr#prerequisites:~:text=If%20your%20app%20uses%20a%20Flutter%20plugin%20and%20you%20plan%20to%20create%20flavors%20for%20iOS%20and%20macOS%2C%20you%20need%20to%20make%20sure%20there%27s%20an%20existing%20Podfile%20file%20under%20the%20ios/macos%20folder.)では iosディレクトリ配下に`Podfile`が必要とありますが、今回の実行環境であるFlutter 3.27.0（flutter create時に`Podfile`は作成されないが`iOS Deployment Target`が12.0に設定されている）では不要です。

### 初期設定

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

以下のコマンドを実行することで、諸々の設定を一括して行なってくれます：
```shell
dart run flutter_flavorizr
```

## ビルド引数設定
利用しているIDEを`flavorizr.yaml`へ以下のように設定します。

VS Codeを利用している場合：
```yaml
ide: "vscode"
```

Android StudioやIntelliJ IDEAを利用している場合：
```yaml
ide: "idea"
```

設定後、以下のコマンドを実行します：

```shell
dart run flutter_flavorizr -p ide:config
```

実際にビルドしてみると、以下のようにflavorごとの文字列が表示されたアプリがビルドされます：
![](https://storage.googleapis.com/zenn-user-upload/995470d8a4be-20241218.png =300x)

:::message
エントリーポイントについて

flavorに関わらず `main.dart`ファイル一つで簡潔に管理したいという方も多いかと思うのですが、

- セキュリティ的により安全（`dev`用の FirebaseOptions ファイルをアプリ配布時に含める必要が無くなる。詳細は[こちら](https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/#all-the-firebase-config-files-are-bundled-no-tree-shaking)）
- flutter_flavorizr が複数の`main._*.dart`等を自動生成してくれるため、手間が少なく済む

という理由から、今回はエントリーポイントを環境ごとに分けた運用にしています。

`main.dart`のみで管理したい場合、 生成された`*.xcconfig`ファイル内の`FLUTTER_TARGET`のフィールドを取り除く or `main.dart`へ変更することで可能となります。
また、`dart run flutter_flavorizr`コマンド実行時の`main._*.dart`の生成が不要であれば、`-p`オプションにより[こちらのデフォルトの引数](https://pub.dev/packages/flutter_flavorizr#default-processors-set)から`flutter:*`を取り除いて実行するのが良いかと思います。

:::

### アイコン設定
（アイコンを環境ごとに切り替えたい場合のみ）

以下のように画像を`assets`ディレクトリ内に格納します。


以下のようにそのパスを`flavorizr.yaml`に記載します。

```yaml

```

Android用アイコンを反映させるために以下のコマンドを実行します：
```shell
dart run flutter_flavorizr -p android:icons
```

iOS用アイコンを反映させるために以下のコマンドを実行します：
```shell
dart run flutter_flavorizr -p ios:icons
```

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


# 終わりに


# 参考
// TODO: add references