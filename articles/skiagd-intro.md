---
title: "Rで雑にクリエイティブコーディングしたい"
emoji: "🌺"
type: "idea"
topics: ["r", "creativecoding", "skia"]
published: true
---

## この記事について

こんにちは。趣味で動画をつくっています。というか、つくりたいと思っています。たとえば、こういう感じのをつくりました。

https://youtu.be/Rs6npDJ4qDY

これはこれで気に入っているのですが、もっとこう、YouTubeで見れる「文字PV」のような、なんかカッコいいやつがつくりたいです。つまり、たぶんモーショングラフィックス的なやつをつくってみたいと思っているのですが、After Effectsに課金する勇気はないし、AviUtlはちょっと合わなくて早々に挫折しました。

そこでというか、自分の場合そもそもコードを書けるのだから、クリエイティブコーディング的なことをして画像を書き出せれば、なんかカッコいい映像がつくれるのではと考えたわけです。私はR言語が好きで、Rについてはそこそこ書けるため、Rで書けたらなおのこと嬉しいと思います。

そんなわけで、Rでクリエイティブコーディング的なことをするのを念頭に、雑に画像を書き出せるパッケージを開発してみました。この記事では、いま開発している、このRパッケージについて紹介します。

https://github.com/paithiov909/skiagd


## Rで雑にクリエイティブコーディングしたい

日本でクリエイティブコーディングというと、[p5.js](https://p5js.org/)を使って素晴らしい作品をつくっている人たちがたくさんいます。コードを書いて画像や映像を生成することが目的であれば、ほかにも[Nannou](https://nannou.cc/)など、洗練されているフレームワークがすでにいくつかあります。また、クリエイティブコーディングというのとはちょっと毛色が違うかもしれませんが、私は[Remotion](https://www.remotion.dev/)が気に入っていて、ちょっとした映像素材をつくるのに使ったことがあります。

ただ、NannouもRemotionも、プロジェクトを切らないといけないのがちょっと大げさな感じがするんですよね。p5.jsはWeb EditorやOpenProcessingがあるのでその点とても手軽なんですが、私はRに慣れすぎてしまっているせいで、そもそもの話、座標の位置を計算するのにforループがわりと頻出することに煩わしさを感じます。

そこで、Rを使いたいわけです。

ところで、日本では（おそらくRユーザーのあいだでも）あまり認知されていないものの、実は、R言語でジェネレティブアートをつくっている人たちもいます。とくに英語圏のRユーザーはRという文字を大文字にすることにアイデンティティを感じているので（※個人の見解です）、Rを使ってつくられたジェネレティブアートには`aRt`とか`aRtsy`とかいったタグが付けられていることが多いです。

- [koenderks/aRtsy: aRtsy is an R package that implements algorithms for making generative art in a straightforward and standardized manner using ‘ggplot2’.](https://github.com/koenderks/aRtsy)
- [Generative Art \| Nicola Rennie](https://nrennie.rbind.io/projects/generative-art/)

こうしたR製のジェネレティブアートは、おもに[ggplot2](https://ggplot2.tidyverse.org/)を使って描かれています。ggplot2というのは、R言語の世界でデファクトスタンダードとなっているデータ可視化パッケージで、リーランド・ウィルキンソンの[The Grammar of Graphics](https://www.amazon.com/Grammar-Graphics-Statistics-Computing/dp/0387245448/ref=as_li_ss_tl)を元にした統一的な書き方によって、さまざまな図を描くことができるものです。

Rのグラフィックスデバイスの仕様それ自体が意外とリッチなのと、拡張パッケージがたくさんあるので、たとえばこんな表現もできます（なお、この例は[ggfx](https://ggfx.data-imaginist.com/)のREADMEに載っているものです）。

```r
library(ggplot2)
library(ggfx)

g <- ggplot() +
  as_reference(
    geom_polygon(aes(c(0, 1, 1), c(0, 0, 1)), colour = NA, fill = 'magenta'),
    id = "displace_map"
  ) +
  with_displacement(
    geom_text(aes(0.5, 0.5, label = 'ggfx-ggfx'), size = 25, fontface = 'bold'),
    x_map = ch_red("displace_map"),
    y_map = ch_blue("displace_map"),
    x_scale = unit(0.025, 'npc'),
    id = "text"
  ) +
  with_blend(
    geom_density_2d_filled(aes(rnorm(1e4, 0.5, 0.2), rnorm(1e4, 0.5, 0.2)),
                           show.legend = FALSE),
    bg_layer = "text",
    blend_type = "in",
    id = "blended"
  ) +
  with_shadow("blended", sigma = 3) +
  coord_cartesian(xlim = c(0, 1), ylim = c(0, 1), clip = 'off') +
  labs(x = NULL, y = NULL)

g
```

![ggfxの使用例1](https://storage.googleapis.com/zenn-user-upload/4d21e5ff6434-20250407.png)

その一方で、ggplot2はあくまでscientificなグラフを描くことを意図して開発されているものなので、一般的なドローイングライブラリと比べると、その書き心地はかなり異なります。たとえば、この上の例でもそうですが、とくに何も指定しないとggplot2はプロットパネルの軸ラベルといった要素を自動的に追加します。そのため、アートっぽい画像として保存したい場合、それらを消しつつ、パネルをプロットエリアいっぱいに広げたりといった調整が必要です。次のような感じですね（このスニペットは[Getting started with generative art \| Nicola Rennie](https://nrennie.rbind.io/blog/getting-started-generative-art/)で紹介されているものを借りたものです）。

```r
bg_col <- "transparent"
g +
  # scale_size(range = c(0.3, max_size)) +
  theme_void() +
  theme(
    plot.background = element_rect(
      fill = bg_col, colour = bg_col
    ),
    legend.position = "none",
    plot.margin = unit(c(0, 0, 0, 0), "cm")
  )
```

![ggfxの使用例2](https://storage.googleapis.com/zenn-user-upload/2eec5887f0e8-20250407.png)

だから、もっとこう「グラフというよりは画像ファイルを書き出したい！」みたいなケースでは、それ用のパッケージを新たに用意したほうが便利かなと考えました。

ちなみに、R言語は、医療画像処理などの分野においても使われることがあるので、画像処理ができるパッケージはいくつかあります。

- [Introduction to EBImage](https://bioconductor.org/packages/release/bioc/vignettes/EBImage/inst/doc/EBImage-introduction.html)
- [imager: an R package for image processing](https://dahtah.github.io/imager/imager.html)
- [Advanced Graphics and Image-Processing in R • magick](https://docs.ropensci.org/magick/)

ただ、私の場合、線とか円とかを描画したいだけで、画像処理がしたいわけではないので、このあたりはちょっと違うかなと思いました。また、gridというcoreパッケージを使うことで、低水準なAPIを介して基本的な図形を描画することもできるのですが、それはさすがに低水準すぎて扱いづらいので、それも違うかなと。

## skiagdパッケージ

そんなわけで、ffmpegやその他のツールを使って画像から動画ファイルをつくることを念頭に、Rっぽい書き心地で、雑に画像（アルファチャンネル付きのPNGファイル）を書き出せる、[skiagd](https://github.com/paithiov909/skiagd)というパッケージをつくりました。

skiagdは、SkiaをRust向けにラップしている[skia-safe](https://github.com/rust-skia/rust-skia)というRustクレートを、[savvy](https://github.com/yutannihilation/savvy)を使いつつ、R向けにさらにラップしたものです。そもそもSkiaをRから使いたかったのですが、C++のコードを直接ラップしてしまうと、Rパッケージのビルド時にSkia本体もビルドしなければいけないのが大変そうすぎたので、prebuilt binariesを提供してくれるskia-safeを使っています。欠点として、R向けの適切なprebuilt binaryが存在しないWindows環境ではビルドできないため、Unix系の環境でしか使えません。

まだ絶賛開発中ですが、基本的な図形はだいたい描けるはずです。たとえば、次のような雰囲気で使えます。

```r
library(skiagd)
library(affiner)

# グラフィックデバイスを明示的に開いておく場合、次のようなことをする
# ragg::agg_png("test.png", width = 848, height = 480)

size <- dev_size()
deg2rad <- function(deg) deg * (pi / 180)

mat <-
  dplyr::tibble(
    i = seq_len(360),
    r = 120 * abs(sin(deg2rad(4 * i))),
    x = r * cos(deg2rad(360 * i / 360)) + size[1] / 2,
    y = r * sin(deg2rad(360 * i / 360)) + size[2] / 2,
    d = 1
  ) |>
  dplyr::select(x, y, d) |>
  as.matrix()

trans <-
  transform2d() %*%
  translate2d(
    -size[1] / 2,
    -size[2] / 2
  ) %*%
  scale2d(1.2) %*%
  translate2d(
    size[1] / 2,
    size[2] / 2
  )

canvas("transparent") |>
  add_point(
    mat %*% trans,
    props = paint(
      color = "violetred",
      width = 8,
      point_mode = PointMode$Lines
    )
  ) |>
  draw_img()
```

![skiagdの使用例](https://storage.googleapis.com/zenn-user-upload/577ca9d7fa95-20250407.png)

skiagdでは、Skiaのキャンバスのデフォルトのサイズなどは、グラフィックスデバイスの設定を参照して決まりますが、画像を描く仕組みとしてはRのグラフィックスデバイスとはまったく別の物なので、あくまで設定を参照するだけです。図形を描画する関数は、描画する要素の属性（位置とか、円なら半径とか）をベクトル（行列）として受け取るほか、`props`という引数に`paint()`という関数を介して設定を与えてやることで、図形の色などの属性を変更できます。

skiagdの描画関数は、Skiaの[picture](https://shopify.github.io/react-native-skia/docs/shapes/pictures/)をrawベクトルとして第一引数に取り、戻り値として新たに図形を描き加えたpictureのrawベクトルを返します。このデザインは、毎回データのコピーが発生しているので描画処理としては遅いですが、データのシリアライズをせずに雑に中間オブジェクトを使いまわせるのと、Rで多用されるパイプを使った書き方とよく馴染むので、コードの見た目がRらしくなる点で気に入っています。

pictureは`draw_img()`するとRのグラフィックスデバイス上にプロットできるほか、この後の例のように`as_png() |> writeBin()`することで、ふつうのPNGファイルを書き出すこともできます。

## 画像をffmpegで動画にしてみる

試しに、上のバラ曲線をアニメーションさせてみましょう。

イメージとしては、次のp5.jsのコードのようなことをしようとしています（本当のことをいうと、これは順序が逆で、実際には先にRのコードを書いたうえで、似たようなことをするp5.jsのコードをChatGPTに考えてもらいました）。

:::details p5.jsのコード

```js
let totalFrames = 200;
let k = 4; // バラ曲線のパラメータ
let maxAngle = 90; // 回転角
let nCircles = 90;
let circles = [];

function setup() {
  createCanvas(800, 500);
  angleMode(DEGREES);
  frameRate(60);
  noFill();

  // 背景ドットの初期化
  for (let i = 0; i < nCircles; i++) {
    circles.push({
      x: random(width),
      y: random(height * 2),
      r: random(8, 16),
      speed: random(0.2, 0.6)
    });
  }
}

function draw() {
  background(250, 245, 230); // 背景色
  // 経過時間を正規化（0〜1）
  let t = constrain(frameCount / totalFrames, 0, 1);

  // --- 背景の水色ドット（MULTIPLYモードを限定的に適用）---
  push();
  blendMode(MULTIPLY);
  noStroke();
  fill(100, 200, 255, 200); // 水色
  for (let c of circles) {
    let y = c.y - (height / 2) * t * c.speed;
    ellipse(c.x, y, c.r * 2, c.r * 2);
  }
  pop(); // ← ここでブレンドモードを元に戻す

  // --- 前景のバラ曲線 ---
  translate(width / 2, height / 2);

  let rotationAngle = t * maxAngle;
  rotate(rotationAngle);

  stroke(255, 0, 255);
  strokeWeight(4);
  noFill(); // ← 塗りなしを明示
  drawingContext.setLineDash([10, 10]);

  beginShape();
  for (let angle = 0; angle < 360; angle++) {
    let r = 100 * sin(k * angle);
    let x = r * cos(angle);
    let y = r * sin(angle);
    vertex(x, y);
  }
  endShape(CLOSE);

  if (frameCount === totalFrames) {
    noLoop();
  }
}
```

:::


バラ曲線だけだと見た目が寂しかったので、背景に水色のドットを描いて、適当に上昇させています。

これは、skiagdを使うと、だいたい次のように書けます。上のp5.jsの例では使っていませんが、ここでは、バラ曲線の回転アニメーションにイージングを設定しています。

```r
n_circles <- 90

circles <-
  dplyr::tibble(
    i = seq_len(n_circles),
    x = runif(n_circles, 0, size[1]),
    y = runif(n_circles, 0, size[2] * 2),
    r = runif(n_circles, 8, 16),
    speed = runif(n_circles, 0.2, 0.6)
  )

duration_in_frames <- 25 * 8

for (frame in seq_len(duration_in_frames)) {
  # フレーム番号から値を補間しておく
  t <- tweenr::tween_at(0, 1, frame / duration_in_frames, ease = "linear")
  angle <- tweenr::tween_at(0, 90, frame / duration_in_frames, ease = "bounce-out")

  # アフィン行列
  trans <-
    transform2d() %*%
    translate2d(
      -size[1] / 2,
      -size[2] / 2
    ) %*%
    rotate2d(angle, "degrees") %*%
    scale2d(1.2) %*%
    translate2d(
      size[1] / 2,
      size[2] / 2
    )

  canvas("#fffae9") |>
    # 背景の円
    add_circle(
      circles |>
        dplyr::mutate(,
          y = y - (size[2] / 2 * t) * speed
        ) |>
        dplyr::select(x, y) |>
        as.matrix(),
      radius = dplyr::pull(circles, r),
      props = paint(
        color = "skyblue",
        blend_mode = BlendMode$Multiply,
      )
    ) |>
    # バラ曲線
    add_point(
      mat %*% trans,
      props = paint(
        color = "violetred",
        width = 8,
        point_mode = PointMode$Lines,
        blend_mode = BlendMode$HardLight,
      )
    ) |>
    as_png() |>
    writeBin(
      # ffmpegで動画にするために連番の名前にする
      paste0("pictures/", stringr::str_pad(frame, 4, "left", "0"), ".png")
    )
}
```

ポイントとしては、skiagdでは入力に使う点群として行列を想定しているので、点群を移動したいときにはn行3列の行列を用意しておいて、アフィン行列を右からかけることによって、点群の座標をまとめてアフィン変換するみたいなことをしています。

こうして書き出した一連のPNGファイルについて、`ffmpeg -i pictures/%04d.png -c:v libx264 out/output.mp4`として動画にしたのが、次の動画です。

https://youtu.be/D49B1ACk7To

ちなみに、skiagdで`as_png()`して得られるPNG画像にはアルファチャンネルが付いているので、`-vcodec qtrle -pix_fmt argb`とかすると、背景が透明な`.mov`ファイルとかを書き出せたりもします。私はもともと、映像素材をどうにかして用意しておいて、後から動画編集ソフトに持ち込んで動画にするみたいなことを想定していたので、アルファチャンネル付きの素材を雑につくれるのは便利そうかなと思っています。

## まとめ

ここでは、私が開発しているskiagdというRパッケージについて紹介しました。使い方の詳しい説明はしませんでしたが、こういうのがつくれるんだというざっくりとしたイメージはもってもらえたかなと思います。

Skiaはいろいろなことができる大きなライブラリであるため、skiagdにもそれらをラップした機能をいろいろ追加していきたいと考えています。skiagd以外のパッケージとも組み合わせることで、Rでもカッコいいアニメーションをつくれるようになったらいいなと思います。

興味があったら、試してみてもらえると嬉しいです。
