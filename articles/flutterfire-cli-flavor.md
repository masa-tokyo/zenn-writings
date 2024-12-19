---
title: "Flutterfire CLI で Flutter x Firebase の環境分けをする"  
emoji: "🔨"   
type: "tech"  
topics: ["Flutter", "flavor", "Firebase", "cli"]
published: false
---

## はじめに

皆さんは普段のFlutter開発において、どのように環境分けをしていますか？
特に、Firebaseを利用しているプロジェクトにおいてはプラットフォームごとの設定ファイルの扱いもあり少し手間かと思います。
今回は、Flutterfire CLI を使った以下の記事のような方法がとても良いと思ったのでご紹介します。

https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/


## flutter_flavorizr 導入

`Flutterfire CLI`を使った環境分けには、`flutter run --flavor dev`のようにflavorオプションによるアプリビルドが必要になります。

https://pub.dev/packages/flutter_flavorizr

`flutter_flavorizr`というパッケージを利用することで、この設定をとてもスムーズに行うことが出来ます。

### 前提

このパッケージの利用については、既存プロジェクトにおいて過去に行っていた設定を想定外に上書きしてしまう可能性があるため、新規プロジェクトでの利用が推奨されています。

iOS環境構築には以下が必要になります：
- [Ruby](https://www.ruby-lang.org/en/documentation/installation/)
- [Gem](https://rubygems.org/pages/download)
- [Xcodeproj](https://github.com/CocoaPods/Xcodeproj) (gem経由でのインストールが必要)

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
      bundleId: "com.example.flavorSampleApp.dev"
  prod:
    app:
      name: "Flavor Sample App"
    android:
      applicationId: "com.example.flavor_sample_app"
    ios:
      bundleId: "com.example.flavorSampleApp"
```

ここでは、`dev` と `prod` の2つの環境を用意しています(`stg`環境も必要な場合は`dev`同様に追加してください）。

以下のコマンドを実行することで、諸々の設定を全て一括で行なってくれます：
```shell
dart run flutter_flavorizr
```

## ビルド引数設定

上記の実行だけで、以下コマンドのように環境ごとのアプリがビルド出来るようになります：

```shell
flutter run --flavor dev -t lib/main_dev.dart
flutter run --flavor prod -t lib/main_prod.dart
```

実際にビルドしてみると、以下のようにflavorごとのアプリがビルドされます：
|                                         本番環境                                         |                                         開発環境                                         |
|:------------------------------------------------------------------------------------:|:------------------------------------------------------------------------------------:|
|![](https://storage.googleapis.com/zenn-user-upload/9cc92e8e1222-20241218.png =300x)|![](https://storage.googleapis.com/zenn-user-upload/bc73a758bce8-20241218.png =300x)|


:::message
エントリーポイントについて

flutter_flavorizr のデフォルトの設定に則り、今回は`main._*.dart`のように環境ごとにエントリーポイントを分けています。

`main.dart`のみで管理したい場合、 生成された`*.xcconfig`ファイル内の`FLUTTER_TARGET`のフィールドを取り除く or `main.dart`へ変更することで可能のようです。
また、`dart run flutter_flavorizr`コマンド実行時の`main._*.dart`等の生成を避けたければ、`dart run flutter_flavorizr`の一括実行の代わりに`-p`オプションを利用することで回避可能です。
[こちらにあるデフォルトの引数](https://pub.dev/packages/flutter_flavorizr#default-processors-set)から`flutter:*`を取り除いて実行するのが良いかと思います。

:::

尚、`flutter run --flavor dev -t lib/main_dev.dart`のようなビルド引数を、`flavorizr.yaml`からIDEへ設定することが出来ます。

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

上記のように記載後、以下のコマンドを実行します：

```shell
dart run flutter_flavorizr -p ide:config
```


### アイコン設定
ここまでを行えば一通りの環境分けは完了ですが、例えば以下のように環境ごとにアイコンを設定することも可能です。

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
      bundleId: "com.example.flavorSampleApp.dev"
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
      bundleId: "com.example.flavorSampleApp"
+     icon: "assets/ios_app_icon.png"

```

Android用アイコンを反映させるために以下のコマンドを実行します：
```shell
dart run flutter_flavorizr -p android:icons
```

すると、各サイズの画像生成や`dart run flutter_flavorizr`で生成されていたダミーのアイコン画像からの置き換えをよしなにしてくれます：
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

実際に各環境ごとに以下コマンドを実行していきます。

dev環境の場合：
```shell
flutterfire config \
  --project=flavor-sample-app-dev \
  --out=lib/firebase_options_dev.dart \
  --ios-bundle-id=com.example.flavorSampleApp.dev \
  --ios-out=ios/flavors/dev/GoogleService-Info.plist \
  --android-package-name=com.example.flavor_sample_app.dev \
  --android-out=android/app/src/dev/google-services.json
```

prod環境の場合：
```shell
flutterfire config \
  --project=flavor-sample-app-prod \
  --out=lib/firebase_options_prod.dart \
  --ios-bundle-id=com.example.flavorSampleApp \
  --ios-out=ios/flavors/prod/GoogleService-Info.plist \
  --android-package-name=com.example.flavor_sample_app \
  --android-out=android/app/src/prod/google-services.json
```

- ` --project`は、自身のFirebaseプロジェクトIDを指定します
- `--ios-bundle-id`は、iOSのバンドルIDを指定します
- `--android-package-name`は、AndroidのApplicationIdを指定します


実際にコマンドを実行すると、プロンプトが表示されるので、それに沿って回答していきます。

`Build configuration`を選択します：
```shell
? You have to choose a configuration type. Either build configuration (most likely choice) or a target set up. ›                                                                                         
❯ Build configuration                                                                                                                                                                                    
  Target        
```

::: details  ここでエラーが出てしまったら...

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

`dev`環境の場合は`Debug-dev`、`prod`環境の場合は`Debug-prod`を選択します：
```shell
? Please choose one of the following build configurations ›                                                                                                                                              
  Debug                                                                                                                                                                                                  
  Release                                                                                                                                                                                                
  Profile                                                                                                                                                                                                
❯ Debug-dev                                                                                                                                                                                              
  Profile-dev                                                                                                                                                                                            
  Release-dev                                                                                                                                                                                            
  Debug-prod                                                                                                                                                                                             
  Profile-prod                                                                                                                                                                                           
  Release-prod        
```

設定したいプラットフォームを選択します：
```shell
? Which platforms should your configuration support (use arrow keys & space to select)? ›                                                                                                                
✔ android                                                                                                                                                                                                
✔ ios                                                                                                                                                                                                    
  macos                                                                                                                                                                                                  
✔ web                                                                                                                                                                                                    
  windows      
```

処理が完了すると以下のように、プロジェクト上にアプリが作成されています：
![](https://storage.googleapis.com/zenn-user-upload/6960cda656be-20241220.png)

また、コマンド上で指定したアウトプット先に`firebase_options.dart`、`GoogleService-Info.plist`、`google-services.json`が作成されています。


## firebase_coreパッケージ導入

ここからは、実際にアプリをビルドするための準備をしていきます。
`firebase_core`パッケージをインストールします：

```shell
flutter pub add firebase_core
```

iOSプラットフォームバージョンは`13.0`以上にする必要があるため、`ios/Podfile`にて以下のように設定します：
```Podfile
# Uncomment this line to define a global platform for your project
platform :ios, '13.0'
```

:::message 
Podfileについて
Flutterバージョン3.27以降の場合、プロジェクト作成時点ではPodfileはありませんが、上記の`flutter pub add firebase_core`時点で追加されます。
これに際して、Podfile内のRunnerについての記載箇所にflavorを反映させて以下のようにしておくと良いかと思います：

```diff Podfile
project 'Runner', {
-  'Debug' => :debug,
-  'Profile' => :release,
-  'Release' => :release,
+  'Debug-dev' => :debug,
+  'Profile-dev' => :release,
+  'Release-dev' => :release,
+  'Debug-prod' => :debug,
+  'Profile-prod' => :release,
+  'Release-prod' => :release,
}
```
※`dart run flutter_flavorizr -p ios:podfile`実行でも反映されます

:::

### Firebase初期化

実際にDartファイル内でFirebaseを初期化しましょう。
今回はエントリーポイントを環境ごとに用意しているため、以下のコマンドでビルド出来るように設定します：
```shell
flutter run --flavor dev -t lib/main_dev.dart
flutter run --flavor prod -t lib/main_prod.dart
```

エントリーポイントとなる`main._*.dart`ファイルにて、FirebaseOptionsをimportします：
    
```file: main_dev.dart
import 'package:flavor_sample_app/firebase_options_dev.dart';

import 'flavors.dart';
import 'main.dart';

void main() async {
  F.appFlavor = Flavor.dev;
  runMainApp(DefaultFirebaseOptions.currentPlatform);
}
```

```file: main_prod.dart
import 'package:flavor_sample_app/firebase_options_prod.dart';

import 'flavors.dart';
import 'main.dart';

void main() async {
  F.appFlavor = Flavor.prod;
  runMainApp(DefaultFirebaseOptions.currentPlatform);
}
```

そして、`main.dart`ではFirebaseOptionsを受け取ってFirebaseの初期化処理を実行出来るようにします：

```file: main.dart
import 'package:firebase_core/firebase_core.dart';
import 'package:flutter/material.dart';

import 'app.dart';

void runMainApp(FirebaseOptions firebaseOptions) async {
  WidgetsFlutterBinding.ensureInitialized();
  await Firebase.initializeApp(options: firebaseOptions);
  runApp(const App());
}
```

## 終わりに

お疲れ様でした！
色々とプラットフォームごとのファイルをいじる必要がなく、とてもシンプルに環境構築が出来たかと思います。
修正箇所等ありましたら、コメントでお知らせいただけますと幸いです🙌

## 参考記事
https://codewithandrea.com/articles/flutter-firebase-multiple-flavors-flutterfire-cli/
https://zenn.dev/ymgn____/articles/f087e2fac830a8
