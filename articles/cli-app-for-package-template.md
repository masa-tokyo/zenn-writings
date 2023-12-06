---
title: "自作CLIツールでテンプレート作成をしてみる"  
emoji: "💻"   
type: "tech"  
topics: ["Flutter", "Dart", "GitHub"]
published: false
---
この記事は、[Flutter 大学アドベントカレンダー 2023](https://qiita.com/advent-calendar/2023/flutteruniv) 6日目の記事です。
最近仕事先で初めてDart製のCLIツールを作ったのですが、その過程がとても有益で勉強になったため、この機会に記事にまとめておくことにしました。

# 読んでみてほしい人
こんな方に読んでもらえたら嬉しいです🙌  
- Dartのコンソールアプリってなにという人  
- 自作のコンポーネントを複数のリポジトリから使って便利に個人開発したい人  
- 日頃からFlutterパッケージ作成を行っている人

# 今回作るもの

![](/images/articles/cli-app-for-package-template/cli_app_behavior.gif)

コマンドをターミナル上に入力してアプリを起動することで、pubspec.yamlファイルの更新や作業用ファイルの作成など、パッケージ開発を行う上で必要な処理を一括して行えるようになっています。

# このリポジトリの用途
今回CLIツールを導入した現場では、デザインを統一したUIコンポーネントを複数リポジトリから利用したい、という目的で社内用のパッケージ化を進めていました。
詳細ご興味ある方はこちらに過去勉強会のアーカイブがあるのでご覧ください。
https://www.youtube.com/live/IpcFqABBKvU?si=BoaeJt8J2tD8GazI&t=4241

この現場のモノレポ構成のリポジトリ内では、毎回Flutterパッケージを新規作成後にいくつか決まって行う必要のある処理があるのですが、この処理が段々と増えてきて煩雑になってきたためこの際自動化しよう、となったのがCLIツールを作ることになった経緯でした。

ただ、この話は上記のようなチーム開発に限ったものではなく、個人の開発者にとっても有用なのかなと思います。
自分がよく使う汎用性の高いソースコードをこのリポジトリ内に置いておき、今後の個人開発の際に別リポジトリから利用するということが出来たらとても便利そうです。


# プロジェクト作成
ここからは実際にステップを追って作成していきます。
今回作るサンプルリポジトリを以下に置いておくので、適宜参考にしてください。
https://github.com/masa-tokyo/flutter_toolkit

まずはいつも通り、
```shell
flutter create <プロジェクト名>
```
でプロジェクトを作成します。  
今回のプロジェクトは個人開発の際に使えそうなflutter関連の色んな汎用的なものを詰め込む用なので、`flutter_toolkit`としました。

このプロジェクトでは[fvm](https://fvm.app/)でバージョン管理を行うので、以下コマンドでstableバージョンを用いるようにします。
```shell
fvm use stable
``` 

その後、[melos](https://pub.dev/packages/melos)を用いたマルチパッケージ構成にしていきます。
melosについての詳細はこちらの記事が参考になります。
https://zenn.dev/altiveinc/articles/melos-for-multiple-packages-dart-projects

melos導入が初めての場合は以下でインストールし、
```shell
dart pub global activate melos
```

pubspec.yamlにも追加します。
```shell
fvm flutter pub add melos
```

不要なプラットフォーム関連のディレクトリを消して、以下のような構成にします。

```
.
├── README.md
├── analysis_options.yaml
├── melos.yaml
├── packages // ここにパッケージを追加していく
├── pubspec.lock
└── pubspec.yaml
```

`melos.yaml`には以下を記述します。

```yaml: melos.yaml
# プロジェクト名
name: flutter_toolkit

# 対象を指定
packages:
  - packages/**

# flutterバージョンはfvmを参照
sdkPath: '.fvm/flutter_sdk'

# `melos bs`実行時にpubspec.yaml上のdartバージョン表記を一括で設定
command:
  bootstrap:
    environment:
      sdk: ^3.2.0

```

:::message
バージョン表記について
通常、
```shell
fvm flutter create -t package <パッケージ名>
```
でパッケージを作成すると、以下のような表記になります。
```yaml: pubspec.yaml
environment:
  sdk: '>=3.2.2 <4.0.0'
  flutter: ">=1.17.0"
```
ただし、今回のプロジェクトでは常にfvmで最新stableを用いるため紛らわしさを避けるため（実際には1.17.0以上が使える訳ではないため）、Flutterバージョンは表記しないこととします。
また、DartのSDKバージョンについては四半期ごとのstableリリースのタイミングで表記を更新する(patchバージョンまで細かく更新するのが大変なため）、という意味合いで`sdk: ^3.2.0`のようにキャレット記号で書いておくくらいがちょうど良いのかなと思います。
:::

また、このプロジェクト内では統一して`pedantic_mono`をlintとして使いたいため、`flutter_lints`から置き換えておきます。  

```yaml: pubspec.yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  pedantic_mono: any
```

```yaml: analysis_options.yaml
# https://pub.dev/packages/pedantic_mono
include: package:pedantic_mono/analysis_options.yaml

# 今後ここに独自のルールを追加していくため、このリポジトリ内は全てこのルールで統一したい
linter:
  rules:
    avoid_classes_with_only_static_members: false

```


# パッケージテンプレートの作成

ここから本題のテンプレートの作成に入ります。  

一般にパッケージを作成する際には、
```shell
fvm flutter create -t package <パッケージ名>
```
を実行しますが、作成後にいくつか毎度修正したい箇所が出てきます。　　
そういった箇所を自動で修正するために、テンプレートを作成していきます。　　

:::message
masonとの比較
テンプレート作成、と聞くと「それ[mason](https://pub.dev/packages/mason)じゃダメなの？」と思われる方もいるかもしれません。
実際、ここで行っている多くのことはmasonでも実現出来るので、masonの扱いに慣れていればそちらのが実装は早いと思います。
ただ、これから説明する`overwritePubspecYamlFile`関数内で行っているような、コマンド実行時に渡した引数を元にした特定箇所の書き換え、Dartバージョンの動的な取得、単体テスト実行のような複雑なことをしたい場合には、CLIツールを使う必要があるかなと思います。
:::

## テンプレート生成用Dartアプリの作成
ターミナル上で起動するDartのコンソールアプリを作成します。  
今回は`scripts`というディレクトリ配下に`bootstrap_package`という名前で作成します。

```shell
fvm dart create bootstrap_package
```

melos.yamlに`scripts`ディレクトリを追加しておきます。
```yaml: melos.yaml
# 対象を指定
packages:
  - packages/**
  - scripts/**
```

pubspec.yaml内に以下のようにパスを設定することでこのアプリのコマンド実行を端的に行えるようになります。
```yaml: pubspec.yaml
dev_dependencies:
  bootstrap_package:
    path: scripts/bootstrap_package
```

## 処理の実装
Dartアプリではlibとは別にbinというフォルダがあり、main関数はこちらに置かれるのですが、今回は以下のようにして主要な処理をlibフォルダ内の`ruCommand`関数で行うようにします。

```dart: bin/bootstrap_package.dart
import 'package:bootstrap_package/run_command.dart';

/// 新規パッケージ作成用の関数
///
/// プロジェクトのルートディレクトリから以下のコマンドにより実行する。
/// `fvm dart run bootstrap_package <パッケージ名>`
/// 実際の処理内容は[runCommand]に記述する。
void main(List<String> args) => runCommand(args);
```

### runCommand

ちょっと長くなってしまいますが、runCommand関数は以下のようになっています。
```dart: lib/run_command.dart
import 'dart:io';

import 'package:args/args.dart';
import 'package:bootstrap_package/overwrite_licence_file.dart';
import 'package:bootstrap_package/show_usage.dart';
import 'package:path/path.dart' as path;

import 'create_working_file.dart';
import 'overwrite_pubspec_yaml_file.dart';
import 'overwrite_test_file.dart';

/// コマンド実行用関数
void runCommand(List<String> args) {
  try {
    // 引数を定義
    final parser = ArgParser()
      ..addOption(
        'description',
        abbr: 'd',
      )
      ..addFlag(
        'help',
        abbr: 'h',
        negatable: false,
      );
    final parsedArgs = parser.parse(args);

    // helpオプションが指定された場合、使い方を表示して処理を終了
    final shouldHelp = parsedArgs['help'] as bool;
    if (shouldHelp) {
      showUsage();
      return;
    }

    // パッケージ名が入力されていない場合、エラー文を表示して処理を終了
    final name = parsedArgs.rest.firstOrNull;
    if (name == null) {
      showUsage(errorMessage: 'パッケージ名を指定してください。');
      return;
    }

    // packagesディレクトリへ移動
    Directory.current = Directory('packages');

    // パッケージ用のプロジェクトを作成
    Process.runSync(
      'fvm',
      ['flutter', 'create', '-t', 'package', name],
    );

    // 作成されたパッケージへ移動
    Directory.current = Directory(name);

    // 作業用ファイルを作成
    createWorkingFile(packageName: name);

    // analysis_options.yamlを削除し、プロジェクトルートのものをsymbolic linkで追加
    final analysisOptionsFile = File('analysis_options.yaml')..deleteSync();
    Link(analysisOptionsFile.path)
        .createSync(path.join('../..', analysisOptionsFile.path));

    // testファイルを上書き
    overwriteTestFile(packageName: name);

    // パッケージ説明が引数として指定されていない場合、パッケージ名から作成
    var description = parsedArgs['description'] as String?;
    description ??= '$name用Flutterパッケージ';

    // pubspec.yamlファイルを上書き
    overwritePubspecYamlFile(packageName: name, description: description);

    // LICENSEファイル削除し、プロジェクトルートのものをsymbolic linkで追加
    final licenseFile = File('LICENSE')..deleteSync();
    Link(licenseFile.path).createSync(path.join('../..', licenseFile.path));


    // READMEファイルをパッケージ名のみに上書き
    final packageTitle = '# $name';
    File('README.md').writeAsStringSync(packageTitle);
  } on FormatException catch (_) {
    // '-d'のようなoptionコマンドに続く引数が入力されていない場合
    showUsage(errorMessage: 'オプションコマンドの使い方が間違っています。');
  }
}
```
ここから要所要所を解説していきます。もし不明点等ありましたら遠慮なくコメントいただけばと思います🙏

### 引数の設定
こちらで[argsパッケージ](https://pub.dev/packages/args)を用いてコマンド実行時の引数を設定しています。

```dart: lib/run_command.dart
    // 引数を定義
    final parser = ArgParser()
      ..addOption(
        'description',
        abbr: 'd',
      )
      ..addFlag(
        'help',
        abbr: 'h',
        negatable: false,
      );
    final parsedArgs = parser.parse(args);

    // helpオプションが指定された場合、使い方を表示して処理を終了
    final shouldHelp = parsedArgs['help'] as bool;
    if (shouldHelp) {
      showUsage();
      return;
    }
```

```shell
fvm dart run bootstrap_package --help
```
の実行時には、`showUsage`により使い方を表示するようにしています。
```dart: lib/show_usage.dart
import 'dart:io';

/// 使い方をターミナル上に表示するための関数
///
/// helpオプションが指定された時や誤った使い方がされた時に用いる。
/// 誤った使い方がされた場合、[exitCode]を1にして[errorMessage]を表示する。
void showUsage({String? errorMessage}) {
  if (errorMessage != null) {
    exitCode = 1;
    // ターミナル上にエラーを出力する関数
    stderr.writeln('[ERROR] $errorMessage');
  }

  const usage = '''
    
Usage: fvm dart run bootstrap_package <パッケージ名> [options]

Options:
-d, --description <パッケージ説明>     パッケージの説明を指定
-h, --help　　　　　　　               使い方を表示

Example:
  fvm dart run bootstrap_package login_form -d "ログインフォーム用Flutterパッケージ"
    ''';
  // ターミナル上に出力
  stdout.writeln(usage);
}

```
これにより、いつも見るようなhelpっぽいものを表示出来るようになります🎉
![](/images/articles/cli-app-for-package-template/show_usage.png)

:::message
exitCodeについて  
[こちらのドキュメント](https://api.flutter.dev/flutter/dart-io/exitCode.html)にあるように、アプリ起動中にはグローバルに保持される変数で、正常状態以外で終了する際にはデフォルト値である0からそれ以外に設定すると良いようです。
ここでは[flutterfire_cli](https://github.com/invertase/flutterfire_cli/blob/9e7b659102146f97cee396a1365ecc5c8b848197/packages/flutterfire_cli/bin/flutterfire.dart#L71)の例に倣って1に設定しています。
:::

また、
```shell
fvm dart run bootstrap_package <パッケージ名> -d <パッケージ説明>
```
のように実行することで、後ほど紹介する`overwritePubspecYamlFile`関数にパッケージ説明を渡すことが出来ます。
```dart: lib/run_command.dart
    // パッケージ説明が引数として指定されていない場合、パッケージ名から作成
    var description = parsedArgs['description'] as String?;
    description ??= '$name用Flutterパッケージ';

    overwritePubspecYamlFile(packageName: name, description: description);
    
```

余談ですが、さらに必要な情報が増えてきて複数引数を渡したくなってきた場合には[cli_dialog](https://pub.dev/packages/cli_dialog)や[interact](https://pub.dev/packages/interact)のようなパッケージを用いて対話形式のインターフェースにするとよりCLIっぽくなって体験が良さそうです。

### 作業用ファイルの作成
[こちらの例](https://github.com/rrousselGit/riverpod/tree/master/packages/flutter_riverpod/lib)のように、libフォルダ直下にはexport文のみを記載したファイルを置き実装自体はsrcフォルダ内で行う、というのが一般的なようなので、その場所を用意します。
```dart: lib/create_working_file.dart
import 'dart:io';

import 'package:path/path.dart' as path;

/// 作業用ファイルを作成するための関数
///
/// srcディレクトリへファイルを作成し、lib直下のファイルにそのpathへのexport文を記載する。
void createWorkingFile({required String packageName}) {
  // srcディレクトリへファイルを作成
  final baseFileName = '$packageName.dart';
  File(path.join('lib/src', baseFileName)).createSync(recursive: true);

  // lib直下のファイルへexport文を記載
  final exportStatement = '''
export 'src/$baseFileName';
''';
  File(path.join('lib', baseFileName)).writeAsStringSync(exportStatement);
}
```
複数パスを組み合わせている箇所では[path](https://pub.dev/packages/path)パッケージを用いています。
Fileクラスにパスを指定して、`writeAsStringSync`にてファイルを作成 or （既に存在するlib直下ファイルは)上書きします。
尚、非同期的に扱う`writeAsString`メソッドもありますが、ここではその必要もなく同期的な方が扱いやすいので`writeAsStringSync`を用いています。

### lint設定
```dart:lib/run_command.dart
    // analysis_options.yamlを削除し、プロジェクトルートのものをsymbolic linkで追加
    final analysisOptionsFile = File('analysis_options.yaml')..deleteSync();
    Link(analysisOptionsFile.path)
        .createSync(path.join('../..', analysisOptionsFile.path));
```
先ほど作ったプロジェクト全体の`analysis_options.yaml`を用いるようにしたいので、そちらへのシンボリックを作成します。
尚、単に自身の`analysis_options.yaml`を削除するだけでも基本的にはトップディレクトリのものを参照してくれますが、時々参照されないバグ的な挙動が発生するようなのでシンボリックリンクを作っておいた方がベターかなと思います。

### testファイルの上書き
```dart: lib/onverwrite_test_file.dart
import 'dart:io';

import 'package:path/path.dart' as path;

/// テスト用ファイルを上書き作成するための関数
///
/// プロジェクト作成段階のテストファイルに含まれているサンプル用のクラスが
/// その後の処理により削除されているため、テスト内容を空にした状態に修正する。
void overwriteTestFile({required String packageName}) {
  final content = '''
import 'package:flutter_test/flutter_test.dart';

void main() {
  test('$packageName test', () {});
}
''';

  File(path.join('test', '${packageName}_test.dart'))
      .writeAsStringSync(content);
}
```
テストファイル内には、
```dart: lib/run_command.dart
    // パッケージ用のプロジェクトを作成
    Process.runSync(
      'fvm',
      ['flutter', 'create', '-t', 'package', name],
    );
```
の直後にはlib直下のファイルに作られているサンプル用のクラス(`Caliculator`クラス)を用いたテストが書いてあるのですが、`createWorkingFile`によりこのクラスを削除してしまっているため、処理の中身を空にしておきます。

### pubspec.yamlの上書き
コメントにあるようにバージョン表記やフィールドの追加などを行っています。
```dart: lib/overwrite_pubspec_yaml_file.dart
import 'dart:io';

import 'package:pub_semver/pub_semver.dart';

/// pubspec.yamlファイルを上書き作成するための関数
///
/// プロジェクト作成段階から以下の箇所を修正：
/// - sdkバージョンはmelos.yaml同様に最新stableをキャレット記号にて記述
/// - flutterバージョンは削除
/// - flutter_lintsをpedantic_monoへ置き換え
/// - 不要なhomepageフィールドの削除
/// - 不要なコメントの削除
/// - 意図しない配信を避けるためpublish_toフィールドを追加
void overwritePubspecYamlFile({
  required String packageName,
  required String description,
}) {
  final dartVersion = _getDartCaretVersion();

  final content = '''
name: $packageName
description: $description
publish_to: 'none'
version: 0.0.1

environment:
  sdk: $dartVersion

dependencies:
  flutter:
    sdk: flutter

dev_dependencies:
  flutter_test:
    sdk: flutter
  pedantic_mono: any

flutter:
''';

  File('pubspec.yaml').writeAsStringSync(content);
}

/// pubspec.yamlに記載するdart sdkのバージョンを取得する関数
///
/// FVMのdart versionを取得し、そのバージョンをキャレット記号にて記述する。
/// 尚、pubspec.yamlファイル作成後に`melos bs`コマンドを実行しても同様の結果になるが、
/// コマンドの実行時間削減のためにこのように実装している。
String _getDartCaretVersion() {
  final ProcessResult versionResult;
  versionResult = Process.runSync('fvm', ['dart', '--version']);
  final versionOutput = versionResult.stdout as String;

  final versionMatch =
      RegExp(r'Dart SDK version: (\d+\.\d+\.\d+)').firstMatch(versionOutput);
  final versionString = versionMatch?.group(1);
  final version = Version.parse(versionString!);
  // melos.yamlの記述内容と揃えるために、patchバージョンは0としておく
  return '^${version.major}.${version.minor}.0';
}
```
:::message
正規表現について
かなり細かい話になりますが、「`_getDartCaretVersion`内の`versionString!`の部分で!(null check operator)を用いてるけど大丈夫そう？」と気になった方は読んでもらえればと思います。
ここではDartバージョンを動的に取得しようとしており、現在の環境(Flutterバージョン3.16.2)ではコマンド実行後に指定している文字列が取得出来るため問題ないものの、将来的なバージョンアップデートによって取得結果が変わることはあり得ることにはあり得るかなと思います(`Dart SDK version:`が`Dart SDK version is`という表記になるみたいなイメージです）。
なので、上記では「!」で済ませてしまっていますが、[こちら](https://github.com/masa-tokyo/flutter_toolkit/blob/794b5957edc21942046b629c924ceda91e34b1f4/scripts/bootstrap_package/lib/overwrite_pubspec_yaml_file.dart#L67)のようにnullの状態に気づけるようにしておく（&もっと言えば単体テストを毎度走らせてコマンド実行前に気づけるようにしておく）とベターかなと思います。
:::

### ライセンスファイルを作成
```dart: lib/run_command.dart
    // LICENSEファイル削除し、プロジェクトルートのものをsymbolic linkで追加
    final licenseFile = File('LICENSE')..deleteSync();
    Link(licenseFile.path).createSync(path.join('../..', licenseFile.path));

``` 
プライベートなリポジトリであればこのファイルは削除すれば良いですが、公にするものであればライセンスも書いておいた方が良さそうです。
プロジェクトのルートに以下のようなライセンスファイルを作成し、それを全てのパッケージで参照するようにしておきます。
```file: LICENSE
MIT License

Copyright (c) 2023 Masaki Sato

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.
```


-----

# 作成したパッケージの使い方
作成したパッケージはリポジトリ内とリポジトリ外から参照することが出来ます。

## リポジトリ内からの参照
pubspec.yamlにパッケージ名を追記し、`melos bs`を実行します。

```yaml: pubspec.yaml
dependencies:
  {パッケージ名}:
```
すると、参照先を示す`pubspec_overrides.yaml`というファイルが自動生成されます。  

```yaml: pubspec_overrides.yaml
# melos_managed_dependency_overrides: {パッケージ名}
dependency_overrides:
  {パッケージ名}:
    path: ../../packages/{パッケージ名}
```

実装例では、`examples`というディレクトリ内にサンプルアプリを用意しており、その中で実際に作成したパッケージを利用しています。
https://github.com/masa-tokyo/flutter_toolkit/tree/main/examples/mobile


## リポジトリ外からの参照

`flutter_toolkit`リポジトリ外からパッケージを参照したい場合は、以下のように`pubspec.yaml`に追加します。

```yaml: pubspec.yaml
dependencies:
  {パッケージ名}:
    git:
      url: git://github.com/masa-tokyo/flutter_toolkit.git
      path: packages/{パッケージ名}
      # 特定コミットに対するタグを指定
      ref: {パッケージ名}/v1.0.0
```
`ref`の記述は無くても参照可能ですが、元のパッケージの変更（破壊的変更など想定外の事態が起こりうる）が常時反映されてしまわないようにタグ管理した方がベターかなと思います。　　

実装例は、こちらに作った別リポジトリをご覧ください。
https://github.com/masa-tokyo/flutter_toolkit_example


外部リポジトリから参照するタグの作成には、リリースしたい時点のコミット上で以下のコマンドを実行します。
```shell
git tag -a {パッケージ名}/v1.0.0 -m 'release {パッケージ名}/v1.0.0'
git push origin {パッケージ名}/v1.0.0
```

Gitタグやリリースの詳細についてはこちらの記事が参考になります。
https://qiita.com/tommy_aka_jps/items/5b39e4b27364c759aa53

ちなみに、[flutterfire](https://github.com/firebase/flutterfire/releases)や[riverpod](https://github.com/rrousselGit/riverpod/releases)などの主要なパッケージを見てみると、あまりリリースドキュメントまでは書いていないようでした。マルチパッケージ構成のリポジトリにおいては、各パッケージ内のCHANGELOGの内容のみでリリース情報は管理する、という方針が良いのかもしれません。　　

# 最後に
ここまでお読みいただきありがとうございました。
今回ご紹介した話はチームでも個人でも使えるものだと思うので、少しでも参考になりましたら幸いです。

最後に少しだけ告知をさせてください...！
FlutterGakkaiという勉強会（[途中で貼っていたリンク](https://www.youtube.com/live/IpcFqABBKvU?si=BoaeJt8J2tD8GazI&t=4241)は過去回のものです）を2024年1月末にオンライン/オフライン同時開催します。
今回もとても素敵な方々に発表いただくため、ぜひ来ていただけると嬉しいです🙌
https://fluttergakkai.connpass.com/event/304163/