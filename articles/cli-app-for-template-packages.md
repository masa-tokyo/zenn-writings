---
title: "自作CLIツールでテンプレート作成をしてみる"  
emoji: "💻"   
type: "tech"  
topics: ["Flutter", "Dart", "GitHub"]
published: false
---

# はじめに

# きっかけ

とても勉強になったのでまとめておきたい。

docodoor紹介

個人でも作れるかな、という感じ。



# プロジェクト作成

まずはいつも通り、
```shell
flutter create <プロジェクト名>
```
でプロジェクトを作成します。  
今回のプロジェクトはflutter関連の色んなものを詰め込む用なので、`flutter_toolkit`としました。

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
├── examples // サンプルアプリ用ディレクトリ（無くてもok)
├── flutter_toolkit.iml
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
  - examples/**

# flutterバージョンはfvmを参照
sdkPath: '.fvm/flutter_sdk'

# `melos bs`実行時にpubspec.yaml上のdartバージョン表記を一括で設定
command:
  bootstrap:
    environment:
      sdk: ^3.2.0

```


また、このプロジェクト内では統一して`pedantic_mono`をlintとして使いため、`flutter_lints`から置き換えておきます。  

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

// todo 補足：vs. mason
// todo 補足 vs. シェルファイル or Process.runSync?

## ①テンプレート作成用のdartアプリを作成
ターミナル上で起動するdartのコンソールアプリを作成します。  
今回は`scripts`というディレクトリ配下に`bootstrap_package`という名前で作成します。

```shell
fvm dart create bootstrap_package
```

melos.yamlに`scripts`ディレクトリを追加しておきます。
```yaml: melos.yaml
# 対象を指定
packages:
  - packages/**
  - examples/**
  - scripts/**
```

pubspec.yaml内に以下のようにパスを設定することでこのアプリのコマンド実行を端的に行えるようになります。
```yaml: pubspec.yaml
dev_dependencies:
  bootstrap_package:
    path: scripts/bootstrap_package
```

## ②処理の実行
Dartアプリではlibとは別にbinディレクトリがあり、main関数はこちらに置かれるのですが、今回は以下のようにして主要な処理をlibフォルダ内の`ruCommand`関数で行うようにします。

```dart: bin/bootstrap_package.dart
import 'package:bootstrap_package/run_command.dart';

/// 新規パッケージ作成用の関数
///
/// FDSプロジェクトのルートディレクトリから以下のコマンドにより実行する。
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

    // ライセンスファイルを上書き
    overwriteLicenseFile();

    // READMEファイルをパッケージ名のみに上書き
    final packageTitle = '# $name';
    File('README.md').writeAsStringSync(packageTitle);
  } on FormatException catch (_) {
    // '-d'のようなoptionコマンドに続く引数が入力されていない場合
    showUsage(errorMessage: 'オプションコマンドの使い方が間違っています。');
  }
}
```
全て説明するととても長くなってしまうため要所要所解説していきます。もし不明点等ありましたら遠慮なくコメントいただけるとありがたいです🙏

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

// todo check dialog behavior
:::message
ダイアログ用パッケージ
余談ですが、さらに必要な情報が増えてきて複数引数を渡したくなってきた場合には[cli_dialog](https://pub.dev/packages/cli_dialog)や[interact](https://pub.dev/packages/interact)のようなパッケージを用いて対話形式のインターフェースにするとよりCLIっぽくなって体験が良さそうです。
ちょっと試したみた感じ僕の環境だと文字化けしてしまったりしてあまり上手く動かなかったのですが、もし使っていらっしゃる方がいらしたらぜひシェアお願いします🙏
:::

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

### ライセンスファイルの上書き


-----

# 使い方
作成したパッケージはリポジトリ内とリポジトリ外から参照することが出来ます。

## リポジトリ内からの参照
// todo examplesディレクトリに関してはこの時点で説明した方が良いか検討。  
pubspec.yamlに追加して`melos bs`を実行します。

```yaml: pubspec.yaml
dependencies:
  {パッケージ名}:
```
すると、`pubspec_overrides.yaml`というファイルが自動生成されます。  
具体例はこちらのexampleアプリをご覧ください。  
https://github.com/masa-tokyo/flutter_toolkit/tree/main/examples/mobile

// todo バージョン管理はタグをつけても出来ないのか確認 after tagging 1.1.0

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

具体例は、こちらに作った別リポジトリをご覧ください。
https://github.com/masa-tokyo/flutter_toolkit_example


外部リポジトリから参照するタグの作成には、リリースしたい時点のコミット上で以下のコマンドを実行します。
```shell
git tag -a {パッケージ名}/v1.0.0 -m 'release {パッケージ名}/v1.0.0'
git push origin {パッケージ名}/v1.0.0
```

// todo showcase the versioning by adding 1.1.0

Gitタグやリリースの詳細についてはこちらの記事が参考になります。
https://qiita.com/tommy_aka_jps/items/5b39e4b27364c759aa53

ちなみに、[flutterfire](https://github.com/firebase/flutterfire/releases)や[riverpod](https://github.com/rrousselGit/riverpod/releases)などの主要なパッケージを見てみると、あまりリリースドキュメントまでは書いていないようでした。マルチパッケージ構成のリポジトリにおいては、各パッケージ内のCHANGELOGの内容のみでリリース情報は管理する、という方針が良いのかもしれません。　　


