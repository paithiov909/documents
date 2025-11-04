---
title: "ggplot2の描画を高速化したかった話"
emoji: "🛻"
type: "idea"
topics: ["r", "ggplot", "ggplot2"]
published: true
---

## ggplot2の描画は遅い？

ここしばらくのあいだ、R言語で画像を出力してちょっとしたアニメーションをつくることに取り組んでいます。興味がある人は、次の記事などを読んでみてください。

- [R言語にシェーダー芸を求めるのは間違っているだろうか](https://lyrikuso.netlify.app/beyond-the-heliosphere/)
- [ggplot2で3Dプロットする](https://lyrikuso.netlify.app/plot3d-in-ggplot2/)

さて、これらの記事では「ggplot2で3Dプロットする」などと書いていますが、私個人としては、Rで画像をつくるときにggplot2はあまり使わないようにしています。ggplot2は、慣れると直観的に書けて便利なのですが、アニメーションをつくるために大量の画像を出力しなければならないといったような場面では、一枚一枚の画像をレンダリングするのに時間がかかりすぎて、使用するのにストレスがたまりがちだからです。

今のところ、私が考えている「Rでアニメーションをつくる」ためのベストな方法は、

1. ggplot2は使わない（フレームの描画にはskiagdを使うか、gridなどを直接使う）
2. フレームにする連番画像の書き出しを`purrr::in_parallel`で並列化する
3. gifskiを使うか、ffmpegを直接使うかして動画にする

といった感じです。

一方で、そうはいっても、やはりggplot2を使いたい場面もあります。とくに、ggfxを使う場合、ggfxのフィルターはgrobを与えたとき以外は常にggplot2関連のオブジェクトを返すので、積極的にggplot2を使いたくてというより、必然的にggplot2を使うことから逃れられなくなるという感じはあります。


## `theme(...)`を変数に格納するのは効果アリ

そこで、ggplot2の描画処理を高速化するために、いくつかのことを試しました。結論を先にまとめておくと、この下で書いている`base::trace()`を使ったトリックは効果がなさそうでした。

試したかぎりでまともに効果がありそうだったのは、`theme(...)`などの、プロットをつくるのに共通する部分の結果をあらかじめ変数に格納しておいて、使いまわすという小ワザです。これは、古くは次の2017年ごろのR-bloggersの記事などで紹介されていたTipsで、現在のggplot2でも有効なようです。

- [Accelerating ggplot2: use a canvas to speed up rendering plots | R-bloggers](https://www.r-bloggers.com/2017/07/accelerating-ggplot2-use-a-canvas-to-speed-up-rendering-plots/)

これについて一応それっぽい説明をすると、まず前提として、ggplot2の「グラフ」としての描画は`ggplot2::ggplotGrob()`という関数の呼び出しとして、二段階にわたる変換を経て実行されています。

`ggplot2::ggplotGrob()`は、`ggplot2::ggplot`クラスのオブジェクト`p`を引数として取り、`ggplot2::ggplot_build(p) |> ggplot2::ggplot_gtable()`というような変換をする関数です。私たちがRのコンソールからggplot2のプロットを`print()`したときに実際に呼ばれている処理は、この`ggplot2::ggplotGrob()`にあたります。

`ggplot2::ggplot_build()`は、`ggplot2::ggplot`クラスのオブジェクトから実際にプロットをおこなうためのデータの変換をしたりする処理で、`ggplot2::ggplot_gtable()`は、そうして変換したデータを元にして実際にレイヤーをgrobに変換したり、それらのgrobを組み合わせて、私たちが目にする「グラフ」になるようにレイアウトしたりする処理です。前者は`ggplot2::ggplot_built`というクラスのオブジェクトを返し、後者は`gtable`クラスのオブジェクト（gTree）を返します。

これらの変換はプロットを表示するタイミングで実行されるものなので、基本的に避けて通ることはできません。一方で、その手前の`ggplot2::ggplot`クラスのオブジェクトや`ggplot2::theme`, `ggproto`という単位のオブジェクトは変数に格納できるので、これらを返す関数が複数回呼ばれる場合、結果を変数に保持しておくことで毎回呼び出されるのを避けることができるというわけです（それぞれの`ggproto`の処理は`ggplot2::ggplot_build()`が呼ばれたタイミングで走るっぽいので、「そうした処理が複数回にわたって呼ばれる」ことを避けられるわけではないです）。

とくに`theme_void() + theme(...)`みたいな部分については、`+`で結合した結果ごと変数に保持できるので、これをやる恩恵が大きいです。手もとでいろいろ時間を計ってみたところ、レイヤーが一つとかのシンプルなプロットの場合だと、`ggplot2::ggplot_build()`にかかる時間の1/3くらいが`ggplot2::theme`の結合だったみたいなことがあったので、すごく雑に言うと、「グラフ」としてのプロットにかかる時間が2/3くらいになる可能性があります。まあ、それは言い過ぎだとしても、少なくともまったく効果がないということはなさそうです。


## base::traceでggplot2の内部処理を並列化する

これとは違うアプローチとして「`ggplot2::ggplotGrob()`の処理そのものを高速化できないのか？」と思ってごちゃごちゃやってみました。ただ、先ほど言いましたが、これについては、私が試した範囲では効果がなかったです。

その方法として注目したのが、ggplot2の関数のなかで内部的に呼ばれている処理です。たとえば、`ggplot2::ggplotGrob()`で呼ばれる`ggplot_build()`のなかでは、次の`by_layer()`という関数が何回か呼ばれています。

```r
# Apply function to layer and matching data
by_layer <- function(f, layers, data, step = NULL) {
  ordinal <- label_ordinal()
  out <- vector("list", length(data))
  try_fetch(
    for (i in seq_along(data)) {
      out[[i]] <- f(l = layers[[i]], d = data[[i]])
    },
    error = function(cnd) {
      cli::cli_abort(c(
        "Problem while {step}.",
        "i" = "Error occurred in the {ordinal(i)} layer."),
        call = layers[[i]]$constructor,
        parent = cnd
      )
    }
  )
  out
}
```

アイデアとして、これを`mirai::mirai_map()`とかを使って並列化したらちょっと速くなるのではということを思いつきました。そこで、次のようなコードを書いてみました。

```r
library(ggplot2)

mirai::daemons(3)

trace(ggplot2::ggplot_build, quote({
  by_layer <- \(f, layers, data, step = NULL) {
    out <-
      mirai::mirai_map(seq_along(data), \(i) {
        f(layers[[i]], data[[i]])
      },
      f = f, layers = layers, data = data,
      )
    rlang::try_fetch(
      out[.stop],
      error = \(cnd) {
        ordinal <- label_ordinal() # nolint
        cli::cli_abort(
          c(
            "Problem while {step}.",
            "i" = "Error occurred in the {ordinal(cnd$location)} layer."
          ),
          call = layers[[cnd$location]]$constructor,
          parent = cnd
        )
      }
    )
  }
}), print = FALSE)
```

`base::trace()`なんて使っている人はいない気がするので軽く説明すると、これを実行すると、`ggplot2::ggplot_build()`が呼び出されたとき、その呼び出しの一番最初で、ここで書いた`by_layer`がローカルに定義されるようになります。ggplot2の内部の`by_layer`の呼び出しは、ほかのパッケージの名前空間からの呼び出しである`packageName::func`のようなかたちにはなっていないので、このように`base::trace()`を使うことで、ここで定義されたローカルな`by_layer`によってggplot2パッケージの名前空間内にある`by_layer`をマスクしてしまうことができます。

ちなみに、traceはそのセッション中ずっと有効になるので、やめるには`untrace(ggplot2::ggplot_build)`などとします。

これについては、効果がまったくないというか、まああるのかもしれないんですけど、少なくとも私が試すかぎりではそんなにレイヤーをたくさん重ねたグラフは描かなかったので、効果がないように見えたのかもしれません。とはいえ、ふつうにggplot2を使っていて、そんなにレイヤーをめちゃくちゃたくさん重ねたグラフをつくることはむしろ稀だろうと思うので、いずれにせよ、これを試す価値はそんなになさそうです。


## むすび

今回いろいろ試してみたなかで、正直あらためて記事にしたいほどの発見はなかったのですが、`base::trace()`を使っているコード例は日本語で検索しても出てこなくなっている感じがしたので、やったことをごく簡単にまとめてみました。

ところで、Rパッケージ開発をしていると`packageName::func`みたいな書き方をすると何が嬉しいんですか？（roxygenタグで`@import`するのと何が違うのか？）みたいな疑問にぶち当たることがあると思います。

これはぶっちゃけその人の趣味によって好きな書き方をすればいい部分だとは思うのですが、実は、名前空間を`@import`するやり方だと、上でやったのと同じ方法でユーザーが呼び出し時に`func`をマスクすることができる一方、`packageName::func`という呼び方だと、基本的にユーザー側がこれをハックする手段はなく、問答無用で`packageName::func`が呼び出されるという違いがあります（少なくとも、私の理解では）。

記事の趣旨からは外れますが、`base::trace()`は古来からあるR言語のオモシロ関数の一つだと思っているので、知らなかったという人は、こういうことができるというのをぜひ覚えて帰ってください！
