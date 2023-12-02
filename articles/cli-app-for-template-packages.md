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
```
flutter create <プロジェクト名>
```
でプロジェクトを作成します。  
今回のプロジェクトはflutter関連の色んなものを詰め込む用なので、`flutter_toolkit`としました。

このプロジェクトでは[fvm](https://fvm.app/)でバージョン管理を行うので、以下コマンドでstableバージョンを用いるようにします。
```
fvm use stable
``` 

その後、[melos](https://pub.dev/packages/melos)を用いたマルチパッケージ構成にしていきます。
melosについての詳細はこちらの記事が参考になります。
https://zenn.dev/altiveinc/articles/melos-for-multiple-packages-dart-projects

melos導入が初めての場合は以下でインストールし、
```
dart pub global activate melos
```

pubspec.yamlにも追加します。
```
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

