---
title: "ggplot2のinkとpaperの色は「混ざる」わけじゃないよ"
emoji: "🧻"
type: "idea"
topics: ["r", "ggplot", "ggplot2"]
published: false
---

# ggplot2のinkとpaperの色は「混ざる」わけじゃないよ

## この記事について

いま、私がこの記事を書いている時点で、ggplot2のv4.0.0がとうとうリリースされようとしています。

https://github.com/tidyverse/ggplot2/pull/6566

あるいは私がこの記事を投稿したころには、すでにggplot2 v4.0.0がリリースされているかもしれません。楽しみですね。

ggplot2のv4.0.0については、内部的にS3からS7を採用するかたちに大幅に書き換えたらしいとか、すでにいくつかの情報が明らかになっています。まあ、詳しい情報については、きっとtidyverseのブログ記事などで紹介されることでしょう。

「v4.0.0で来るぞ！」と言われていた新機能として、`theme_*`から`ink`と`paper`という引数を指定できるようになるという話があります。おそらく次のPRで追加された機能のことでしょう。

- [Ink and paper settings for built-in themes by teunbrand · Pull Request #6059 · tidyverse/ggplot2](https://github.com/tidyverse/ggplot2/pull/6059)
- [Ink and paper settings for built-in themes (re: #6059) by teunbrand · Pull Request #6063 · tidyverse/ggplot2](https://github.com/tidyverse/ggplot2/pull/6063)

上のほうのPRでのteunbrandの説明によると、

> The `ink` sets all foreground elements, which by default is 'black'. Likewise, `paper` sets all background elements, which by default is 'white'.

ということで、グラフを構成する要素のうち、`geom_*`で描かれるようなメインの要素（foreground）と、たぶんパネルとか軸とか凡例とか？いった背景になるべき要素（background）の色をある程度まとめて指定できるもののようです。ユタニさんがすでに試していましたが、具体的には、次の投稿みたいな雰囲気のものみたいですね。

https://x.com/yutannihilation/status/1934077365891821783

実際にマージされているPRにおける実装を雑に眺めた感じ、これらの色は、それぞれの要素ごとに`ink`と`paper`の色を直に渡したり、あらかじめ指定された割合で補間したりすることによって決められているように見えます。どんなふうに使われることを意図しているのか私にはよくわかりませんが（それっぽい配色をもっと手早く書けるようにしたいとか？）、なんだかおもしろい機能です。

この記事では、このへんの色の補間の処理で何がおこなわれているのかを簡単に説明しつつ、どんなことに気を付けて色を指定すればよさそうか紹介します。


## inkとpaperのあいだの補間処理

さて、ggplot2のRC4.0.0であるv3.5.2.9002で、次のコードを使ってグラフを書いてみたところ、

```r
library(ggplot2)

ggplot(iris, aes(x = Sepal.Length, y = Sepal.Width, color = Species)) +
  geom_point() +
  theme_gray(
    ink = "yellow",
    paper = "gray40"
  )
```

次のような画像になりました。

![inkにyellow、paperにgray40を指定したggplot](https://storage.googleapis.com/zenn-user-upload/e8818ec86c5c-20250805.png)

ここで、たとえばパネルの背景色を決めているのは、どうやら[この部分のコード](https://github.com/tidyverse/ggplot2/blob/6d48584c6c3396bc108218487f60fa577da5a767/R/theme-defaults.R#L215C4-L215C86)で、

```r
panel.background = element_rect(fill = col_mix(ink, paper, 0.925), colour = NA),
```

というように決められています。わかりにくいですが、これは`gray40`そのままではなく、ごくわずかに黄色がかっているようです。

この`col_mix()`は、[scalesパッケージのこの関数](https://scales.r-lib.org/reference/col_mix.html)で、2色`a,b`と割合`amount`を与えると、任意の色空間のなかでそれらの色のあいだの補間をおこなうものです。内部的には、`farver::decode_colour()`で色空間を変換してから`new <- (a * (1 - amount) + b * amount)`のようにして線形補間してやって、できた色`new`を`farver::encode_colour()`で元の色空間に戻しています。ここで`decode_colour()`に渡される色空間は4番めの`space`引数から指定できるのですが、ggplot2のコード中では指定していないので、そのままsRGBで補間されています。

つまり、上の`panel.background`の色は、次と同じです。

```r
scales::col_mix("yellow", "gray40", .925, space = "rgb")
#> [1] "#71715EFF"
```

このあたりのデフォルトの補間の割合は`theme_*`のなかで決め打ちされているっぽいので、実際にどんな発色になるかは試してみないことにはわからないと思います。


## scales::col_mix()は色を混ぜる関数ではない

やや罠っぽいというか、初見殺しなところがあるのですが、`scales::col_mix()`はその名前に反して、色を混ぜる（混色する）ことができる関数ではありません。

たとえば、サクラクレパスの[この記事](https://www.craypas.co.jp/press/feature/009/sa_pre_0245.html)によると、黄色と青の絵の具を1：1の割合で混ぜると緑になるらしいです。そんな気がしますよね。

一方で、デジタルの色表現で黄色と青とのあいだのちょうど中間点を取ったときにどんな色になるかは、色空間によっても異なるものの、少なくとも、sRGBで黄色（`#ffff00ff`）と青（`#0000ffff`）の中間点を取ったら`#808080ff`になるので、まあ、緑にはなりません。

```r
interp_lab <- scales::col_mix("yellow", "blue", seq(0, 1, length.out = 21), space = "lab")
interp_lch <- scales::col_mix("yellow", "blue", seq(0, 1, length.out = 21), space = "lch")
interp_oklab <- scales::col_mix("yellow", "blue", seq(0, 1, length.out = 21), space = "oklab")
interp_oklch <- scales::col_mix("yellow", "blue", seq(0, 1, length.out = 21), space = "oklch")
interp_rgb <- scales::col_mix("yellow", "blue", seq(0, 1, length.out = 21), space = "rgb")
interp_mixbox <-
  purrr::map_int(seq(0, 1, length.out = 21), \(t) {
    mixboxr::lerp("yellow", "blue", t)
  }) |>
  colorfast::int_to_col()

opar <- par(mfrow = c(6, 1), mar = c(1, 1, 2, 1))
image(1:21, 1, as.matrix(1:21), col = interp_lab, axes = FALSE, main = "CIELAB")
image(1:21, 1, as.matrix(1:21), col = interp_lch, axes = FALSE, main = "LCH")
image(1:21, 1, as.matrix(1:21), col = interp_oklab, axes = FALSE, main = "Oklab")
image(1:21, 1, as.matrix(1:21), col = interp_oklch, axes = FALSE, main = "Oklch")
image(1:21, 1, as.matrix(1:21), col = interp_rgb, axes = FALSE, main = "RGB")
image(1:21, 1, as.matrix(1:21), col = interp_mixbox, axes = FALSE, main = "Mixbox")
```

![いくつかの色空間における黄～青のグラデーション](https://storage.googleapis.com/zenn-user-upload/f713333c92fd-20250805.png)

farverがサポートしている色空間のうちでも、たとえばLCHなどは、黄色と青とのあいだで補間したときに中間色で緑っぽい色ができます。しかしこれも、ただこういうふうに補間される色空間であるというだけで、べつに混色しているわけではないです。

デジタルの色について、アナログ世界の絵の具の振る舞いのような色の混ざり方を再現したい場合、[Mixbox](https://scrtwpns.com/mixbox/)のような、ガチの混色がおこなえる実装が必要になります。この上のコード例（一番下のグラデーション）で使っている[mixboxr](https://github.com/paithiov909/mixboxr)は、MixboxのC++実装を使って簡単なRパッケージとして整備したものです。


## inkとpaperの指定のポイント

いずれにせよ、`ink`と`paper`から色を指定する際に気をつけるべきなのは、それらの色がsRGB空間で補間されるという点です。`scales::col_mix()`による補間は、べつに色を混ぜているわけではなく、あくまで補間処理であるため、`ink`と`paper`から両端の色を指定するやり方では、中間色で意図していない色味が出てしまうことがあるかもしれません。

実際、先ほどのいくつかの色空間におけるグラデーションを掲載した図では、黄色～青のあいだのsRGB空間におけるグラデーションは、途中で灰色っぽくくすんでしまっていました。したがって、`ink`と`paper`を指定する際には、たぶんですが、こうしたくすみが生じないような配色を意識するのが無難だろうと思われます。

具体的には、

- どちらも極端に彩度の高い色にしすぎない
- ある程度似たトーンや色味の組み合わせを選ぶ（たとえば、両端を補色どうしとかにはしないほうがよい）

とかがポイントになるかもしれません。


## まとめ

というわけで、ggplot2 v4.0.0の新機能である`ink`と`paper`の補間について、ごく簡単に紹介しました。

この記事でおもに伝えたかったことは「`ink`と`paper`を介してできるのは、`scales::col_mix()`による線形補間を使った配色であって、これは色を混ぜてるわけじゃない」ということです。デジタルの世界の色の話なので、考えてみればそれはそうなんですが、その点を誤解して使ってしまうと、意図した配色が全然できなさそうな予感がします。

ごく簡単な記事ではありますが、この機能でよさげな配色をつくる手がかりになればいいなと思います。
