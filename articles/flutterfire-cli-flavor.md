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
flutter create flavor_sample_app --empty
```

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
// todo explain more about `flutter run --flavor dev`

利用しているIDEを`flavorizr.yaml`へ以下のように設定します。

VS Codeを利用している場合：
```yaml
ide: "vscode"
flavors:
  dev:
# ...
```

Android StudioやIntelliJ IDEAを利用している場合：
```yaml
ide: "idea"
flavors:
  dev:
# ...
```

設定後、以下のコマンドを実行します：

```shell
dart run flutter_flavorizr -p ide:config
```

実際にビルドしてみると、以下のようにflavorごとのアプリがビルドされます：
|                                         本番環境                                         |                                         開発環境                                         |
|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|
|![](https://storage.googleapis.com/zenn-user-upload/9cc92e8e1222-20241218.png =300x)|![](https://storage.googleapis.com/zenn-user-upload/bc73a758bce8-20241218.png =300x)|


:::message
エントリーポイントについて

flavorに関わらず `main.dart`ファイル一つで簡潔に管理したいという方も多いかと思うのですが、

- セキュリティ的により安全（`dev`用の FirebaseOptions ファイルをアプリ配布時に含める必要が無くなる。詳細は[こちら](https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/#all-the-firebase-config-files-are-bundled-no-tree-shaking)）
- flutter_flavorizr が複数の`main._*.dart`等を自動生成してくれるため、手間が少なく済む

という理由から、今回はエントリーポイントを環境ごとに分けた運用にしています。

`main.dart`のみで管理したい場合、 生成された`*.xcconfig`ファイル内の`FLUTTER_TARGET`のフィールドを取り除く or `main.dart`へ変更することで可能のようです。
また、`dart run flutter_flavorizr`コマンド実行時の`main._*.dart`の生成が不要であれば、`-p`オプションにより[こちらのデフォルトの引数](https://pub.dev/packages/flutter_flavorizr#default-processors-set)から`flutter:*`を取り除いて実行するのが良いかと思います。

:::

### アイコン設定
上記までを行えば一通り環境分けが完了しますが、例えば以下のように環境ごとに異なるアイコンを設定することが可能です：

|                                         本番環境                                         |                                         開発環境                                         |
|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|
| ![](https://storage.googleapis.com/zenn-user-upload/c56a0423ec43-20241218.png =200x) | ![](https://storage.googleapis.com/zenn-user-upload/ecac23f49b5f-20241218.png =200x) |


それぞれのプラットフォーム向けに作成した画像を`assets`ディレクトリ内に格納し、そのパスを`flavorizr.yaml`に記載します。

```diff yaml
ide: "idea"
flavors:
  dev:
    app:
      name: "Flavor Sample App Dev"
    android:
      applicationId: "com.example.flavor_sample_app.dev"
+     icon: "assets/android_app_icon_dev.png"
+     adaptiveIcon:
+       foreground: "assets/android_app_icon_foreground_dev.png"
+       background: "assets/android_app_icon_background.png"
    ios:
      bundleId: "com.example.flavor_sample_app.dev"
+     icon: "assets/ios_app_icon_dev.png"
  prod:
    app:
      name: "Flavor Sample App"
    android:
      applicationId: "com.example.flavor_sample_app"
+     icon: "assets/android_app_icon.png"
+     adaptiveIcon:
+      foreground: "assets/android_app_icon_foreground.png"
+      background: "assets/android_app_icon_background.png"
    ios:
      bundleId: "com.example.flavor_sample_app"
+     icon: "assets/ios_app_icon.png"

```

Android用アイコンを反映させるために以下のコマンドを実行します：
```shell
dart run flutter_flavorizr -p android:icons
```

すると、各サイズの画像生成や最初に実行したコマンドで生成されていたダミーのアイコン画像からの置き換えをよしなにしてくれます：
![](https://storage.googleapis.com/zenn-user-upload/d34f5201bad0-20241218.png)

iOS側も同様に以下のコマンドを実行します：
```shell
dart run flutter_flavorizr -p ios:icons
```

## Firebaseプロジェクトの準備

https://console.firebase.google.com/

上記のページから、環境ごとのFirebaseプロジェクトを作成します：
![](https://storage.googleapis.com/zenn-user-upload/7692578ca4ee-20241219.png)

## FlutterFire CLI 実行

### インストール

Firebaseの環境設定には iOS用の`GoogleService-Info.plist`やAndroid用の`google-services.json`が必要になります。

これらのファイルは各環境のFirebaseプロジェクトごとにコンソールからダウンロードする必要がありますが、FlutterFire CLIを利用することでこの手順を簡略化することが出来ます。

まずは、前段階として`firebase --version`コマンドでバージョンが表示されない場合、Firebase CLIをインストールします：

```shell
npm install -g firebase-tools
```

以下のコマンドでFirebaseプロジェクトを作成したアカウントへログインします：
```shell
firebase login
```

以下コマンドでFlutterFireCLI(`1.0.0`以上が必要)をインストールします：
```shell
dart pub global activate flutterfire_cli
```

※ 過去にインストール済みの場合でも、依存関係が原因で再実行が必要な場合があります

### コマンド実行

各環境ごとに以下コマンドを実行します。

dev環境の場合：
```shell
flutterfire config \
  --project=flavor-sample-app-dev \
  --out=lib/firebase_options_dev.dart \
  --ios-bundle-id=com.example.flavorSampleApp.dev \
  --ios-out=ios/flavors/dev/GoogleService-Info.plist \
  --android-package-name=com.codewithandrea.flavor_sample_app.dev \
  --android-out=android/app/src/dev/google-services.json
```

prod環境の場合：
```shell
flutterfire config \
  --project=flavor-sample-app-prod \
  --out=lib/firebase_options_prod.dart \
  --ios-bundle-id=com.example.flavorSampleApp \
  --ios-out=ios/flavors/prod/GoogleService-Info.plist \
  --android-package-name=com.codewithandrea.flavor_sample_app \
  --android-out=android/app/src/prod/google-services.json
```

- ` --project`は、自身のFirebaseプロジェクトIDを指定します
- `--ios-bundle-id`は、iOSのバンドルIDを指定します
- `--android-package-name`は、AndroidのApplicationIdを指定します

### プロンプト回答

実際にコマンドを実行すると、プロンプトが表示されるので、それに沿って回答していきます。

「Build configuration」を選択します：
```shell
? You have to choose a configuration type. Either build configuration (most likely choice) or a target set up. ›                                                                                         
❯ Build configuration                                                                                                                                                                                    
  Target        
```

::: details
ここでエラーが出てしまったら...

```shell
Exception: /Users/masaki/.rbenv/versions/3.2.2/lib/ruby/site_ruby/3.2.0/rubygems/specification.rb:2242:in `check_version_conflict': can't activate rexml-3.2.8, already activated rexml-3.4.0 (Gem::LoadError)
```
上記のようなエラーが出てしまった場合、gemによりインストールされた`xcodeproj`と`rexml`のバージョンが競合している可能性があります。

```shell
gem list rexml
```
によりインストール済みの`rexml`を確認し、`xcodeproj --version`と互換性の無いものを以下のように削除しましょう：

```shell
gem uninstall rexml --version 3.2.8
```

:::

## アプリビルド

パッケージ追加

### Android

### iOS

### エントリーポイント



## 感想

// TODO: comment after actually trying it out
実際にやってみて、
pros
- 開発環境もリリースモードで作れる?
- dart-from-file不要で済む

cons
- 割と手間？

## 代替案
// TODO: alternative solution if any
二つを組み合わせて以下のようにするのが良いかもしれません


# 終わりに


# 参考
// TODO: add references