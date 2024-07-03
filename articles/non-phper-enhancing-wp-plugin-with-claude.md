---
title: "Claude 3.5 SonnetでのんぺちぱーがWordPressプラグインを改造してKaTeX環境を構築した話"
emoji: "💪"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["生成ai", "php"]
published: true
---

前回は私のブログの本番運用にこぎつけた技術たちを紹介した。

https://zenn.dev/papanyanko/articles/free-blog-with-notion

しかしそこに至るまでには~~迷走~~探索の期間があった。その過程で得られたものがいつかどこかの誰かの参考になることを願ってネットの海に放流して供養する。

## 前提

やりたかったことを改めて書いておく。

- $\TeX$が表示できる
- Markdownで書ける
- Markdownはgit管理できる
- GitHubにpushしたら自動で記事が公開される

これらをWordPressで実現した。今回は$\TeX$まわりの話をする。

個人的な結論としては生成AIがすごく便利だということを認識した。一方でちゃんと期待通りに動く成果物が一発では出てこないことも多いので適切にレビューする能力は必要だと感じた。

## $\KaTeX$の読み込み

まずは$\TeX$を適切に表示できるように$\KaTeX$をCDNから読み込むスクリプトを書く。MathJaxよりも表示が速いので$\KaTeX$を選んだ。

[これ](https://www.t3nro.net/posts/katex/)を参考にするとAuto-render Extentionを使うのが楽そう。DOMツリーを再帰的にみていって$\TeX$を含むテキストを探してレンダリングしてくれる。APIはrenderMatnInElementのみだ。

`function renderMathInElement(elem, options)`

第1引数のHTML要素の中のテキストノードに含まれる数式をレンダリングする。その際、`katex.render`と同様のoptionを渡すことができる。ここでdelimiterやmacroを設定することができるのでVSCodeなど手元の環境との差異があった場合は解消する。

テーマはCocoonを使うのでCocoon Child: tmp-user/head-insert.phpに以下のようなコードを書く。他のテーマでもheadタグに同じコードが書ければOK。

```html
<link rel="stylesheet" href="<https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/katex.min.css>" integrity="sha384-wcIxkf4k558AjM3Yz3BBFQUbk/zgIYC2R0QpeeYb+TwlBVMrlgLqwRjRtGZiK7ww" crossorigin="anonymous">
<script defer src="<https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/katex.min.js>" integrity="sha384-hIoBPJpTUs74ddyc4bFZSM1TVlQDA60VBbJS0oA934VSz82sBx1X7kSx2ATBDIyd" crossorigin="anonymous"></script>
<script defer src="<https://cdn.jsdelivr.net/npm/katex@0.16.10/dist/contrib/auto-render.min.js>" integrity="sha384-43gviWU0YVjaDtb/GhzOouOXtZMP/7XUzwPTstBeZFe/+rCMvRwr4yROQP43s0Xk" crossorigin="anonymous"></script>
<script>
    document.addEventListener("DOMContentLoaded", function() {
        renderMathInElement(document.body, {
          delimiters: [
              {left: '$$', right: '$$', display: true},
              {left: '$', right: '$', display: false},
          ],
          throwOnError : false
        });
    });
</script>

```

「簡単！WordPressにLaTeXを導入する方法！」みたいな記事を読んでも、ほとんどでプラグインを使うか使わないかの一方しか書かれていなかった。例外は[これ](https://www.kennzo.net/wordpress-latex)くらい。

## 運命の動作確認

$\KaTeX$を読み込むスクリプトが書けたので試しにWordPressのエディタで記事を作って動作確認する。これが導入前。

![KaTeX導入前](/images/before_katex.png)

スクリプトを追加し、pushする。

![KaTeX導入後](/images/after_katex.png)

ちゃんと反映された。と思ったら特大落とし穴があった。

Markdownで改行を含む式を書いてWordPress REST APIでPOSTすると...

![displayスタイルの数式がパースできてない](/images/before_multiline_display_math.png)

なんじゃこりゃあああああ！！！

どうやら改行のところでpタグやbrタグが作られてしまい、$\TeX$コードだと認識できなくなってしまうようだ。

改行するのをやめればいいのだが、それでは快適に数式を書けないためブログが続かなくなってしまう。

ここの部分は今後もブログの根幹に関わる仕組みなので多少こだわってもよかろう（**沼の入り口**）。

WordPressにMarkdownをパースさせるとすると何かしらコードを書いて対処するしかない。他には手元でMarkdownからHTMLを生成しておいてそれを配信するという手もあるが、デザインなどの面で劣ってしまう。また、MarkdownからHTMLを生成するVSCodeの拡張機能も日本語をうまく扱えないので別の問題が生じてしまう。

## というわけで、Markdownパーサーに手を加えることにした。

[parsedown](https://github.com/erusev/parsedown)が速くて評判が良さそうだったが、開発が止まっている様子。PRを出しても取り入れられる望みは薄いのでforkすることにした。メンテが活発でないプラグインを使い、あまつさえオレオレ改造を加えるという負債...ぐぬぬ...。

唇を噛みつつClaudeにプロンプトを投げる。PHPを読んだことも書いたこともない私にもわかるよう親切に修正箇所を教えてくれた。

初見では何をやっているのかさっぱりわからないコードも、ファイルをアップロードすれば瞬時に読み解いてくれる。コードは書く時間より読む時間が長いというが、**他人の書いたコードを理解する時間を圧倒的に短くしてくれる**という点ではとてつもない効果だ。

さらにやりたいことを伝えれば修正案を出してくれる。それで理解できない部分は「ここでは何をしていますか」と質問していけばあっという間にコード全体が何をやっているのかまでつかめてしまう。

しかし実際に修正案を適用してサンプル記事を表示させてみたがうまくいかない。

とりあえずエラーメッセージを再びClaudeに投げる。「申し訳ございません。」といって修正したコードを出力する。

やりとりを続けること数回、ここまでくるとこちらもコードの動きがだいぶ把握できているので「ここがおかしいんじゃない？」と指摘できる。おかしい箇所を適切に指摘してあげるレビュー能力はまだ人間に求められている。

よさそうなコードができあがったのでzipに固めたのをアップロードして適用してみる...

![displayスタイル対応後](/images/after_multiline_display_math.png)

やったああああああああ！！！

未経験の言語で他人が書いた初見のコードを修正してやりたいことを実現できてしまった。作業時間も夜の数時間×数日程度だ。こりゃあすげえや。

リポジトリはこちら。

https://github.com/papanyanko/my-parsedown

というわけでWordPressに$\KaTeX$環境を構築することができた。

結局使わなくなってしまったけど、面白かったからヨシ！👈🐱
