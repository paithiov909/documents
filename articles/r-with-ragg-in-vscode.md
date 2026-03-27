---
title: "R Extension for VS Codeでraggでプレビューしたい"
emoji: "📈"
type: "tech"
topics: ["r", "ggplot", "ggplot2"]
published: true
---

## 問題

[R Extension for Visual Studio Code](https://github.com/REditorSupport/vscode-R)の[Plot viewer](https://github.com/REditorSupport/vscode-R/wiki/Plot-viewer)では、httpgdを使わない場合、pngデバイスが使われるらしい。

SVGになって困らないケースではhttpgdを使っておけばよさそうなのだが、httpgdだと、Remote SSH越しに使っているときに意図した文字（絵文字）が表示されないことがあった。[このissue](https://github.com/nx10/httpgd/issues/138#issuecomment-1636582482)に「Fonts are not embedded in the SVG so they have to be installed both on the server as well as locally」と書かれているように、ローカルにないフォントを指定した場合、文字が見つからなくて豆腐になるっぽい。

具体的にはggplot2の図のなかで絵文字を使いたかったので、たぶんフォントのフォールバックとかをよしなにしてもらいつつ、AGG（ragg）を使って描いた図をPlot viewerのなかでプレビューしたかった。しかし、現状のvscode-Rだとグラフィックデバイスを差し替える方法がよくわからない（そもそもそんなことできるのかが謎）。


## 解決策1

書いた当人もよくわからないコードになったが、下のようなコードを`.Rprofile`とかに書いておいて実行すると、やりたかったようなことが実現できた。なお、これはuse.httpgdがtrueになっている前提のコード。また、ggplot2のグラフ以外に対しては何もしない。

```r
if (interactive() && Sys.getenv("TERM_PROGRAM") == "vscode") {
  #' Capture ploting with ggplot on AGG into UNIGD device
  #' @param ... other parameters
  #' @param .res resolution
  #' @param .scaling scaling
  print.gg <- function(..., .res = 72, .scaling = 1, .force = FALSE) {
    rlang::check_installed(c("ggplot2", "ragg", "showtext"))

    size <- dev.size(units = "in")
    message(
      "Capturing plot from AGG \nwhere original device size is: ",
      paste(signif(size), collapse = " x "),
      " (in)."
    )
    p <- ggplot2::last_plot()
    cap <-
      ragg::agg_capture(
        width = size[1], height = size[2],
        units = "in",
        res = .res,
        scaling = .scaling,
      )
    if (.force) {
      showtext::showtext_begin()
    }
    NextMethod(object = p)
    raster <- cap(native = TRUE)
    if (.force) {
      showtext::showtext_end()
    }
    dev.off()

    if (names(dev.cur()) != "unigd") {
      dev.set(dev.list()["unigd"])
    }
    grid::grid.newpage()
    grid::grid.raster(raster, interpolate = FALSE)
  }
}
```

このコードでは、`print.gg`というメソッドをつくっている。同名のメソッドはggplot2パッケージのなかにあるもの。`ggplot2::ggplot`でつくったグラフのオブジェクトには`gg`と`ggplot`というクラスが付けられているので、この`print.gg`はそれらをprintしようとしたときに呼ばれるメソッドをマスクすることになる。本来の`print.gg`が何をしているのか知らないのでマスクしてしまっていいものなのかよくわからないが、なんとなく期待した感じに動いているので、たぶん大丈夫。

実際にやっていることとしては、`cap <- ragg::agg_capture(); raster <- cap(native = TRUE)`とすることで、raggのそのときのデバイスの状態をnativeRasterとしてキャプチャできるので、それをやったうえでこのnativeRasterをhttpgd（unigd）に渡してプロットしなおすといったことをやっている。やりたかったようなことはどうやらできているが、ちょっと挙動がもっさりするような気はする。

ちなみに、QuartoとかRmdのなかでは、knitrのオプションを通じてグラフィックデバイスを指定できるので、knit/renderするときにAGGを使いたいという場合なら、こういう謎のハックは使わなくてよい。


## 解決策2（2026年3月追記）

解決策1の挙動がもっさりする主な原因は、`ragg::agg_capture`をそのたびごとに開いたり閉じたりしなければならない点にある。

フォントのフォールバックや比較的最近になってR言語に追加されたグラフィックデバイスの機能を利用したい場合、httpgd（unigd）ではなくraggが有力な選択肢になるのだが、`ragg::agg_capture`を開いてしまうとアクティブなグラフィックデバイスがこれに切り替わってしまうので、キャプチャした画像をプレビューするためには一度画像ファイルに書き出すか、このデバイスを閉じる（あるいは、このデバイスとは別のデバイスに切り替える）必要がある。VS Codeで描画の結果を確認したいだけなら画像ファイルに書き出したうえでそのファイルを開いてしまってもよいのだが、それだったら最初から画像ファイルを書き出しながら作業すればいいわけで、それだと「プレビュー」にはなっていない。

そもそも、Rのグラフィックデバイスはやや独特なつくりをしている。画像を描画するステップと表示するステップとが上手く分離されていないのだ。Rでは`plot`や`print.ggplot`が描画から表示までをまとめておこなう副作用をもっており、しかも、それらは副作用であるため、ふつうに使っていると、描画結果を値としてもつことができない。R言語ユーザーは当初からRでのグラフの扱いはそういうものだと思っているふしがあるので違和感を感じないかもしれないが、これは体験としてはちょっと奇妙だと思う。

多くのユーザーが実際にやっているのは画像を描画したら試しに表示してみて、よかったら画像ファイルとして保存するというステップであり、これらは可能ならばRコンソールから離れることなく実現できたほうがスマートなはずだ。つまり、私の当初の目的は、

1. raggでグラフを描画する
2. その描画結果をプレビューする
3. OKだと思ったら画像ファイルとして書き出す

というステップを踏むことなのであって、「AGG（ragg）を使って描いた図をPlot viewerのなかでプレビューする」というのはそのための方法の一つでしかない。むしろ、描画は`ragg::agg_capture`でおこないつつも、プレビュー表示にはPlot viewer（グラフィックデバイス）を使わないほうが`ragg::agg_capture`の利便性を活かしやすいだろう。

もともとそのためにつくったわけではないのだが、それが実現できる機能を自作Rパッケージに用意した。

- [paithiov909/rravif: AVIF writer for R](https://github.com/paithiov909/rravif)

このパッケージに含まれる`rravif::print_nr`を次のように使うと、`ragg::agg_capture`による描画結果のnativeRaster画像を、グラフィックデバイスとして開かれる窓ではなく、Rコンソールの中にインライン表示することができる。

```r
library(ggplot2)

print.nativeRaster <- \(x, ...) rravif::print_nr(x, ...)

cap <- ragg::agg_capture(width = 4, height = 3, units = "in")

ggplot(mtcars, aes(x = wt, y = mpg)) +
  geom_point() +
  theme_dark()

cap(native = TRUE)
```

使っているターミナルエミュレータの対応状況によって上手く表示されたりされなかったりするかもしれないものの、たとえばVS Codeのターミナルで必要な設定を有効にしていると、だいたい次の動画のような感じに表示される。

https://x.com/paithiov909/status/2037115900068020371

これはterminal graphicsと呼ばれるような技術によって表示しているもので、内部的には[viuer](https://github.com/atanunq/viuer)というRustのライブラリを使って雑に実現している。ちなみに、これと似たような発想のもとで開発されているらしい既存のRパッケージとしては次の2つがある。

- [terminalgraphics](https://codeberg.org/djvanderlaan/terminalgraphics)
- [rsixel](https://github.com/Fan-iX/rsixel)

このやり方でプレビューしてみて、OKだと思ったら`fastpng::write_png`とか、好みの方法で画像ファイルに書き出せばよい。`rravif::write_avif`でも、nativeRasterからAVIFファイルを書き出すことができる。

```r
cap(native = TRUE) |>
  rravif::write_avif("test.avif")
```
