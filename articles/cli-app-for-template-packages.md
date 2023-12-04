---
title: "自作CLIツールでテンプレート作成をしてみる"  
emoji: "💻"   
type: "tech"  
topics: ["Flutter", "Dart", "GitHub"]
published: false
---

# はじめに


# プロジェクト作成

まずはいつも通り、
```bash
flutter create <プロジェクト名>
```
でプロジェクトを作成します。  
今回のプロジェクトはflutter関連の色んなものを詰め込む用なので、`flutter_toolkit`としました。

このプロジェクトでは[fvm](https://fvm.app/)でバージョン管理を行うので、以下コマンドでstableバージョンを用いるようにします。
```bash
fvm use stable
``` 

その後、[melos](https://pub.dev/packages/melos)を用いたマルチパッケージ構成にしていきます。
melosについての詳細はこちらの記事が参考になります。
https://zenn.dev/altiveinc/articles/melos-for-multiple-packages-dart-projects

melos導入が初めての場合は以下でインストールし、
```bash
dart pub global activate melos
```

pubspec.yamlにも追加します。
```bash
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

```yaml
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
// todo テストに関しても記載するならテストコマンドも入れておく


また、このプロジェクト内では統一して`pedantic_mono`をlintとして使いため、`flutter_lints`から置き換えておきます。  

pubspec.yaml
```yaml
dev_dependencies:
  flutter_test:
    sdk: flutter
  pedantic_mono: any
```

analysis_options.yaml
```yaml
# https://pub.dev/packages/pedantic_mono
include: package:pedantic_mono/analysis_options.yaml

# 今後ここに独自のルールを追加していくため、このリポジトリ内は全てこのルールで統一したい
linter:
  rules:
    avoid_classes_with_only_static_members: false

```



```yaml

# パッケージテンプレートの作成

ここから本題のテンプレートの作成に入ります。  

一般にパッケージを作成する際には、
```bash
fvm flutter create -t package <パッケージ名>
```
を実行しますが、作成後にいくつか毎度修正したい箇所が出てきます。　　
そういった箇所を自動で修正するために、テンプレートを作成していきます。　　

// todo 補足：vs. mason

## ①テンプレート作成用のdartアプリを作成
ターミナル上で起動するdartのコンソールアプリを作成します。  
今回は`scripts`というディレクトリ配下に`bootstrap_package`という名前で作成します。

```bash
fvm dart create bootstrap_package
```

melos.yamlに`scripts`ディレクトリを追加しておきます。
```yaml
# 対象を指定
packages:
  - packages/**
  - examples/**
  - scripts/**
```

pubspec.yaml内に以下のようにパスを設定することでこのアプリのコマンド実行を端的に行えるようになります。
```yaml
dev_dependencies:
  bootstrap_package:
    path: scripts/bootstrap_package
```

// todo長くなりすぎるようだったらここまでは書かなくてもいいかも
pubspec.yamlに`pedantic_mono`を追加し、 先ほど作成したルートディレクトリの`analysis_options.yaml`へのシンボリックリンクを作成します。

```yaml
```bash
rm analysis_options.yaml && ln -s ../../analysis_options.yaml analysis_options.yaml
```
自身のプロジェクト内の`analysis_options.yaml`ファイルを削除するだけでも基本的にはルートのものを参照するようにはなるものの時々上手くいかないことがあるようなので、シンボリックリンクで参照するようにしておきます。

## ②処理の実行
Dartアプリではlibとは別にbinディレクトリがあり、main関数はこちらに置かれるのですが、今回は以下のようにして主要な処理をlibフォルダ内の`ruCommand`関数で行うようにします。

```dart
import 'package:bootstrap_package/run_command.dart';

void main(List<String> args) => runCommand(args);
```

// todo runCommand内でそれぞれの処理の概要を説明
// todo 各処理を説明（これをどこまでやるかを要検討）

-----

# 使い方
作成したパッケージはリポジトリ内とリポジトリ外から参照することが出来ます。

## リポジトリ内からの参照
// todo examplesディレクトリに関してはこの時点で説明した方が良いか検討。  
pubspec.yamlに追加して`melos bs`を実行します。

```yaml
dependencies:
  {パッケージ名}:
```
すると、`pubspec_overrides.yaml`というファイルが自動生成されます。  
具体例はこちらのexampleアプリをご覧ください。  
https://github.com/masa-tokyo/flutter_toolkit/tree/main/examples/mobile

// todo バージョン管理はタグをつけても出来ないのか確認 after tagging 1.1.0

## リポジトリ外からの参照

`flutter_toolkit`リポジトリ外からパッケージを参照したい場合は、以下のように`pubspec.yaml`に追加します。

```yaml
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
```bash
git tag -a {パッケージ名}/v1.0.0 -m 'release {パッケージ名}/v1.0.0'
git push origin {パッケージ名}/v1.0.0
```

// todo showcase the versioning by adding 1.1.0

Gitタグやリリースの詳細についてはこちらの記事が参考になります。
https://qiita.com/tommy_aka_jps/items/5b39e4b27364c759aa53

ちなみに、[flutterfire](https://github.com/firebase/flutterfire/releases)や[riverpod](https://github.com/rrousselGit/riverpod/releases)などの主要なパッケージを見てみると、あまりリリースドキュメントまでは書いていないようでした。マルチパッケージ構成のリポジトリにおいては、各パッケージ内のCHANGELOGの内容のみでリリース情報は管理する、という方針が良いのかもしれません。　　


