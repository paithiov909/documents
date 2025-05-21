---
title: "WebRでも動くgganimateのレンダラーを書く"
emoji: "🖼"
type: "tech"
topics: ["r"]
published: false
---


## gganimateのレンダラーを書く

次は、実際にWebRでアニメーションをつくってみているデモです。

https://paithiov909.quarto.pub/poc-apng_renderer-for-gganimate/

ここでは、`apng::apng()`を使ったgganimateのレンダラー（`apng_renderer()`）を書いて、それを使ってAPNGファイルを返して表示しています。

[APNG](https://wiki.mozilla.org/APNG_Specification)というのは、PNGを拡張してアニメーションとして表示できるようにしたファイル形式で、Webでアニメーションを表示させるのに稀に使われたりするもののようです。WebPのアニメーションのほうが便利そうな気がするものの、たとえば、LINEのスタンプのうちでも動くやつとかはこの形式で作成されるらしいです。

RでAPNGファイルを書き出すには、[apng](https://CRAN.R-project.org/package=apng)というパッケージを利用することができます。apngパッケージはCRANからインストールできるうえに、bitopsパッケージへの依存を除けば、これ自体はR言語だけで実装されていることから、WebRでも問題なく（たぶん何かしらのwarningは出してくるものの）動かすことができます。

上のデモページでは、`apng_renderer()`は、だいたい次のようなfactoryとして書いています。

```r
apng_renderer <- function(file = NULL, loop = TRUE) {
  rlang::check_installed("apng", "to use the `apng_renderer`")
  function(frames, fps) {
    if (is.null(file)) {
      file <- tempfile(fileext = ".png")
    }
    if (!all(grepl(".png$", frames))) {
      cli::cli_abort("{.pkg apng} only supports png files", call. = FALSE)
    }
    loop <- if (loop) 0 else 1
    apng::apng(frames, output_file = file, num_plays = loop, delay_num = 1, delay_den = fps)
    apng_file(file)
  }
}
```

`gganimate::animate()`の`rederer`に渡すレンダラーを自分で実装する場合、`frame`と`fps`をこの順に受け取る関数を返すようなfactoryとして書きます。`frames`はgganimateが書き出したフレーム画像の一時ファイルへのパス（文字列ベクトル）で、`fps`はfpsを表すnumeric scalarのようです。

gganimateのレンダラーの実装例としては、実際のgganimateのレンダラーを参考にするのが手っ取り早いです（[renderers.R](https://github.com/thomasp85/gganimate/blob/main/R/renderers.R)を見ましょう）。`file`をそのまま返すかたちにしてしまえば、上の`apng_renderer()`だけでもAPNGを書き出すという用は足りるのですが、実際のgganimateのレンダラーは、`print()`したときなどにいい感じに表示されるように工夫されていたので、それを真似するなら次のようにクラスを付けつつ、メソッドを定義するとよいでしょう。

```r
apng_file <- function(file) {
  if (!grepl(".png$", file)) cli::cli_abort("{.arg file} must point to a png file")
  class(file) <- "apng_image"
  file
}

print.apng_image <- function(x, ...) {
  viewer <- getOption("viewer", utils::browseURL)
  if (rlang::is_function(viewer) && length(x)) {
    viewer(x)
  }
  invisible(x)
}

knit_print.apng_image <- function(x, options, ...) {
  knitr::knit_print(htmltools::browsable(as_apng_html(x, width = get_chunk_width(options))), options, ...)
}

as_apng_html <- function(x, width = NULL, alt = "") {
  rlang::check_installed("base64enc", "for showing apng")
  rlang::check_installed("htmltools", "for showing apng")
  if (is.null(width)) width <- "100%"
  image <- base64enc::dataURI(file = x, mime = "image/apng")
  htmltools::tags$img(src = image, alt = alt, width = width)
}

split.apng_image <- function(x, f, drop = FALSE, ...) {
  cli::cli_abort("{.cls apng_image} objects does not support splitting")
}
```

こうして一連のメソッドを定義したうえで、`apng_renderer()`を次のように使うと、使っている環境のviewerが開いてAPNG画像が表示されます。なお、この使用例のコードは[gganimateのWiki](https://github.com/thomasp85/gganimate/wiki/Temperature-time-series)に載っている例から借りたものです。

```r
suppressPackageStartupMessages({
  library(ggplot2)
  library(gganimate)
})

airq <- airquality
withr::with_locale(c(LC_TIME = "en_US.UTF-8"), {
  airq$Month <- format(ISOdate(2004,1:12,1),"%B")[airq$Month]
})

p <- ggplot(airq, aes(Day, Temp, group = Month)) +
  geom_line() +
  geom_segment(aes(xend = 31, yend = Temp), linetype = 2, colour = 'grey') +
  geom_point(size = 2) +
  geom_text(aes(x = 31.1, label = Month), hjust = 0) +
  transition_reveal(Day) +
  coord_cartesian(clip = 'off') +
  labs(title = 'Temperature in New York', y = 'Temperature (°F)') +
  theme_minimal() +
  theme(plot.margin = margin(5.5, 40, 5.5, 5.5))

# `apng::apng()`はPNGファイルの一部のチャンクを上手くパースせず、警告を出してくるため、`suppressWarnings()`でくくる
suppressWarnings({
  animate(p, renderer = apng_renderer(), nframes = 50, type = "cairo-png")
})
```

ZennがAPNGを素直に表示してくれるのか謎ですが、雰囲気としては次のようなアニメーションが表示されます。

![gganimateで書き出したAPNG画像](https://storage.googleapis.com/zenn-user-upload/da432494b55c-20250521.png)

ちなみに、APNGはおそらく一般的な画像ビューアでは表示できませんが、主要なモダンブラウザでは問題なく表示されるはずです。RStudioのviewerペインやVSCodeなどなら、たぶんふつうに表示されます。


## APNGをRで書き出す際の注意点

`apng::apng()`で書き出されるAPNGファイルはファイルサイズが大きくなりがちです。本来、APNGファイルを書き出すときには、一連のPNG画像をzlibなどで圧縮するようなのですが、[apngのソースコード](https://github.com/qstokkink/apng)を見た感じそんなことはしておらず、すべてのフレーム画像がそのまま一つのAPNGファイルに格納されることになるため、ちょっと油断すると、無圧縮の動画ファイルみたいな巨大なAPNGファイルが書き出されてしまいます。

そのため、もしここで紹介した`apng_renderer()`を使うつもりなら、フレーム画像のサイズを大きくしすぎない・フレームの数を多くしすぎないといった点に十分に注意して使う必要があります。


## まとめ

というか、WebPのアニメーションが書き出せるRパッケージがほしいですよね。Jeroenが[webp](https://github.com/jeroen/webp)というRパッケージをすでに作っていて、これはlibwebpにリンクしているものらしいのだけど、これはアニメーションを書き出すAPIは使えないみたい。できそうなら書いてみたいけど、C++のAPIを直に触るのは自分にはしんどそうなので悩ましいです。
