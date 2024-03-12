---
title: "Rで日本語テキストをいい感じに折り返す方法"
emoji: "🗾"
type: "tech"
topics: ["r", "css", "quarto"]
published: true
---

## この記事で説明すること

この記事では、Rで長めの日本語テキストの「いい感じの位置」に改行を入れて折り返す方法を紹介する。

次にあげるブログ記事が見つかるように、まれにこういった需要があるらしいのだが、これは日本語などスペースによって単語が区切られていない言語に特有の問題のように思われているため、インターネットで調べても有用な情報にアクセスしづらい。あと、この手の情報はR言語に関連する知識とはほぼ関係ないので、そもそも知っている人がいなさそう。

1. [ggplotのfacet日本語テキストを折り返す - まずは蝋の翼から。](https://knknkn.hatenablog.com/entry/2020/06/19/191317)
2. [gtパッケージの表を便利にするための小技 - 戯言日記](https://doubtpad.hatenablog.com/entry/2023/12/21/000509)
3. [nonentity data scientist - QuartoのReveal.js上でBudouXを利用できる拡張機能を作ってみた](https://statditto.com/posts/budoux/)

## ggplot2のグラフなどで日本語テキストを折り返す

とりあえず、適当な日本語文字列を含むデータを用意する。`audubon::polano`は、宮沢賢治の「ポラーノの広場」という小説を段落ごとに一つの要素とした文字列ベクトル。ここでは、そのうち3つの要素だけを`tidytext::unnest_tokens()`で適当に分かち書きして、`dplyr::count()`で段落内での単語（のようなもの）の出現頻度の列をつくっている。

```{r}
dat <-
  tibble::tibble(
    doc_id = 4:6,
    text = audubon::polano[doc_id]
  ) |>
  tidytext::unnest_tokens(token, text, drop = FALSE) |>
  dplyr::count(text, token, name = "freq")

dat
```

このデータについて、とくに意味のあるグラフではないが、次のように`text`列でfacetして散布図を描く。すると、長いテキストについては一部が見きれてしまう。

```r
library(ggplot2)

dat |>
  ggplot(aes(x = token, y = freq)) +
  geom_jitter(show.legend = FALSE) +
  facet_wrap(~ text) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

![Rplot-1](https://storage.googleapis.com/zenn-user-upload/0e5d590350a2-20240312.png)

ここで、次のことを実現したいとする（X軸の軸ラベルもなかなか見栄えが悪いが、それはここでは無視する）。

- こうした見きれている日本語テキストに「いい感じの位置」で改行を挿入して、折り返す
- そもそも長すぎる文字列については、適当な字数で切り詰めてしまって、すべては表示しない

これを実現するためには、たとえば次のような関数を用意すればよい。

```{r}
strj_segment <- \(x, engine = c("icu", "budoux")) {
  engine <- match.arg(engine, choices = c("icu", "budoux"))
  if (engine == "budoux") {
    audubon::strj_segment(x)
  } else {
    stringi::stri_split_boundaries(
      x,
      opts_brkiter = stringi::stri_opts_brkiter(locale = "ja@lw=phrase;ld=auto")
    )
  }
}
facet_splitter <- \(x, wrap = 21, trunc = 50, temp_char = "\u200b", collapse = "\n") {
  reg_pat <- paste0("(.{1,", wrap, "})")
  dplyr::mutate(
    x,
    across(where(is.character),
    ~ strj_segment(.) |>
      purrr::map_chr(\(.x) paste0(.x, collapse = temp_char)) |>
      stringr::str_extract_all(reg_pat) |>
      purrr::map_chr(\(.x) stringr::str_remove_all(.x, temp_char) |> paste0(collapse = collapse)) |>
      stringr::str_trunc(trunc))
  )
}
```

`facet_splitter`を`ggplot2::facet_wrap()`の`labeller`引数に渡すと、おおむね期待したような挙動をするはず。

```r
dat |>
  ggplot(aes(x = token, y = freq)) +
  geom_jitter(show.legend = FALSE) +
  facet_wrap(~ text, labeller = facet_splitter) +
  theme(axis.text.x = element_text(angle = 45, hjust = 1))
```

![Rplot-2](https://storage.googleapis.com/zenn-user-upload/55977164ef6b-20240312.png)

## gtパッケージの表のセル内で日本語テキストを折り返す

そういう需要もあるらしい。まずそもそも長すぎるテキストについては`stringr::str_trunc()`であらかじめ切り詰めておくのがオススメ。

```r
dat |>
  dplyr::mutate(text = stringr::str_trunc(text, 120)) |>
  dplyr::slice_head(n = 1, by = text) |>
  gt::gt()
```

![gt テキスト分割なし](https://storage.googleapis.com/zenn-user-upload/6e46fcbf2f10-20240312.webp)

これを「いい感じの位置」で折り返すには、ようするに「いい感じの位置」で改行されるようなHTML文字列をつくってから、`gt::md()`に渡すとよいらしい。たとえば、次のような関数を用意するとよい。

```{r}
cell_splitter <- \(x) {
  strj_segment(x) |>
    purrr::map(~
      paste0(
        '<span style="word-break: keep-all; overflow-wrap: anywhere;">',
        paste0(., collapse = "<wbr />"),
        "</span>",
        collapse = ""
      )
    ) |>
    purrr::map(gt::md)
}
```

これを次のように使うと、たぶん「いい感じの位置」で改行されるようになる。なお、この方法では「いい感じの位置」をすべてspanタグでくくりながら保持しているので、HTMLウィジェットとして表示している場合、表の表示幅にあわせて折り返し位置も変化する。

```r
dat |>
  dplyr::mutate(text = stringr::str_trunc(text, 120)) |>
  dplyr::slice_head(n = 1, by = text) |>
  dplyr::mutate(
    across(where(is.character), ~ cell_splitter(.))
  ) |>
  gt::gt()
```

![gt テキスト分割あり](https://storage.googleapis.com/zenn-user-upload/82dfd6da5afb-20240312.webp)

## Quartoの見出しや地の文で日本語テキストを折り返す

ここでは、HTMLとしてレンダリングする場合を想定している。Word文書やLaTeXとして出力する場合にどうすればよいのかはよくわからないが、それはどちらかというと最初に紹介した改行を挿入するケースが参考になるのではないかと思う。

HTMLとしてレンダリングする場合、冒頭に貼った「[nonentity data scientist - QuartoのReveal.js上でBudouXを利用できる拡張機能を作ってみた](https://statditto.com/posts/budoux/)」で紹介されている拡張機能を使えば、JavaScriptで「いい感じの位置」に区切りを挿入することができる。

あるいは、古いブラウザでの表示に対応する必要がない場合、最近のブラウザではCSSだけでも同じような位置で自動的に改行されるようにすることができる。たとえば、Quarto（Presentations）のタイトルスライド要素（`#title-slide`）内においてこのような区切り位置で改行されるようにしたい場合、YAMLフロントマターで`lang: ja`を指定したうえで、次のようなスタイルを書いたSCSSファイルをincludeするとよい。

```scss
/*-- scss:rules --*/
#title-slide {
  word-boundary-detection: auto(ja);
  word-break: auto-phrase;
}
```

CSSやSCSSをincludeする方法については、出力フォーマットによっていくつかの方法があるようなので、ここでは紹介しない。詳しくはQuartoのドキュメントを読んでもらいたい。

注意点として、`word-break: auto-phrase;`はこの記事を書いている2024年3月の時点でSafariやFirefoxではサポートされていない。このCSSプロパティのサポート状況については次のページなどで確認できる。サポートされていないブラウザで同じような表示をしたい場合には、拡張機能を使ったほうがよい。

- ["auto-phrase" | Can I use... Support tables for HTML5, CSS3, etc](https://caniuse.com/?search=auto-phrase)

## この文節区切りは何なのかという技術的な話

この記事で紹介したような「長めの日本語テキストの「いい感じの位置」に改行を入れて折り返す」という課題は、Web技術の界隈で「日本語改行問題」としてしばしば言及されるもの。これをいい感じに実現するマークアップの方法については、簡単にググると次のような記事がヒットする。

- [Webブラウザの日本語改行問題 -改行を実現するHTML/CSS-（1） #JavaScript - Qiita](https://qiita.com/tamanyan/items/e37e76b7743c59235995)

この記事でも紹介されているように、「いい感じの位置」に区切りを与えるタスクそれ自体については、[BudouX](https://github.com/google/budoux/)というJavaScriptライブラリが開発されていて、これで解決できる。[READMEの説明](https://github.com/google/budoux/?tab=readme-ov-file#how-it-works)によると、このライブラリでは、AdaBoostを用いて各々の文字境界について区切り位置にあたるかを2値分類するように学習したモデルを利用しているらしい。

このBudouXをもとにしている実装が、実はICUに取り入れられていて、[ICU 73.2](https://icu.unicode.org/download/73#h.f7y1p2b6nduw)からすでに利用できるようになっている。その経緯については、次のWebページでなんとなくうかがい知ることができる。

- [ICU で文節単位の改行をするための提案](https://zenn.dev/komatsuh/scraps/b860963b65083b)

この日本語テキストに文節区切りを与える機能は、どうやらChromiumベースのブラウザにおいて、先ほど書いたようなCSSプロパティを使ってこれを利用できるようにすることを念頭に取り入れられたものらしい。

R言語の文脈でいうと、BudouXについては従来から、筆者が開発しているRパッケージ[audubon](https://github.com/paithiov909/audubon)の`audubon::strj_segment()`という関数を使って、[V8](https://cran.r-project.org/package=V8)経由でJSスクリプトを呼ぶことで利用することができた。一方で、同じことを実現する機能がICU4Cにすでに取り入れられているため、この記事ですでに見たように、`stringi::stri_split_boundaries()`を使っても、同様の文節区切りを与えることができるようになった。

ただし、ICU 73.2というのは比較的新しいバージョンであるため、stringiをビルドするときにリンクされたICU4Cのバージョンによっては、この機能が使えず、期待したような出力が得られない可能性がある。その場合には、stringi 1.8.1のソースパッケージにはすでにICU 74.1がバンドルされているため、以下のページで案内されている方法にしたがって、バンドルされているICU4Cを使ってビルドされるように設定するとよい。

- [Installing stringi - R Package stringi](https://stringi.gagolewski.com/install.html)
