---
title: "arb_translateパッケージでCLIからGemini APIを使ってみた"  
emoji: 🌎"   
type: "tech"  
topics: ["Flutter", "AI", "Gemini"]
published: true
---
## はじめに

多言語対応でarbファイルをいじるのって地味に手間で大変ですよね…

ということで、今回は[先日monoさんもおすすめされていた](https://x.com/_mono/status/1829048488967188978)、arb_translateというパッケージを使ってみました。

https://pub.dev/packages/arb_translate

この記事が同じ悩みを持っている方の参考になったら幸いです！

## 実際の挙動

`arb_translate` コマンドを実行することで、デフォルト言語から翻訳が走ってファイルの中身を自動生成してくれます。
既にファイル上にフィールドが存在する場合には、足りていないもののみ補ってくれます。

https://x.com/masaki_hideout/status/1841759256338710550


## 前提

ここでは、既に多言語化対応されたプロジェクトを前提に進めます。

多言語化対応設定については、以下の公式ドキュメントなどをご参照ください：

https://docs.flutter.dev/ui/accessibility-and-internationalization/internationalization

また、デフォルト言語ファイル(`L10n.yaml` の `template-arb-file` に指定されているもの）は日本語（`app_ja.arb`)になっていることとします。

L10.yamlファイルを用意しない場合の設定については、ドキュメントの以下の箇所をご覧ください：

https://pub.dev/packages/arb_translate#configuration-without-l10nyaml

実装詳細については、こちらのサンプルリポジトリも適宜ご参照ください：

https://github.com/masa-tokyo/arb_translate_sample

## 使い方

では、ここから使い方を手順を追ってみていきましょう。

### 1. インストール


まずは、パッケージを以下コマンドでインストールします：

```bash
dart pub global activate arb_translate
```

### 2. APIキー取得

本パッケージではOpenAIやGemini、その他カスタムモデルなども利用できるようですが、今回は無料枠のあるGeminiのAPIを使って進めていきます。

以下よりAPIキーを取得しましょう：

https://aistudio.google.com/app/apikey

GCP上の既存プロジェクト選択 or 必要なら新しくプロジェクト作成します：

![](/images/articles/arb_translate/google_ai_studio.png)

### 3. APIキーの設定

このAPIキーの扱いが悩ましいかと思いますが、指定の仕方としては以下の3通りがあります。

**選択肢１） `L10n.yaml` へ直指定**

以下のように `L10n.yaml` に直接指定することが出来ます：

```yaml
template-arb-file: app_ja.arb
output-class: L10n
nullable-getter: false
arb-translate-api-key: XXXXXXXXXXXXXXXX
```

但し、このファイル自体は通常Git管理しているはずであり、APIキー自体をコミットするのは原則NGなのであまり好ましい方法ではないかなと思います。

**選択肢２）環境変数設定**

`.zshrc` 等シェルの設定ファイルに以下のように指定することが出来ます：

```bash
export ARB_TRANSLATE_API_KEY="XXXXXXXXXXXXXXXX"
```

但し、複数のプロジェクトで `arb_translate` パッケージを利用していて別々のAPIキーを利用したいというケースは起こりそうです。

そういった場合は、以下のdirenvのようなツールでプロジェクト内に環境変数を設定する必要があるかなと思います。

https://qiita.com/kompiro/items/5fc46089247a56243a62

**選択肢３）コマンドライン上で指定**

以下のように、引数としてAPIキーを指定することが出来ます：

```bash
 arb_translate --api-key XXXXXXXXXXXXXXXX
```

都度指定するのは手間ですが、APIキーを含んだコマンドを定義しておくことでその手間を省くことが出来ます。

詳細は最後のおまけパートでお話します。

:::message
APIキーの取り扱いについて

Gemini APIを利用出来るパッケージとして他に [google_generative_ai](https://pub.dev/packages/google_generative_ai) がありますが、APIキーの取り扱い方という点については大きく異なります。こちらのパッケージはアプリ内で利用者が都度APIを利用する想定となっているためです。その場合、ストアへ公開するアプリ内にAPIキーを含む必要が出てきます。例えば、[flutter_dotenv](https://pub.dev/packages/flutter_dotenv) パッケージを利用して環境ごとに別々のキーを読み込むようにした場合、（.envファイル自体はGit管理から外したとしても）実際に配布されるアプリ内のアセットに文字列として含める必要が出てきてしまいます。この意味で、google_generative_ai パッケージはプロトタイプ利用のみにするよう推奨されています。

一方で、今回のarb_translateは開発時のみ作業者がAPIを利用すれば良いため、商用利用のアプリであっても問題なく利用が可能です。
:::


### 4. モデル選択

以下のように、コマンドライン or L10n.yaml から指定出来ます。

**選択肢１）コマンドライン上で指定**

```bash
arb_translate --model gemini-1.5-flash
```

**選択肢２） `L10n.yaml` で指定**

```yaml
template-arb-file: app_ja.arb
output-class: L10n
nullable-getter: false
arb-translate-model: gemini-1.5-flash
```

Gemini API では、gemini-1.5-flash, gemini-1.0-pro (デフォルト), gemini-1.5-pro があり、どのモデルも無料で利用が可能です。

利用上限に関しては、分単位と日単位の制限があるようですが、最上位モデルの `gemini 1.5-pro` であっても、かなりの量利用出来そうな印象を受けました。

一日に何十回も大量の言語ファイルを翻訳するような状況でない限りは最上位モデルであっても良いのかなと思います。

gemini-1.5proで日本語→英語への翻訳を2回ほど実行した時の様子：

![](/images/articles/arb_translate/gcp.png)


## 5. contextの付与

翻訳時の精度を向上させるために、踏まえてほしい追加の文脈を渡すことが出来ます。

指定方法に関しては、モデル指定の場合と同様にコマンドライン or L10n.yaml から可能です。

**選択肢１）コマンドライン上で指定**

```bash
arb_translate --context {アプリ情報や翻訳の方針などの追加文脈}
```

**選択肢２） `L10n.yaml` で指定**

```yaml
template-arb-file: app_ja.arb
output-class: L10n
nullable-getter: false
arb-translate-context: {アプリ情報や翻訳の方針などの追加文脈}
```

例えば、元々「日本酒」→ 「Japanese rice wine」と翻訳されていた場合に「日本語をそのまま訳してほしい」というcontextを渡すと、「Sake」と翻訳してくれるようになったりします。

## おまけ - 実行コマンド作成

今回の[サンプルリポジトリ](https://github.com/masa-tokyo/arb_translate_sample)では、[grinder](https://pub.dev/packages/grinder) パッケージを利用して専用コマンドを定義しています。

こちらを利用することで、上記で登場したAPIキーやモデル指定などを一括で指定出来ます。
環境構築用コマンドなど、よく使うコマンド諸々も定義出来るためとても便利です。
定義したコマンドは、`grind XXX`のように使うことが出来ます。

::: details tools/grind.dart
```dart
import 'dart:io';

import 'package:grinder/grinder.dart';

main(args) => grind(args);

/// Installs the `arb_translate` package globally.
@Task()
setup() {
  run(
    'dart',
    arguments: ['pub', 'global', 'activate', 'arb_translate'],
  );
}

/// Runs the `arb_translate` command to generate the `.arb` files.
///
/// You can run this command with `grind`.
///
/// Use this command after:
/// - installing the package with [setup]
/// - copying `.secret.example` and creating `.secret` file for the API key
///
/// To improve the quality of translation, update [_translateContext].
@DefaultTask()
translate() {
  try {
    final envEntries = File('.secret').readAsLinesSync();
    final apiKey = envEntries
            .firstWhere((e) => e.startsWith('ARB_TRANSLATE_API_KEY='))
            .split('=')[1];

    // Context to improve translation quality.
    const context = '''
Japanese words should be translated into English directly.
For example, おにぎり should be Onigiri.
''';

    // choose the highest model, which is expected to be within the free tier
    const model = 'gemini-1.5-pro';

    run('arb_translate', arguments: [
      '--api-key',
      apiKey,
      '--context',
      context,
      '--model',
      model,
    ]);
  } on PathNotFoundException catch (_) {
    exitCode = 1;
    stderr.writeln(
            'PathNotFoundException: `.secret` file is not found. Please create it by copying `.secret.example`.');
  } on StateError catch (_) {
    exitCode = 1;
    stderr.writeln(
            'StateError: `.secret` file does not contain `ARB_TRANSLATE_API_KEY`.');
  }
}

```
:::

今回は、APIキーを `.secret` ファイルに記載して、以下のように取得しています。
こういった処理は`Makefile`のような利用しても出来るとは思いますが、grindパッケージを利用すればDartで書けて少し複雑な処理やエラーハンドリングがしやすいという点が良いのかなと思っています。
```dart
    final envEntries = File('.secret').readAsLinesSync();
    final apiKey = envEntries
        .firstWhere((e) => e.startsWith('ARB_TRANSLATE_API_KEY='))
        .split('=')[1];
```

ちなみに、APIキーを参照する設定ファイル自体は `.gitignore` に入れていますが、`.secret.example` のようなものをgit管理するようにすると、開発者個々人が手元で作る際に親切かなと思います。

```file: .secret.example
# 実際の値はこちらを参照：https://www.XXXXXX.com
ARB_TRANSLATE_API_KEY=YOUR_API_KEY
```

以下の箇所で、コマンド実行をしています：

```dart
    const context = '''
Japanese words should be translated into English directly.
For example, おにぎり should be Onigiri.
''';

    const model = 'gemini-1.5-pro';

    run('arb_translate', arguments: [
      '--api-key',
      apiKey,
      '--context',
      context,
      '--model',
      model,
    ]);
```

`run`はgrinderパッケージが用意しているメソッドで、内部的に`Process.runSync`を利用してコマンド実行をしています。
CLIツールについて興味ある方は以前書いたこちらの記事もぜひご覧ください：

https://zenn.dev/masa_tokyo/articles/cli-app-for-package-template

## おわりに

これまでは、IDE内でGitHub Copilotに提案してもらっていましたが、arb_translateを利用することで、以下のようなメリットがあるのかなと感じました：

- コマンド1回で全て完結する
    - Copilotを使っていた際は、1行1行の提案を待って生成していたのでかなり手間が省けるようになったと思います。
- フィールドの設定忘れの心配が無い
    - もちろん最後には開発者本人が確認する必要があるものの、デフォルト言語に足りていないものを全て生成してくれる仕様なので「あれどれが足りて無いんだっけ」という手間を省けます。
- ファイル内のフィールドの並び順までデフォルト言語に揃えてくれる
    - 並び順がぐちゃぐちゃになることで抜け漏れが発生する要因になっていたので、とてもありがたいです。
- contextが渡せることで翻訳精度の向上が見込める
    - モデル自体の賢さも関係しますが、翻訳の方針をカスタマイズ出来るのは嬉しいですね。

お読みいただきありがとうございました！

何か間違っている点や実際使ってみて分かった点などあったらコメントいただけると幸いです🙌