---
title: "そのpurrr::map() 本当に必要ですか？"
emoji: "📏"
type: "idea"
topics: ["r"]
published: false
---

## この記事について

[vctrs](https://vctrs.r-lib.org/)というRパッケージがあります。`install.packages("tidyverse")`するとインストールされるパッケージの一つで、私たちが意識する必要のない部分においてではあるものの、dplyrなどの関数のなかで使われています。

vctrsそのものは、おもにRパッケージの開発者が使うことを想定されているパッケージ（a developer-focussed package）です。実際、vctrsを一般のRユーザーが使う機会はほとんどないでしょうし、dplyrやpurrrの関数のように、普段づかいに便利な機能ばかり提供しているわけでもありません。

一方で、vctrsには一般のRユーザーでも知っていると便利だろう関数がいくつかあり、そうした関数を知っていると、特定のシーンでパフォーマンスに配慮した書き方をするのに役立ちます。この記事では、とくに`vctrs::list_sizes()`と`vctrs::vec_chop()`を取り上げ、`purrr::map()`などの関数の中でデータフレームをつくるパターンにおいて、これらの関数を使いながら書き換えできる例を紹介します。

## purrr::mapのアンチパターン？

ここで考えるのは、シンプルな例では次のようなパターンです。`purrr::map_dfr()`を使えるようなパターンで、同じ構造のリストを持っていたとして、その各要素をデータフレームに詰めこんだうえで、最後に縦に結合してまとめあげたいとします。

```r
use_map <- \(text = c("新しい朝が来た 希望の朝だ", "喜びに胸を開け 大空あおげ",
                      "ラジオの声に 健やかな胸を", "この香る風に 開けよ", "それ 一二三")) {
  stringr::str_split(text, "[^[:alnum:]]+", n = 2) |>
    purrr::map(\(x) {
      data.frame(
        s1 = x[1],
        s2 = x[2]
      )
    }) |>
    dplyr::bind_rows() |>
    dplyr::as_tibble()
}
use_map()
#> # A tibble: 5 × 2
#>   s1             s2
#>   <chr>          <chr>
#> 1 新しい朝が来た 希望の朝だ
#> 2 喜びに胸を開け 大空あおげ
#> 3 ラジオの声に   健やかな胸を
#> 4 この香る風に   開けよ
#> 5 それ           一二三
```

先にこの記事にとってあまり本質的でないtipsを指摘しておくと、このようにlist of data.framesをつくってから一つのtibbleにまとめたい場合、`purrr::map()`のなかでtibbleをつくって結合するよりも、結合してから`as_tibble()`したほうが速くなります。

いずれにせよ、このコードは無駄なことをしています。もちろん、だからただちにダメなコードであるというつもりはありません。`purrr::map()`を使ったコードの良い点の一つは、みんながだいたい似たような書き方をするから読みやすいところで、このコードも何をしているのかが一見してわかりやすいという点では、悪いコードだとは思いません。

しかし、とにかく速く処理したいというだけなら、これは次のように書いたほうが速くできることがあります。

```r
use_transpose <- \(text = c("新しい朝が来た 希望の朝だ", "喜びに胸を開け 大空あおげ",
                            "ラジオの声に 健やかな胸を", "この香る風に 開けよ", "それ 一二三")) {
  stringr::str_split(text, "[^[:alnum:]]+", n = 2) |>
    purrr::list_transpose() |>
    (\(x) {
      dplyr::tibble(
        s1 = unlist(x[1], use.names = FALSE),
        s2 = unlist(x[2], use.names = FALSE)
      )
    })()
}
use_transpose()
#> # A tibble: 5 × 2
#>   s1             s2
#>   <chr>          <chr>
#> 1 新しい朝が来た 希望の朝だ
#> 2 喜びに胸を開け 大空あおげ
#> 3 ラジオの声に   健やかな胸を
#> 4 この香る風に   開けよ
#> 5 それ           一二三
```

「できることがあります」というのは、もちろん速くならないケースもあるからで、与えるデータとか処理の内容にもよるのですが、少なくともこの例では、`use_transpose()`のほうがやや速いです。

```r
microbenchmark::microbenchmark(
  use_map(),
  use_transpose(),
  times = 100,
  check = "equal"
)
#> Unit: microseconds
#>             expr      min       lq      mean    median        uq      max neval
#>        use_map() 1086.731 1102.798 1127.5265 1115.6380 1134.3345 1306.771   100
#>  use_transpose()  842.432  851.032  914.0257  856.7855  872.5255 2994.578   100
```

よく知られているように、もとからvectorizeされている関数を使うときには、map系の関数やループのなかで一つ一つの要素に対して呼び出すよりも、vectorizeされているものとして呼び出すほうが速くなりやすいです。（vectorizeされているというのかはよくわからないものの）`data.frame()`にはベクトルを渡せるので、この例のように列になるべき各要素をベクトルにまとめあげてしまってから、一度だけ渡してしまったほうが速くなりやすいと思います。

この例はシンプルなケースですが、原則として、グループごと（リストの要素ごと）に実行しなかった場合と結果が同じになる処理については、素直にグループごとに繰り返すのではなく、入力をまとめてから呼び出したほうが速くなるケースが多いと思います。もっとも、当たり前ですが、本当にグループごとに実行したい処理（グループごとに実行するのと、まとめあげた全体に対して実行するのとでは意味が違うような処理）の場合には、このような書き方はできません。一方で、そうではないケースはまさにこのパターンによる書き換えが可能であり、実のところは`purrr::map_dfr()`みたいなことをする必要はなかったりします。

## グループの名前を保持したいパターン

とはいっても、上のようなパターンには書き換えづらいケースもあります。たとえば、`purrr::map()`ではなく`purrr::imap()`を使っていて、グループの名前などを保持したい場合です。

もう少しだけ複雑な例を考えてみましょう。たとえば、次のような構造のリストを持っていたとします。

```r
dat <-
  modeldata::tate_text |>
  dplyr::mutate(medium = as.character(medium)) |>
  dplyr::group_by(artist) |>
  dplyr::group_map(~ {
    list(
      title = .x$title,
      medium = .x$medium,
      year = .x$year
    )
  }) |>
  rlang::set_names(
    levels(modeldata::tate_text$artist)
  )

str(dat[1:2])
#> List of 2
#>  $ Absalon    :List of 3
#>   ..$ title : chr [1:6] "Proposals for a Habitat" "Cell No. 1" "Solutions" "Noises" ...
#>   ..$ medium: chr [1:6] "Video, monitor or projection, colour and sound (stereo)" "Wood, fibreboard, fabric and fluorescent lights" "Video, monitor or projection, colour and sound (mono)" "Video, monitor or projection, colour and sound" ...
#>   ..$ year  : num [1:6] 1990 1992 1992 1993 1993 ...
#>  $ Abts, Tomma:List of 3
#>   ..$ title : chr [1:6] "Noeme" "Untitled # 1" "Untitled no. 6" "Untitled no. 8" ...
#>   ..$ medium: chr [1:6] "Oil paint and acrylic paint on canvas" "Graphite, coloured graphite and ink on paper" "Graphite on paper" "Graphite on paper" ...
#>   ..$ year  : num [1:6] 2004 2004 2008 2008 2008 ...
```

このリストは[modeldata::tate_text](https://modeldata.tidymodels.org/reference/tate_text.html)から加工したもので、Tate Galleryが管理しているアート作品について、作者ごとに`title`, `medium`,
`year`という要素をもつ名前付きリストのリスト（list of named lists）になっています。1階層めの各リストの名前（`Absalon`,
`Abts, Tomma`…）が作者名です。

これを次のようにしてtibbleのかたちに戻すコードを、上で見たのと同じようなパターンで書き換えるにはどうすればいいでしょうか。

```r
use_imap <- \(input) {
  purrr::imap(input, \(x, y) {
    data.frame(
      name = y,
      title = x$title,
      medium = x$medium,
      year = x$year
    )
  }) |>
    purrr::list_rbind() |>
    dplyr::as_tibble()
}
use_imap(dat)
#> # A tibble: 4,284 × 4
#>    name        title                   medium                               year
#>    <chr>       <chr>                   <chr>                               <dbl>
#>  1 Absalon     Proposals for a Habitat Video, monitor or projection, colo…  1990
#>  2 Absalon     Cell No. 1              Wood, fibreboard, fabric and fluor…  1992
#>  3 Absalon     Solutions               Video, monitor or projection, colo…  1992
#>  4 Absalon     Noises                  Video, monitor or projection, colo…  1993
#>  5 Absalon     Battle                  Video, monitor or projection, colo…  1993
#>  6 Absalon     Assassinations          Video, monitor or projection, colo…  1993
#>  7 Abts, Tomma Noeme                   Oil paint and acrylic paint on can…  2004
#>  8 Abts, Tomma Untitled # 1            Graphite, coloured graphite and in…  2004
#>  9 Abts, Tomma Untitled no. 6          Graphite on paper                    2008
#> 10 Abts, Tomma Untitled no. 8          Graphite on paper                    2008
#> # ℹ 4,274 more rows
```

これは、たとえば、次のように書くことができます。

``` r
use_transpose <- \(input) {
  input_t <- purrr::list_transpose(input)
  dplyr::tibble(
    name = rep(names(input), vctrs::list_sizes(input_t$title)),
    title = unlist(input_t$title, use.names = FALSE),
    medium = unlist(input_t$medium, use.names = FALSE),
    year = unlist(input_t$year, use.names = FALSE)
  )
}
use_transpose(dat)
#> # A tibble: 4,284 × 4
#>    name        title                   medium                               year
#>    <chr>       <chr>                   <chr>                               <dbl>
#>  1 Absalon     Proposals for a Habitat Video, monitor or projection, colo…  1990
#>  2 Absalon     Cell No. 1              Wood, fibreboard, fabric and fluor…  1992
#>  3 Absalon     Solutions               Video, monitor or projection, colo…  1992
#>  4 Absalon     Noises                  Video, monitor or projection, colo…  1993
#>  5 Absalon     Battle                  Video, monitor or projection, colo…  1993
#>  6 Absalon     Assassinations          Video, monitor or projection, colo…  1993
#>  7 Abts, Tomma Noeme                   Oil paint and acrylic paint on can…  2004
#>  8 Abts, Tomma Untitled # 1            Graphite, coloured graphite and in…  2004
#>  9 Abts, Tomma Untitled no. 6          Graphite on paper                    2008
#> 10 Abts, Tomma Untitled no. 8          Graphite on paper                    2008
#> # ℹ 4,274 more rows
```

このコードのポイントは`vctrs::list_sizes()`です。この関数は、次のように、リストの中身のその階層における長さを名前付きベクトルとして返します。

```r
vctrs::list_sizes(
  list(three = 1:3, five = 1:5, ten = 1:10, one = list(1:10), two = list("a", NA))
)
#> three  five   ten   one   two
#>     3     5    10     1     2
```

ここでは、もとのリストの名前（作者名）をそれぞれに対応するリストの長さ分だけ繰り返さないといけないため、`rep(names(input), vctrs::list_sizes(input_t$title))`として作者名の列（`name`）になるベクトルをつくっています。このデータは先ほどの例と比べるとそこそこのサイズがあるので、明らかに`use_transpose()`のほうが速くなっていることがわかると思います。

```r
microbenchmark::microbenchmark(
  use_imap(dat),
  use_transpose(dat),
  times = 10,
  check = "equal"
)
#> Unit: milliseconds
#>                expr        min         lq       mean     median         uq
#>       use_imap(dat) 167.205981 170.388488 172.245336 172.014184 173.274481
#>  use_transpose(dat)   5.055937   5.250031   5.649814   5.360375   5.662203
#>         max neval
#>  177.343791    10
#>    8.180303    10
```

## リストのまま返したいパターン

さらにいえば、最後に一つのデータフレームにまとめあげるのではなく、list of data.framesのままのかたちで返したい場合であっても、これまで見たような書き方をしてからグループごとに再度分割したほうがむしろ速いこともあります。

これもシンプルな例ですが、この下のコードは`tolower(x$title)`の部分がグループごとに実行する必要のない処理なので、これまでの例と同様にまとめあげてから実行したほうが速くできそうです。しかし、その場合、すでにグループごとに分割されているものを一度まとめあげて処理したうえで、その結果を再度グループごとに分割することになります。それはどのように書けばいいのでしょうか。

```r
use_map <- \(input) {
  purrr::map(input, \(x) {
    data.frame(
      title = tolower(x$title)
    )
  }) |>
    rlang::set_names(names(input))
}
use_map(dat) |> head(2)
#> $Absalon
#>                     title
#> 1 proposals for a habitat
#> 2              cell no. 1
#> 3               solutions
#> 4                  noises
#> 5                  battle
#> 6          assassinations
#>
#> $`Abts, Tomma`
#>             title
#> 1           noeme
#> 2    untitled # 1
#> 3  untitled no. 6
#> 4  untitled no. 8
#> 5 untitled no. 10
#> 6            zebe
```

たとえば、`vctrs::vec_chop()`という関数を使って、次のように書くことができます。この関数は`base::split()`と似たようなことができる関数で、`sizes`引数に各グループの長さをベクトルとして与えると、与えたデータを先頭から順にその長さにくぎった結果を返します。

```r
use_transpose <- \(input) {
  input_t <- purrr::transpose(input)
  data.frame(
    title = unlist(input_t$title, use.names = FALSE) |>
      tolower()
  ) |>
    vctrs::vec_chop(sizes = vctrs::list_sizes(input_t$title)) |>
    rlang::set_names(names(input))
}
use_transpose(dat) |> head(2)
#> $Absalon
#>                     title
#> 1 proposals for a habitat
#> 2              cell no. 1
#> 3               solutions
#> 4                  noises
#> 5                  battle
#> 6          assassinations
#>
#> $`Abts, Tomma`
#>             title
#> 1           noeme
#> 2    untitled # 1
#> 3  untitled no. 6
#> 4  untitled no. 8
#> 5 untitled no. 10
#> 6            zebe
```

ここまでこの記事を読んできていると、この`use_map()`はさすがに遅いだろうとわかりきっているような例ですが、実際、`use_transpose()`のほうが速くなることがわかります。

```r
microbenchmark::microbenchmark(
  map = use_map(dat),
  vctrs = use_transpose(dat),
  times = 10,
  check = "equal"
)
#> Unit: milliseconds
#>   expr       min        lq      mean    median        uq       max neval
#>    map 68.298803 68.801993 70.052978 69.628583 70.724475 74.074373    10
#>  vctrs  3.365909  3.388804  3.939303  4.036676  4.249901  4.644325    10
```

## 試すときは実際に時間を測ろう！

この記事では、同じ構造のリストからその各要素をデータフレームに詰めこむような例を念頭に、`purrr::map()`の中で処理を繰り返すよりも、vctrsの関数などを使って書くほうが速いケースがあることを紹介しました。

注意してほしいのが、この記事で指摘したいのは、あくまで「コストの高い処理はできれば繰り返さないほうがよい」ということに尽きるということです。この記事で紹介した例は、いずれも`purrr::map()`のなかで文字列処理をしたり、データフレームをつくってからまとめあげたりといった、比較的コストの高い処理を繰り返しているものでした。

一方で、単純な算術計算など、それ自体のコストが小さい処理については、この記事で紹介したようなパターンで書くよりも、シンプルに`purrr::map()`のなかで繰り返してしまったほうが速いケースもあります。たとえば、次のような例です。

```r
use_map <- \(input) {
  purrr::map(input, \(x) log(x$year)^pi) |>
    rlang::set_names(names(input))
}

use_transpose <- \(input) {
  input_t <- purrr::transpose(input)
  log(unlist(input_t$year, use.names = FALSE))^pi |>
    vctrs::vec_chop(sizes = vctrs::list_sizes(input_t$title)) |>
    rlang::set_names(names(input))
}

microbenchmark::microbenchmark(
  map = use_map(dat),
  vctrs = use_transpose(dat),
  times = 10,
  check = "equal"
)
#> Unit: microseconds
#>   expr      min      lq     mean    median       uq      max neval
#>    map  749.036  757.46 1074.170  773.8225  783.917 3787.647    10
#>  vctrs 1059.168 1074.10 1488.174 1090.6915 1165.816 4986.824    10
```

このように、どのあたりがそのコードのボトルネックになっているのかは、与えるデータや処理の内容によって異なります。この記事で紹介したようなパターンへの書き換えを試すときには、実際に与えるデータに似た小さなサンプルデータを与えながら、必ず実行時間を測りつつ試すようにしてください。
