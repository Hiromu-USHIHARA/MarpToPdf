# 【Github Actions】Marpのスライドを自動でPDF化する【ステップバイステップ】

> この解説は[Zennでも公開](https://zenn.dev/hiromu_ushihara/articles/daf78b96ccb003)しています．

## はじめに

[Marp](https://marp.app/)はMarkdownでスライドを作るツールです．
例えば，
```md
# すごい見出し

## とてもすごい見出し

- すごい箇条書き
- とてもすごい箇条書き
```
を変換して，
![](https://storage.googleapis.com/zenn-user-upload/03ff4084d48e-20250628.png)
のようなスライドを生成することができます．

Markdownでスライドを作成できるという手軽さや文法のシンプルさが魅力です．
例えば，Markdownで作成した文書をベースに簡単なスライドを作成したいときや，普段Beamerを使っている人がBeamerを使うほどではない（がパワポは使いたくない/持っていない）というときに便利です^["Google Slideを使え"というツッコミはお控えください．~~Beamerを使うような人は（私も含め）GUIでスライドを作り慣れていません~~]．

しかし，（これはBeamerにも共通する点ですが，）Markdownで記述されたファイルをスライド（HTMLまたはPDF）に変換するという一手間が必要になります．
この記事では，GithubにプッシュしたファイルからスライドのPDFを作成するプロセスを自動化する方法について紹介します．
Marpの使用方法からGithub Actionsを用いた自動化まで，ステップバイステップに解説しますので，説明どおりに進めることで，初心者でもMarpによるスライド作成ができるようになるはずです．
記事内で作成したファイルは[Githubのレポジトリ](https://github.com/Hiromu-USHIHARA/MarpToPdf.git)で公開していますので，参考にしてください^[このリポジトリをクローンして，Markdownファイルを編集すればそのままスライド作成に利用することもできます．]．

:::message
準備
この記事では`npm`と`npx`を使用します．
```bash
which npm npx
```
でコマンドが使えることを確認してください（インストールされていない場合は，これらをインストールした上で記事を読み進めてください）．
:::

## Marp入門

まずは，実際にスライドを作成することから始めます．
ディレクトリを作成して，MarpのCLIをインストールします．

```bash
mkdir MarpToPdf
cd MarpToPdf
npm install --save-dev @marp-team/marp-cli
```

`MarpToPdf`ディレクトリに，`node_modules/`，`package.json`，`package-lock.json`が作成されたはずです．

`src/md/sample.md`ファイルを作成して，簡単なスライドのコードを記述します^[ほとんど説明は不要だと思いますが，一点だけ．`paginate: true`はページ番号を付与するための設定です．]．

```md: src/md/sample.md
---
marp: true
html: true
paginate: true
math: katex
---

<!--
_class: title 
header: header
footer: footer
-->

# Marpサンプル

名前
（所属）
発表日

---

# すごい見出し

## とてもすごい見出し

- すごい箇条書き
- とてもすごい箇条書き
```

このファイルは以下のコードのいずれかを実行することで，スライドとして表示することができます:

```bash
npx marp src/md/sample.md --preview
npx marp src/md/sample.md --html
npx marp src/md/sample.md --pdf
```

PDFとして次のような出力が得られれば成功です．

![](https://storage.googleapis.com/zenn-user-upload/6b2cd34d7a19-20250628.png)
![](https://storage.googleapis.com/zenn-user-upload/03ff4084d48e-20250628.png)

### テーマを変更する

Beamerやパワポには様々なテンプレートが存在します．
Marpでも公開されているCSSファイルを利用したり，自分でCSSを書くことでスタイルを変更することができます．
ここでは，Beamer風のテーマであるBeamを例に取って，テーマの変更方法を紹介します．

まず，[Beamのページ](https://rnd195.github.io/marp-community-themes/theme/beam.html)からCSSファイルをダウンロードして，`src/theme/beam.css`を作成します．

:::message
日本語利用時の注意
Beamで使われているフォントは日本語に対応していないため，ダウンロードした`beam.css`をそのまま使うと文字化けが発生します．
文字化けを回避するためには，
```diff yml: src/theme/beam.css
/* @theme beam */
/* Author: rnd195 https://github.com/rnd195/ */
/* beam license - GNU GPLv3 https://github.com/rnd195/my-marp-themes/blob/live/licenses/LICENSE_beam */
/* License of beamer which inspired this theme - GNU GPLv2 https://github.com/rnd195/my-marp-themes/blob/live/licenses/LICENSE_GPLv2 */

+ @import url('https://fonts.googleapis.com/css2?family=Noto+Sans+JP:wght@400;700&display=swap');
@import "default";

:root {
+  font-family: 'Noto Sans JP', 'Segoe UI', Helvetica, Arial, sans-serif;
-  font-family: "CMU Sans Serif", "Segoe UI", Helvetica, sans-serif;
  --main: #1f38c5;
  --secondary: #141414;
}
```
のようにして日本語フォントを導入してください．
これ以外の`font-family`という箇所についても全て同様です．
:::

テーマとして，`beam.css`が読み込まれるように，`.marprc.yml`というファイルを作成して設定を記述します:
```yml: .marprc.yml
lang: ja-JP
allowLocalFiles: true
theme: src/theme/beam.css
```

これらの設定の上で再度スライドを生成すると以下のように，あたかもBeamerを使用しているかのようなスライドが得られます．

![](https://storage.googleapis.com/zenn-user-upload/ecd617014502-20250628.png)
![](https://storage.googleapis.com/zenn-user-upload/56a62ee3b234-20250628.png)

元のスタイルに戻したい場合は`theme`の行をコメントアウトすれば良いです．

## Github Actionsによる自動化

さて，以上がMarpを使ったスライド作成の概要ですが，毎回PDF生成を行うのは面倒なので，MarkdownファイルをGithubリポジトリにpushすると，

1. PDFを生成する
1. PDFをGithubリポジトリにpushする

というプロセスを自動で行うようにします．

自動化にはGithub Actionsを利用します．
`.github/workflow/marp.yml`にワークフローファイルを作成します．

まず，ワークフローの名前を設定します（任意の名前で構いません）．

```yml: .github/workflow/marp.yml
name: Convert all Markdown in src/md to PDF
```

実行タイミングは，`main`ブランチへのpush時と手動実行時とします．

```yml: .github/workflow/marp.yml
on:
  push:
    branches:
      - main
  workflow_dispatch:
```

コンテンツの書きこみとプルリクエストを許可します．

```yml: .github/workflow/marp.yml
permissions:
  contents: write
  pull-requests: write
```

具体的な実行内容は次の項目からなります:
- `name: Set up Node.js`
    - Node.jsのセットアップ
- `name: Install dependencies`
    - 依存環境のインストール
- `name: Create output directory`
    - PDFを保存するディレクトリの作成
- `name: Install Japanese fonts`
    - 日本語フォントのインストール
- `name: Convert all md to pdf`
    - MarkdownファイルからのPDF生成（および保存用ディレクトリへの移動）
- `name: List PDF files`
    - 保存されているPDFの一覧表示
- `name: Commit changes`
    - Githubリポジトリへのpush

```yml: .github/workflow/marp.yml
jobs:
  conversion-and-commit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '24'

      - name: Install dependencies
        run: npm ci

      - name: Create output directory
        run: mkdir -p output/pdf

      - name: Install Japanese fonts
        run: |
          sudo apt-get update
          sudo apt-get install -y fonts-noto-cjk

      - name: Convert all md to pdf
        run: |
          for file in src/md/*.md; do
            filename=$(basename "$file" .md)
            npx marp "$file" --pdf --verbose
            mv "src/md/${filename}.pdf" "output/pdf/${filename}.pdf"
          done

      - name: List PDF files
        run: ls -la output/pdf

      - name: Commit changes
        if: success()
        run: |
          git config --local user.email "action@github.com"
          git config --local user.name "GitHub Action"
          git add output/pdf/*.pdf
          git status
          git diff --quiet && git diff --staged --quiet || (git commit -m "Update pdf" && git push)
```

以上を記述した`.github/workflow/marp.yml`を含めたファイルをGithubにpushすると，自動でPDFを生成，保存する動作が確認できるはずです．

## おわりに

この記事では，初心者向けにMarpの導入方法からGithub Actionsを利用した自動PDF生成の方法までを解説しました．
カスタムテーマの設定や日本語フォントの埋め込みは失敗しがちなポイントだと思いますので，どなたかの参考になれば幸いです．
