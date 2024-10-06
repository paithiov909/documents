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


## 解決策

書いた当人もよくわからないコードになったが、下のようなコードを`.Rprofile`とかに書いておいて実行すると、やりたかったようなことが実現できた。

```r
if (interactive() && Sys.getenv("TERM_PROGRAM") == "vscode") {
  #' Capture ploting with ggplot on AGG into UNIGD device
  #' @param ... other parameters
  #' @param .res resolution
  #' @param .scaling scaling
  print.gg <- function(..., .res = 72, .scaling = 1, .force = FALSE) {
    rlang::check_installed(c("ggplot2", "ragg", "showtext"))

    size <- dev.size()
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
    raster <- cap()
    if (.force) {
      showtext::showtext_end()
    }
    dev.off()

    if (names(dev.cur()) != "unigd") {
      dev.set(dev.list()["unigd"])
    }
    plot(as.raster(raster))
  }
}
```

このコードでは、`print.gg`というメソッドをつくっている。同名のメソッドはggplot2パッケージのなかにあるもの。`ggplot2::ggplot`でつくったグラフのオブジェクトには`gg`と`ggplot`というクラスが付けられているので、この`print.gg`はそれらをprintしようとしたときに呼ばれるメソッドをマスクすることになる。本来の`print.gg`が何をしているのか知らないのでマスクしてしまっていいものなのかよくわからないが、なんとなく期待した感じに動いているので、たぶん大丈夫。

実際にやっていることとしては、`cap <- ragg::agg_capture(); raster <- cap()`とすることで、raggのそのときのデバイスの状態をrasterとしてキャプチャできるらしいので、それをやったうえでこのrasterをhttpgd（unigd）に渡してプロットしなおすといったことをやっている。やりたかったようなことはどうやらできているが、ちょっと挙動がもっさりするような気はする。


## むすび

ちなみに、QuartoとかRmdのなかでは、knitrのオプションを通じてグラフィックデバイスを指定できるので、knit/renderするときにAGGを使いたいという場合なら、こういう謎のハックは使わなくてよい。

もっとよさげなやり方があったら教えてください。
