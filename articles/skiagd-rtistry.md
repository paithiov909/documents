---
title: "R言語でのアート制作・Rtistryを考える"
emoji: "🖼️"
type: "idea"
topics: ["r", "creativecoding", "skia"]
published: false
---

## この記事について

この記事は[R言語 Advent Calendar 2025](https://qiita.com/advent-calendar/2025/rlang)のXX日めの記事です。

みなさんにとって2025年はどんな年でしたか？　私は、この2025年のあいだ、個人でいくつかのRパッケージを開発していました。

- [paithiov909/skiagd: A toy R wrapper for 'rust-skia'](https://github.com/paithiov909/skiagd)
- [paithiov909/rasengan: Generation of geometric curves](https://github.com/paithiov909/rasengan)
- [paithiov909/aznyan: Image filters for R](https://github.com/paithiov909/aznyan)
- [paithiov909/nativeshadr: Introduction of HLSL-like syntax to Rcpp for writing pixel shaders](https://github.com/paithiov909/nativeshadr)

これらのパッケージは、ざっくりいうと、R言語を使って「絵を描く」ためのものです。開発したパッケージのことや、実際にアート制作に取り組んでみた話は、これまでにも何度かブログ記事を書いて紹介しています。

- [Rで雑にクリエイティブコーディングしたい](https://zenn.dev/paithiov909/articles/skiagd-intro)
- [Rでクリエイティブコーディング企画に挑戦している話 - リリカルはなくそオーガスタ](https://lyrikuso.netlify.app/minacoding-in-r/)
- [R言語にシェーダー芸を求めるのは間違っているだろうか - リリカルはなくそオーガスタ](https://lyrikuso.netlify.app/beyond-the-heliosphere/)

私がそもそもやりたかったことは、この一番上の記事で説明しています。要約すると、私は、何らかのアニメーション作品をつくることをゴールとして、R言語を使って動画のフレーム画像を描き出せるような仕組みをつくろうとしていました。

結果として、開発当初の「モーショングラフィックス的な映像作品をつくりたい」というイメージからは離れてしまったものの、実際にさまざまな表現が可能なRパッケージ群をつくることができました。

この記事では、これらのパッケージ群を使ったR言語でのアート制作の事例について、2025年の個人的な活動のまとめとして紹介します。

## R言語とジェネラティブアート

先ほど挙げた記事のなかでも触れていたのですが、はじめに、R言語とアート制作との関係について、あらためて紹介しましょう。

R言語ユーザーにはあまり馴染みがない世界かもしれませんが、何らかの創作物をつくるうえでプログラミングをしているクリエイターというのは、実は世の中にたくさんいます。創作の手段としてコードを書いて画像や映像・音楽などをつくっている人たちのなかでも、「コードを書いてアート制作するという活動そのもの」を目的としている人たちは、そうした活動を指してクリエイティブコーディングと呼んだりします。

その作者がクリエイティブコーディングという言葉を使っているかはともかく、プログラムを書くことによって画像や映像を生成するタイプのアートはある種のジャンルとして確立されていて、そうしたアート作品はジェネラティブアートと呼ばれたりします。ジェネラティブアートの「ジェネラティブ」というのは、同じアルゴリズムを実装したコードであっても、渡されるパラメータやランダムシードの違いによって、生成される作品がコードを実行するたびに変化するという側面について言及しているものです。

もっとも、一口にクリエイティブコーディングとかジェネラティブアートなどと言っても、その制作過程で実際にやっていることは、作り手によってかなり違います。使用されるプログラミング言語・フレームワークやライブラリも、比較的メジャーと言えそうなものはいくつかありますが、それぞれに得意とする表現や書き心地がけっこうバラバラな感じです。ある意味で掴みどころのない世界なわけですが、良い意味に捉えれば、とても自由な世界だとも言えます。

そんな自由な世界なので、R言語を使ってジェネラティブアートを制作している人たちもいます。R言語製のジェネラティブアートには英語圏では[#rtistry](https://x.com/search?q=%23rtistry)というタグが付けられていることが多いです。活発に作品を投稿している人の数はそれほど多くないものの、SNSなどで作品を探すことができます。また、こうしたrtistryの例は、Koen Derksさんがメンテナンスしている[aRtsy](https://github.com/koenderks/aRtsy)というパッケージを使えば、誰でも簡単に次のようなアートを生成してみることができます。

![aRtsyで生成されたアート1](https://storage.googleapis.com/zenn-user-upload/935089068fbb-20251118.png)
*aRtsyで生成された'Forests'の例*

![aRtsyで生成されたアート2](https://storage.googleapis.com/zenn-user-upload/0c391acf1e11-20251118.png)
*aRtsyで生成された'Swirls'の例*

ほかにも、Nicola Rennieさん、Danielle Navarroさん、Thomas Lin Pedersenさんなど、rtistry界隈でよく名前を見かけるジェネラティブアーティストが何人かいます。このへんの人たちの作品を眺めてみることでも、R言語でつくるジェネラティブアートの雰囲気は伝わるんじゃないかと思います。

- [Generative Art – Nicola Rennie](https://nrennie.rbind.io/projects/generative-art/)
- [Art by Danielle Navarro](https://art.djnavarro.net/)
- [Selected work – Thomas Lin Pedersen](https://thomaslinpedersen.art/work/#category=R)

## Rtistryとはどのようなアートなのか

さて、既存のrtistry作品の雰囲気は、実際の作品の画像を見ればなんとなくわかってもらえただろうと思います。ジェネラティブアートの一つの系統であるrtistryは、絵づくりとしては「欧米っぽい」というか、わりと伝統的なジェネラティブアートからの影響を強く感じさせます。これらはどのようにして描かれているのでしょうか？

rtistryを生成するコードの説明はインターネット上に散らばっていて、なかなか一か所で網羅的に確認するのは難しいのですが、Danielle Navarroさんの次の資料などは比較的ボリュームがあって読みやすいです。

- [ART FROM CODE](https://art-from-code.netlify.app/)

この資料は2022年のRStudio Confで開催されたワークショップ用に準備されたもので、Danielleさんがrtistryを制作するのに使ったテクニックの一部がrtistry初心者にもわかりやすく説明されています。資料としてとてもよくできていますが、Rの使用例としてはかなりいろいろな方面に手を伸ばしているため、R言語の総合格闘技っぽさがあります（たぶん2日間の日程でこなすには相当難しかったはず）。

この資料を参考にしつつエッセンスだけを指摘すると、こういったrtistryはふつう、[ggplot2](https://ggplot2.tidyverse.org/)を使って描かれています。みなさんも知っているだろう、R言語のグラフ作成ライブラリのデファクトスタンダードであるところの、あのggplot2です。したがって、一見どうやって描かれているのか想像できないような作品であっても、rtistryは基本的にただのグラフです。`theme_void()`などで背景や凡例を消しながら、`geom_point()`とか`geom_segment()`とかで描かれています。

たとえば、次はART FROM CODEからの例ですが、こういうデータを用意しておいて、

```r
n <- 50
dat <- dplyr::tibble(
  x0 = runif(n),
  y0 = runif(n),
  x1 = x0 + runif(n, min = -.2, max = .2),
  y1 = y0 + runif(n, min = -.2, max = .2),
  shade = runif(n),
  size = runif(n)
)
dat
#> # A tibble: 50 × 6
#>        x0     y0      x1      y1 shade  size
#>     <dbl>  <dbl>   <dbl>   <dbl> <dbl> <dbl>
#>  1 0.988  0.938   0.965   0.990  0.540 0.186
#>  2 0.455  0.746   0.625   0.811  0.818 0.102
#>  3 0.182  0.0389  0.227  -0.0856 0.134 0.314
#>  4 0.0276 0.271  -0.0288  0.122  0.103 0.527
#>  5 0.160  0.131   0.347   0.173  0.470 0.878
#>  6 0.527  0.927   0.405   0.930  0.157 0.106
#>  7 0.329  0.189   0.461   0.0136 0.691 0.491
#>  8 0.0605 0.407   0.231   0.341  0.630 0.960
#>  9 0.439  0.931   0.316   0.883  0.879 0.799
#> 10 0.863  0.737   0.754   0.618  0.348 0.566
#> # ℹ 40 more rows
```

こうしたりすると、何かしらのアートらしき画像が描けます。実際はもっと複雑なデータを準備するわけですが、大枠としてはこんな感じです。

```r
library(ggplot2)

dat |>
  ggplot(aes(
    x = x0,
    y = y0,
    xend = x1,
    yend = y1,
    colour = shade,
    linewidth = size
  )) +
  geom_segment(show.legend = FALSE) +
  coord_polar() +
  scale_y_continuous(expand = c(0, 0)) +
  scale_x_continuous(expand = c(0, 0)) +
  scale_color_viridis_c() +
  scale_size(range = c(0, 10)) +
  theme_void()
```

![簡単なrtistryの例](https://storage.googleapis.com/zenn-user-upload/106d5c105fdf-20251118.png)
*ランダムデータから極座標系で線分を引いたアート*

そもそも、ggplot2は本来はグラフを描くためのライブラリです。そのため、設計思想として、プロットしたいデータ系列をまるごと与えながら、それらを一枚のレイヤーとして一息にプロットできるようにデザインされています。だから、ggplot2を使う場合、rtistryを制作するときであっても、実際に絵を描画する部分のコードとそこに与えるデータ系列を準備する部分のコードが自然と分離されるような書き方になります。

一方で、ジェネラティブアートの制作によく用いられる他のライブラリと比較すると、実は、この書き方はけっこう異質なものです。

たとえば、クリエイティブコーディングしている人たちがよく使っているp5.jsのようなProcessing系のライブラリでは、線はその線分を描くための点（始点と終点）を与えることによって描きます。つまり、こうしたライブラリは、素朴なイメージとしてはペンプロッターのような挙動をするようにデザインされており、ユーザーはふつう`for`文などのなかでペンの位置を少しずつ動かしながら、任意の位置に図形を描き重ねていくような気持ちでコードを書いています。

これに対して、あえて単純化して言えば、rtistryというのは、データをまるごと与えることを通じて図形の配置を決めるという、データ駆動的なアプローチで制作されるジェネラティブアートだと言えます。もちろん、それだからどちらのほうが優れているといった話ではなくて、シンプルに書き方の違いです。言い換えると、rtistryは「データの前処理を自由にいじることによって、出力される作品が変化する」ように書かれている、ジェネラティブアートのサブジャンルだと理解してもらえば、だいたいあっていると思います。

### ggplot2（rtistry）と p5.js（Processing系）の描画モデルの違い

| 観点 | rtistry（ggplot2） | p5.js / Processing系 |
|----|----|----|
| **描画単位** | レイヤー（多くの図形をまとめて描く） | 図形1つ（ペンの1ストローク） |
| **描き方の発想** | 事前に「点群・線分などのデータ」を全部まとめて用意し、それを一括で渡す | ループの中で毎回「ペンを動かして描く」イメージ |
| **コード構造** | データ生成（dplyr・行列計算）と描画（geom）を分離して書く | `for` などのループの中で `line()` や `ellipse()` を順次呼ぶ |
| **ブレンドのタイミング** | レイヤー単位（同一レイヤーの図形間ではアルファ合成のみ） | 図形ごとにブレンドが即時適用される（逐次ブレンド） |
| **向いている表現** | データ駆動型・統計グラフ的・大量の点群を整然と処理する構造 | ペンプロッターのような連続的・手描き的なストローク表現 |
| **作品の考え方** | 「どんなデータを用意すればこの形が生まれるか」を考える | 「どんな軌跡でペンを動かすか」を考える |

## Rでのアート制作の課題感

rtistry流のデータ駆動的なジェネラティブアートは、R言語ユーザーにとってはわかりやすく、コードを書きやすいアプローチです。しかし、私がゴールとしていたような「アニメーション作品」をつくろうとする場合、ggplot2を使ったフレーム画像づくりには、いくつかの点で限界がありそうでした。

まず第一に、ggplot2は描画速度が遅すぎるので、アニメーションをつくったりするのは苦手です。ggplot2は本来の用途としてはグラフを描くためのものなので、グラフを静止画として書き出しているぶんには不便さは感じませんが、これでアニメーションをつくろうとすると、1枚のグラフをプロットするのに500ms以上かかったりして大変なことになります。

また、ggplot2では、画像を描くための手段としてR言語のグラフィックデバイスに依存していることに由来する独特な制限もあります。

たとえば、p5.jsなどでは、図形をキャンバスに描きくわえていくとき、まさにそのタイミングごとにブレンドモードが適用されるため、円や線といった一つ一つの図形どうしをブレンドモードで重ねることができます（いわゆる「逐次ブレンド」）。

一方、Rのグラフィックデバイスにもブレンドモードは機能として存在しているものの、Rではこのブレンドモードを適用できる単位が`grob`オブジェクトであるせいで、とくにggplot2ではレイヤー単位でしかブレンドモードが適用できません。つまり、同一のレイヤー内ではアルファブレンドしかできず、一枚の`geom_point()`内で描かれる点どうしをスクリーンで重ねるといったことが不可能なのです。

## ggplot2じゃないRtistryを目指して

前置きが長くなりましたが、このあたりの「rtistry的なやり方でクリエイティブコーディングしたいけど、道具はggplot2じゃないやつがほしい！」というギャップを埋めるために開発したのが、2025年に開発した一連のパッケージになります。

その中心となっているのがskiagdパッケージです。これは[skia-safe](https://github.com/rust-skia/rust-skia)というRustのライブラリを[savvy](https://github.com/yutannihilation/savvy)を使ってラップしているパッケージで、Rのグラフィックデバイスからは独立しているSkiaのパイプラインを使って、画像を生成することができるというものです。

これによってR言語からはじめて可能になっただろう表現もたくさんあります。が、説明するよりも、サクッと例を見てもらったほうがわかりやすいと思うので、まずは作品を見てもらいましょう。次は、[#minacoding](https://minacoding.online/index-ja)というクリエイティブコーディング企画に挑戦したときに制作した作品です。

https://x.com/paithiov909/status/1938766924072309023

これは全体として360フレームからなるアニメーションで、次のようなコードを使って書き出したものです（けっこう長いので、興味がなければ読み飛ばしても大丈夫です！）

:::details 実際のRコード
```r
library(skiagd)
library(affiner)

cv_size <- c(800L, 600L)
prps <- list(canvas_size = cv_size)

n_seq <- 360

dat <-
  dplyr::tibble(pid = seq_len(4)) |>
  dplyr::reframe(
    sid = seq_len(n_seq),
    pos = dplyr::tibble(
      x = e1071::rbridge(frequency = n_seq),
      y = e1071::rbridge(frequency = n_seq),
      z = e1071::rbridge(frequency = n_seq),
      w = 1
    ) |>
    as.matrix(),
    .by = pid
  )

pngs <- purrr::imap_chr(seq_len(n_seq), \(k, i) {
  t <- 1 - rasengan::ease_in(i / n_seq, "sine")
  d <- dat |>
    dplyr::slice_head(n = k, by = pid) |>
    dplyr::slice_tail(n = floor(k * t + 1), by = pid) |>
    dplyr::mutate(
      pos = pos %*%
        transform3d() %*%
        scale3d(250) %*%
        rotate3d("z-axis", k) %*%
        rotate3d("x-axis", 45) %*%
        rotate3d("y-axis", 35) %*%
        translate3d(x = cv_size[1] / 2, y = cv_size[2] / 2, z = 0),
      r = log(rasengan::mag(pos)) * seq(.5, 4, length.out = dplyr::n()),
      .by = pid
    )

  img <-
    canvas("snow", canvas_size = cv_size) |>
    purrr::reduce(unique(d$pid), \(cv, id) {
      cv |>
        add_circle(
          d |>
            dplyr::filter(pid == id) |>
            dplyr::pull(pos),
          radius = d |>
            dplyr::filter(pid == id) |>
            dplyr::pull(r),
          props = paint(
            !!!prps,
            style = Style$Fill,
            path_effect = PathEffect$discrete(3, 2, 1),
            blend_mode = BlendMode$Difference,
            color = "yellow", ## 白地にDifferenceするので補色の青っぽい色になる
          )
        )
    }, .init = _) |>
    as_png(props = paint(!!!prps))

  fp <- file.path("public/pictures", sprintf("%04d.png", i))
  writeBin(img, fp)

  fp

}, .progress = TRUE)

gifski::gifski(
  pngs,
  "out/minacode-28.gif",
  width = cv_size[1],
  height = cv_size[2],
  delay = 1 / 20,
  progress = TRUE
)
```
:::

ggplot2ではこれと同じような表現は不可能だと思いますし、そもそも比較しようもないのですが、このコードで実際に画像を書き出している箇所（`pngs <- purrr::imap_chr(...)`）は、手もとの環境だと20秒くらいで実行できました。Rでごちゃごちゃとやっている処理のわりには、まあまあ速いと思います。

もうすこしこの作品について解説すると、上のコードで`add_circle()`に渡している点群データ`dat`の中身は、だいたい次のようになっています。

```r
n_seq <- 360

dat <-
  dplyr::tibble(
    pid = seq_len(4)
  ) |>
  dplyr::reframe(
    sid = seq_len(n_seq),
    pos = dplyr::tibble(
      x = e1071::rbridge(frequency = n_seq),
      y = e1071::rbridge(frequency = n_seq),
      z = e1071::rbridge(frequency = n_seq),
      w = 1
    ) |>
      as.matrix(),
    .by = pid
  )

dat
#> # A tibble: 1,440 × 3
#>      pid   sid  pos[,1]    [,2]   [,3]  [,4]
#>    <int> <int>    <dbl>   <dbl>  <dbl> <dbl>
#>  1     1     1  0.00335 -0.0117 0.0210     1
#>  2     1     2  0.0410  -0.0978 0.0816     1
#>  3     1     3  0.0100  -0.0763 0.147      1
#>  4     1     4  0.0346  -0.0432 0.0968     1
#>  5     1     5  0.0142  -0.117  0.0826     1
#>  6     1     6 -0.0816  -0.153  0.118      1
#>  7     1     7 -0.114   -0.239  0.0586     1
#>  8     1     8 -0.188   -0.211  0.0624     1
#>  9     1     9 -0.155   -0.248  0.0319     1
#> 10     1    10 -0.216   -0.280  0.0678     1
#> # ℹ 1,430 more rows
```

`e1071::rbridge()`は、`frequency`で1周期となるようなブラウン橋（Brownian bridge）を返す関数です。ブラウン橋というのは確率過程の一種で、`0`からスタートして、1周期後にまた`0`に戻ってくるようなランダムウォークのことを言います。

`dat$pos`は、360ステップかけて、XYZの各方向についてランダムに移動してから`0`に帰ってくる3次元ブラウン橋のデータを4本分縦に重ねているもので、先ほどのアニメーションに映っていた4本の青い「腕」構造は、ここで用意しているそれぞれの橋に相当します。

レンダリングループとなる`purrr::imap_chr(...)`の中では、この`dat$pos`を経過しているフレーム数に応じて適当に切り詰めたうえで、affinerパッケージの関数を使ってアフィン変換しています。とくに`rotate3d("z-axis", k)`としているので、Z軸については経過フレーム数に応じて回転していることになります。

データは、切り詰め・回転をしたうえで`add_circle()`に与えることで、「腕」ごとに黄色の円の連なりとして描画していますが、ここではdiscreteパスエフェクトをかけながらDifferenceで逐次ブレンドをしています。Skiaのdiscreteパスエフェクトは[こういうエフェクト](https://shopify.github.io/react-native-skia/docs/path-effects/#discrete-path-effect)で、ようするに図形の輪郭をランダムにブレさせるというものです。これがDifferenceで合成されることで、円どうしが重なる箇所では黄色の反転色である青色のピクセルが生じて、まるで細胞分裂しているようにうごめいて見えています。

## もっと複雑な質感をつくる

今見たのは青と白の2色だけという表現としてはシンプルな例でしたが、やろうと思えばもっと複雑な見た目のもつくれます。

たとえば、こういうのとか。

https://x.com/paithiov909/status/1937253028131930166

あるいは、こんなのとか。この一つ上の作品の質感は、実はブレンドモードだけで実現していたのですが、次の例ではピクセルシェーダーを使ったエフェクトをかけています。

https://x.com/paithiov909/status/1990665175582769639

Skiaは、ライブラリそれ自体の機能としてシェーダー言語を解釈することができます。これはSkSLというGLSLの方言みたいな言語で、このSkSLで記述したエフェクト（Runtime Effect）をShaderやImage Filterとしてパイプライン中で適用することが可能です。

skiagdではこのあたりの機能もがんばって使えるようにしているので、やろうと思えば、これだけでシェーダー芸みたいなこともできます（ただし、skiagdはCPUでレンダリングをおこなうので、本物のGPUシェーダーのように爆速で実行できたりはしません）。

たとえば、この上の作例は次のようなコードで、SkSLからコンパイルしたエフェクトを適用しながら描画していました（ｙはり長いので、読み飛ばしても大丈夫です！）

:::details 実際のRコード
``` r
library(skiagd)
use("rasengan", "%!*%")
# https://github.com/cran/cooltools から借りてきた球面調和関数のコード
source("docs/sphericalharmonics.R")

N <- 40
W <- 640L
H <- 480L

# alphaチャンネルをディザリングして上書きするエフェクト
sksl <- readLines("src/shaders/dither-alpha.sksl")
effect <- RuntimeEffect$make(paste0(sksl, collapse = "\n"))

n_frames <- 360

imgs <-
  purrr::imap_chr(seq_len(n_frames), \(t, i) {
    tau <- 2 * pi

    f <- t %% 40 / 40
    g <- rasengan::fract(cos(f * tau))
    theta <-
      rep(seq(0, tau, length.out = N) +
        rasengan::blend(-tau, tau, f), each = N)
    phi <- rep(seq(0, pi, length.out = N), times = N)

    l <- 8
    m <- 2
    Y <- sphericalharmonics(l, m, matrix(cbind(theta, phi), ncol = 2))

    r <- (1.2 + 0.3 * cos(t / n_frames * tau)) * rasengan::normalize(Y, to = c(-1, 1))
    points <-
      dplyr::tibble(
        x = r * sin(phi) * cos(theta),
        y = r * sin(phi) * sin(theta),
        z = r * cos(phi),
        w = 1
      )

    texture <-
      canvas("transparent", canvas_size = c(W, H)) |>
      add_point(
        as.matrix(points) %*%
          rasengan::lookat3d(eye = c(3 * cos(f * tau), -3, 3 * sin(f * tau)), center = c(.11, 2, .11)) %*%
          rasengan::persp3d(fovy = pi / 2.8, aspect = W / H) %!*%
          rasengan::viewport3d(W, H, 0, 40),
        group = seq_len(nrow(points)),
        color = rasengan::normalize(Y) |>
          rasengan::shift(40 * sin(f)) |>
          grDevices::hsv(.66, .88, 1) |>
          colorfast::col_to_rgb(),
        props = paint(
          canvas_size = c(W, H),
          width = 5,
          sigma = 2 + 3 * g,
          blur_style = BlurStyle$Solid,
          blend_mode = BlendMode$Plus,
        )
      )

    png <-
      canvas("gray20", canvas_size = c(W, H)) |>
      add_rect(
        matrix(c(0, 0, W, H), ncol = 4),
        props = paint(
          canvas_size = c(W, H),
          shader = Shader$from_picture(texture, TileMode$Clamp, c(W, H), diag(3)),
          image_filter =
            ImageFilter$runtime_shader(
              effect,
              list(c = 1 + 12 * g, iResolution = as.double(c(W, H)))
            ),
        )
      ) |>
      as_png(props = paint(canvas_size = c(W, H)))

    fp <- sprintf("public/pictures/%04d.png", i)
    writeBin(png, fp)
    fp
  },
  .progress = TRUE
  )
```
:::

個人的な見解ですが、従来のrtistryはggplot2で描くものであるため、質感にバリエーションを持たせるのはやや苦手です。

もちろん、上で名前を挙げたNicola Rennieさん、Danielle Navarroさん、Thomas Lin Pedersenさんあたりは「訓練された」ジェネラティブアーティストなので、おそらくパーリンノイズやシンプレックスノイズを上手に使いながら、ザラザラしたサンドアートみたいなテクスチャなどを実現しています。しかし、データ駆動型のジェネラティブアートでは、そうした質感を実現するために、その質感と結びつくようなデータから用意したりする必要があるわけで、そのあたりを攻略するにはたぶん相当な訓練が必要です。

その点、skiagdのようにシェーダーを扱うことができれば、狙った質感に近づける部分についてはシェーダーを使ったポストエフェクト的な処理に任せてしまえるので、データ駆動的な部分では「描くべき図形の配置を決める」という本来の目的に集中することができます。また、エフェクトがデータからある程度分離されていれば、文字通りポストエフェクトとして、後からエフェクトだけ差し替えるといったことも可能になるので、コードを書くぶんにも気楽になります。

## まとめ

この「エフェクトはポストエフェクト的なものとしてrtistryを描くためのコードとは分離して持っておいたほうがよいのでは？」というアイデアはけっこう重要で、実際、nativeRaster画像にエフェクトをかけられる仕組みはaznyanとnativeshadrというパッケージとしてskiagdとは別に用意しています。SkSLもとても強力なので、ほとんどの場合はskiagdだけで十分そうですが、凝ったエフェクトを用意できる手段がたくさんあるに越したことはないです。

また、rasenganパッケージについては、掲載したコード中にちらほら出てきていましたが、イージング関数や、点群のモデルビュー変換、その他のクリエイティブコーディングに便利な関数の詰め合わせみたいなものになっています。当初はskiagdに渡せるようなgeometricな点群データを生成する機能を中心にするつもりだったのですが、ちょっと混沌としたパッケージになってきています。

この記事では、R言語でのアート制作に挑戦するにあたって開発したパッケージについて、実際につくった作品の例と一緒に紹介しました。今回紹介したパッケージ群はまだ発展途上のものであり、Rを使ったアート制作にはきっとさらなる可能性があると思っています。来年ももっといろいろな表現にチャレンジしていくつもりなので、興味を持ってもらえたら、ぜひ一緒に遊んでみてください！　Rで絵を描く仲間が少しでも増えたら、とても嬉しいです。
