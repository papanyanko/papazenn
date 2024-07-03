---
title: "【初期費用ゼロ】Notion + Fruition + Cloudflareで始めるブログ【独自ドメイン】"
emoji: "🆓"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["notion", "cloudflare"]
published: true
---

[CAREER SKILLS](https://www.amazon.co.jp/CAREER-SKILLS-%E3%82%BD%E3%83%95%E3%83%88%E3%82%A6%E3%82%A7%E3%82%A2%E9%96%8B%E7%99%BA%E8%80%85%E3%81%AE%E5%AE%8C%E5%85%A8%E3%82%AD%E3%83%A3%E3%83%AA%E3%82%A2%E3%82%AC%E3%82%A4%E3%83%89-%E3%82%B8%E3%83%A7%E3%83%B3%E3%83%BB%E3%82%BD%E3%83%B3%E3%83%A1%E3%82%BA-ebook/dp/B07FCYSNXT/ref=sr_1_1?crid=CH9JK78B567U&dib=eyJ2IjoiMSJ9.NCIHrEHKMzdF0IKnhrbpgTIFD95CDQqXm8Dw6bt_z6KjEpCbTmwvKUPUdq9s2Nl-G34tXNPCCxicx-nu4QAJmDQJTvEvzh5J-X2kueBjsy9HKkzJX1TRIZ7GVv8GIkQmBpOU9sE79QLU1B5Ium_skeDVKJnp7KtZsvf1q7aR8ur2a-DAWnq2QqoshmWJFHira0KOxArmyJE0Li22_b9cRzxEzAzGc3X-rydhnteDxSBSoIJNroaBsvdTvLobOO7vI24gpJGxAAHhVniFvEKjl07_ArjzuBJhqpGWP_dGpxY.LcudNJhUfTwaqon1OOhWLnzXpjYUnOoepkuAMBPLsUs&dib_tag=se&keywords=career+skills&qid=1720008398&sprefix=career+ski%2Caps%2C193&sr=8-1) の影響でブログを始めた。

https://papanyanko.com/

物事をするときに努力するのではなく楽できる仕組みを考えたいのがエンジニアの性だ。
ブログを習慣化するためには執筆や投稿の負荷をできるだけ軽くするのが望ましい。

結論としてはNotionで記事を書いてFruition + Cloudflareで公開することにした。

## やりたいこと

ブログの趣旨と私の趣味で以下のような要件が挙がった。

- 書きたい記事の性質上、$\TeX$環境は必須
- 本を引用するのに$\TeX$を手打ちするのはめんどいのでなんとかしたい
- $\TeX$の書き方にはいくつか種類があるが、執筆中のプレビューと実際のブログの表示の差異をなくしたい
- 記事はMarkdownで書きたい
- できればMarkdownをgitで管理したい
- デザインやパフォーマンスの盆栽がしたいわけじゃない、SEOもめんどくさい

## Notion + Fruition + Cloudflare(ﾟ∀ﾟ)ｷﾀｺﾚ!!

Notionはデフォルトで$\TeX$が打てる。git管理はできないけどMarkdownも使える。プレビューと本番の表示の差異を気にする必要もない。

しかもヘッダー画像も自分で見つける必要がなく、Unsplashがシームレスに使えるのだ。

…あれ、これが答じゃね？

ただ独自ドメインで運用しようとした場合Notion siteは有料プランが必要で、年19800円かかるので費用面だけ何とかしたい。

そんなことを思ってちょっと調べてみたらいっぱい情報あるじゃん。

[Fruition を使って Notion を独自ドメインで公開する - Qiita](https://qiita.com/mizunokura/items/5a46d97dad397fc255df)

[FruitionのデメリットはCloudflareのセットアップが必要なこと](https://stephenou.notion.site/Fruition-Free-Open-Source-Toolkit-for-Building-Websites-with-Notion-771ef38657244c27b9389734a9cbff44)だといっているが、こちとらWebエンジニアだ。恐るるに足らん。

お名前ドットコムでドメインは初年度無料で取得できた（xserverドメインのほうが更新料含めると安かったので移管しよう）。Cloudflareもfreeプランでいい。

上のQiita記事などを参考にちょいちょいっと設定すれば…はいできた！

というわけでほぼ望み通りの環境が初期費用ゼロで構築できてしまいましたとさ。ブログは継続できるかどうか確実じゃないので初期費用がかからないのは嬉しい。

ちなみにconsoleになにやらAPI呼び出しのエラーが出ているが、簡易的な個人サイトなので目をつぶろう。。。

## 参考文献

すでにネット上にたくさん情報があるので今さら詳しいやり方を書かなくてもと思い、参考にした記事のリンクをまとめておく。

https://qiita.com/mizunokura/items/5a46d97dad397fc255df

https://zenn.dev/cbmrham/articles/58d250415663e7

https://qiita.com/iwaken71/items/6fb63a7ba500b59437e5

特筆することとしては、主に古い記事などでCloudflareでAレコードに `1.1.1.1` を設定すると書かれているが、そこは3つ目の記事を参考にしてnotion.soのIPアドレス `104.18.23.110` を設定する必要がある。

また、Fruitionのスクリプトはダークモードのボタンがおかしいらしく、ブラウザバックするとNotionのログイン画面に飛んでしまう。
それを避けるためにはworker.jsのaddDarkModeButtonを呼び出している行をコメントアウトする。

https://x.com/datsukan_tech/status/1776553496764088792

ページ右上に出るNotionアイコンの消し方はこちら。

https://qiita.com/kazukimuta/items/31f8da3381a587263903