---
title: "Rでランダムっぽい文字列のIDを振る"
emoji: "🆔"
type: "tech"
topics: ["r"]
published: false
---


## baseだけでランダムな文字列IDを振るやり方

データフレーム中の観測単位に対して、適当なIDを与えたいときがあると思います。そうした場合、単純には`dplyr::row_number()`などで行番号を振ることによって、それをIDに代えることができます。

```r
cars <-
  dplyr::tibble(name = rownames(mtcars), mtcars) |>
  dplyr::slice_sample(n = 100, replace = TRUE) |>
  dplyr::select(1:6)

dplyr::mutate(cars, id = dplyr::row_number())
#> # A tibble: 100 × 7
#>    name               mpg   cyl  disp    hp  drat    id
#>    <chr>            <dbl> <dbl> <dbl> <dbl> <dbl> <int>
#>  1 Dodge Challenger  15.5     8 318     150  2.76     1
#>  2 Valiant           18.1     6 225     105  2.76     2
#>  3 Ferrari Dino      19.7     6 145     175  3.62     3
#>  4 Dodge Challenger  15.5     8 318     150  2.76     4
#>  5 Fiat 128          32.4     4  78.7    66  4.08     5
#>  6 Merc 280C         17.8     6 168.    123  3.92     6
#>  7 Duster 360        14.3     8 360     245  3.21     7
#>  8 Duster 360        14.3     8 360     245  3.21     8
#>  9 Mazda RX4         21       6 160     110  3.9      9
#> 10 Merc 240D         24.4     4 147.     62  3.69    10
#> # ℹ 90 more rows
```

しかし、本来は、このようにIDとして連番を振ることは避けるべきです。IDとして連番を振ってしまうと、たとえば、IDの数字が大きいほど新しいレコードであることが推測できるようになります。その結果、そうしてIDを振ったデータからサンプリングしたとしても、IDの最大値から元のデータの規模がある程度わかってしまったり、IDからレコードの順序を復元できてしまったりと、一般にIDによっては伝えられるべきでないような情報が伝達されてしまう恐れがあります。

そのため、IDは、見た目から元の順序が容易には推測できないような文字列にするべきです。実際、何かの単位にIDを与えたいというとき、私たちがやりたいことは、その単位を識別できるような名付けをおこなうことであって、番号を与えるというのはその一つのやり方にすぎません。適当な文字列によって名付けをおこなうことができるのなら、あえて番号を与える必要はないはずなのです。

しかし、適当な文字列をIDとして与えるというのは、Rだとやや書きづらい処理です。もちろん、ただの行番号の代わりであれば、データフレームに含まれるレコードの数だけ適当な文字列を生成すればいいでしょう。Rでランダムな文字列を生成する方法としては、`stringi::stri_rand_strings()`のほか、[ids](https://github.com/reside-ic/ids)、[RcppUUID](https://github.com/eddelbuettel/rcppuuid), [ulid](https://github.com/eddelbuettel/ulid)などのパッケージを使うこともできますし、（IDの衝突などについて深く考えないなら）baseだけ使って次のような感じでやることもできます。

```r
my_rand_strings <- \(n, length) {
  chrs <- c(letters, LETTERS, 0:9)
  sample(chrs, length * n, replace = TRUE) |>
    split(factor(seq_len(n))) |>
    lapply(\(s) paste0(s, collapse = "")) |>
    unlist(use.names = FALSE)
}

dplyr::mutate(cars, id = my_rand_strings(dplyr::n(), 11))
#> # A tibble: 100 × 7
#>    name               mpg   cyl  disp    hp  drat id
#>    <chr>            <dbl> <dbl> <dbl> <dbl> <dbl> <chr>
#>  1 Dodge Challenger  15.5     8 318     150  2.76 boEfuNhIbN9
#>  2 Valiant           18.1     6 225     105  2.76 ZFmHesilzhE
#>  3 Ferrari Dino      19.7     6 145     175  3.62 OBUXGQb4zxi
#>  4 Dodge Challenger  15.5     8 318     150  2.76 0Ly3fO1hSXq
#>  5 Fiat 128          32.4     4  78.7    66  4.08 B2JvFiC9JGz
#>  6 Merc 280C         17.8     6 168.    123  3.92 E7PVmpfRQkA
#>  7 Duster 360        14.3     8 360     245  3.21 s4bqsMfNyIS
#>  8 Duster 360        14.3     8 360     245  3.21 biRtFrYYTIK
#>  9 Mazda RX4         21       6 160     110  3.9  2V1CKFxnVGo
#> 10 Merc 240D         24.4     4 147.     62  3.69 3yxE5zZqs1d
#> # ℹ 90 more rows
```

一方で、これだと、たとえばグループ単位でIDを振り直したいというときに上手くできません。グループ単位でIDを振り直したい場合、つまり、同じグループについては同じIDが振られてほしいといった場合には、少なくともグループの数だけのランダムな文字列を用意しておいて、どのグループかに応じてそれらを割り当てるような処理をする必要があります。書き方はいろいろありそうですが、これはたとえば次のように書けそうです。

```r
pseudonymize <- \(x, .length = 11, .rand_strings = my_rand_strings) {
  pool <- .rand_strings(length(x[!is.na(x)]), .length)
  tab <- hashtab()
  for (i in seq_along(pool)) {
    sethash(tab, i, pool[i])
  }
  unlist(lapply(rank(x, ties.method = "min"), \(i) {
    gethash(tab, i, nomatch = NA)
  }), use.names = FALSE)
}

# nameについて仮名化
dplyr::mutate(cars, name = pseudonymize(name))
#> # A tibble: 100 × 6
#>    name          mpg   cyl  disp    hp  drat
#>    <chr>       <dbl> <dbl> <dbl> <dbl> <dbl>
#>  1 zDF2vmzaJHS  15.5     8 318     150  2.76
#>  2 nd0RCzEbFNa  18.1     6 225     105  2.76
#>  3 yvGHLGMrVFy  19.7     6 145     175  3.62
#>  4 zDF2vmzaJHS  15.5     8 318     150  2.76
#>  5 9073WPs8kRT  32.4     4  78.7    66  4.08
#>  6 24qaZdKXmOE  17.8     6 168.    123  3.92
#>  7 F4VREmpYBFQ  14.3     8 360     245  3.21
#>  8 F4VREmpYBFQ  14.3     8 360     245  3.21
#>  9 IGfJ5VaISpH  21       6 160     110  3.9
#> 10 FF0BlCJdjQz  24.4     4 147.     62  3.69
#> # ℹ 90 more rows

# グループの数が変化していないことの確認
(length(unique(cars$name)) == length(unique(pseudonymize(cars$name))))
#> [1] TRUE
```

:::details このコードについて
`utils::hashtab()`は、R>=4.2から使えるようになっている、experimentalな関数です。いわゆる連想配列を提供します。

従来のRでkey-value storeみたいなものを使いたい場合には、しばしば名前付きベクトル、リストや、環境（`new.env()`）などが代用されていましたが、ペアが多くなったときに遅かったり、[実はメモリリークする](https://r-lib.github.io/fastmap/)ことが報告されていたりと、扱いづらいものでした。

`utils::hashtab()`はそのあたりの問題を解決するもので、比較的新しい[ベンチマーク](https://randy3k.github.io/collections/articles/benchmark.html#dictionary)によると、`fastmap::fastmap()`とあまり変わらないくらいのパフォーマンスを発揮するようです。
:::


## ハッシュ関数を使うやり方

さて、上の例のように、データの中の識別子について、元の値と一対一で対応しているような別の識別子に置き換えていく処理のことを仮名化（pseudonymization）といいます。また、そうして置き換えた後の仮の識別子のことを、仮IDとか仮名IDなどと呼びます（ちなみに、仮名化ではなく、匿名化とかいった場合、「同じ準識別子をもつレコードがk件を下回らない状態になるように、属性値を一般化・かく乱することによって書き換える」といった処理を意味したりするため、用語を混同しないよう注意が必要です）。

識別子を仮名化するとき、実際にどのような方法で仮IDを用意すればよいかは、仮名化をおこなう目的によって異なります。しかし、たいていは、おそらく「個人識別符号の削除」みたいなことが目的だろうと思われるので、その場合「当該個人識別符号を復元することのできる規則性を有しない方法により他の記述等に置き換える」ようなやり方が求められます。[^1]

これはようするに、その方面の技術的な知識をもっていない一般の人が容易に復元できなければOKということらしいのですが、仮IDは元の値と一対一で対応していなければならない（一つの仮IDが複数の元の値に対応してはいけない）ことからも、仮名化をおこなう際には、何かのハッシュ関数を使って、直接識別子のハッシュ値を取ってしまうということがよくおこなわれるようです。これはRでは、次のような感じでできます。

```r
my_salt <- my_rand_strings(1, 10)
dplyr::mutate(cars, name = openssl::sha256(paste0(name, my_salt)))
#> # A tibble: 100 × 6
#>    name                                              mpg   cyl  disp    hp  drat
#>    <hash>                                          <dbl> <dbl> <dbl> <dbl> <dbl>
#>  1 7b4ac239be25aa7c156cfe3d7090fd6942c37d3efb3e35…  15.5     8 318     150  2.76
#>  2 b038910c74c1bee427eb9a0a1f30b6210f7ef77ab3d79f…  18.1     6 225     105  2.76
#>  3 8d19c2f53ce793d2283066621dc1d31af8264f0dbacaf7…  19.7     6 145     175  3.62
#>  4 7b4ac239be25aa7c156cfe3d7090fd6942c37d3efb3e35…  15.5     8 318     150  2.76
#>  5 6fdb5a092c50b9d7de397402d6bc589ac69efee7b478d4…  32.4     4  78.7    66  4.08
#>  6 f55e191090a6fe063d27fe8dd1aceab51216fb0189dd94…  17.8     6 168.    123  3.92
#>  7 314ad5f8e3b837bf74534e7985cf2e455920332935f537…  14.3     8 360     245  3.21
#>  8 314ad5f8e3b837bf74534e7985cf2e455920332935f537…  14.3     8 360     245  3.21
#>  9 dba1b506d82d1e4772c9386bcba0f6b131f43c2f2d99c2…  21       6 160     110  3.9
#> 10 d742845b93d2ca84d781c7b63fe87a324970d8f18bf15e…  24.4     4 147.     62  3.69
#> # ℹ 90 more rows
```

このあたりは、以前は[anonymizer](https://github.com/paulhendricks/anonymizer)というパッケージがCRANに公開されていて、それを使っても同じようなことができました。ただ、anonymizerは[digest](https://github.com/eddelbuettel/digest)のハッシュ関数を使っていて、文字列ではなくRオブジェクトとしてのハッシュ値を取っていたようなのですが、ここでは別に仮IDが何のハッシュであるべきかは問われていないため、ふつうに[openssl](https://github.com/jeroen/openssl)を使っておくほうが速く処理することができるはずです。


## ldccr::sqidsを使うやり方

識別子を仮名化したいという場合、多くのケースでは、仮IDから元の値へは復元できないほうがむしろ好ましいため、ふつうはここまでに紹介したようなやり方でやるのがおすすめです。

その一方で、

- 識別子のハッシュ値を取るのでは、生成される文字列が見た目的に長すぎる
- とはいえ、いちおう仮IDが重複しないことは保証されていてほしい（ただのランダムな文字列だと不安）
- また、逆に、仮IDから元の値を復元できてほしい

といった場面では、それっぽい別のライブラリを使うのが便利な場合もあります。

[sqids](https://sqids.org/)は、上にあげたような場面で利用できるライブラリです。正の整数の配列を入力として、一見するとランダムな見た目の文字列を一意に生成することができます。もっとも、これはハッシュ関数とか、何かの暗号論的な仕組みによるものではないため、生成というよりは変換みたいなものらしいです。実際、生成した文字列がsqidsであることがわかってしまえば、簡単に元の整数の配列を復元することができます。

したがって、sqidsは、元の順序を復元されてしまうと困るようなIDについて、そのままsqidsの入力にして仮名化するような用途では使えません。なんなら連番のIDがそのまま見えても問題ないのだけれど、「でもやっぱりIDは適当な文字列にしておいたほうがいいのでは？」みたいなシーンにおいて使います。

sqidsをRで使うには、[公式のR実装](https://sqids.org/r)もあるのですが、それは本当にpure Rな実装なため、CやC++の実装をパッケージに組み込んで使うのがよさそうです。[ldccr::sqids()](https://paithiov909.github.io/ldccr/reference/sqids.html)は、実際に筆者がsqidsのC++実装をRcppを使っていい感じにラップしてみたものです。この関数は[dplyr::row_number()](https://dplyr.tidyverse.org/reference/row_number.html)などの関数を参考にしているため、次のように、dplyrの関数のなかで雑に使うことができます。

```r
# `dplyr::row_number()`の代わり
dplyr::mutate(cars,
  id = ldccr::sqids(),
  raw_rank = ldccr::unsqids(id)
)
#> # A tibble: 100 × 8
#>    name               mpg   cyl  disp    hp  drat id         raw_rank
#>    <chr>            <dbl> <dbl> <dbl> <dbl> <dbl> <chr>         <int>
#>  1 Dodge Challenger  15.5     8 318     150  2.76 3TWsdpHeRt        1
#>  2 Valiant           18.1     6 225     105  2.76 rvCwolRpZd        2
#>  3 Ferrari Dino      19.7     6 145     175  3.62 gZHVqJXC9M        3
#>  4 Dodge Challenger  15.5     8 318     150  2.76 SlIJUsJqL6        4
#>  5 Fiat 128          32.4     4  78.7    66  4.08 RbetENzTML        5
#>  6 Merc 280C         17.8     6 168.    123  3.92 YFENeTs8qe        6
#>  7 Duster 360        14.3     8 360     245  3.21 4IOohl9IqU        7
#>  8 Duster 360        14.3     8 360     245  3.21 0yTdDtiaTO        8
#>  9 Mazda RX4         21       6 160     110  3.9  oElBIrRxRb        9
#> 10 Merc 240D         24.4     4 147.     62  3.69 MJneCyQRfy       10
#> # ℹ 90 more rows

# 同じnameについて同じIDを振る場合
dplyr::mutate(cars,
  name = ldccr::sqids(name, .ties = "min"),
  raw_rank = ldccr::unsqids(name)
)
#> # A tibble: 100 × 7
#>    name          mpg   cyl  disp    hp  drat raw_rank
#>    <chr>       <dbl> <dbl> <dbl> <dbl> <dbl>    <int>
#>  1 P9N3UMSE7b   15.5     8 318     150  2.76       11
#>  2 gXzHaZJmCXj  18.1     6 225     105  2.76       94
#>  3 VrXp7meUcG   19.7     6 145     175  3.62       18
#>  4 P9N3UMSE7b   15.5     8 318     150  2.76       11
#>  5 wMf3bJxn8K   32.4     4  78.7    66  4.08       22
#>  6 fGp9r2WRdi5  17.8     6 168.    123  3.92       65
#>  7 XtbxpbVzN2   14.3     8 360     245  3.21       15
#>  8 XtbxpbVzN2   14.3     8 360     245  3.21       15
#>  9 QwAUC4Mf97   21       6 160     110  3.9        49
#> 10 vSmCgz01pw   24.4     4 147.     62  3.69       58
#> # ℹ 90 more rows

# 元の順序をごまかしたい場合はこうすればいいが、そういう場合そもそもsqidsは使わないほうがよい
dplyr::mutate(cars,
  name = factor(name) |>
    forcats::fct_anon() |>
    ldccr::sqids(.ties = "min"),
  raw_rank = ldccr::unsqids(name)
)
#> # A tibble: 100 × 7
#>    name          mpg   cyl  disp    hp  drat raw_rank
#>    <chr>       <dbl> <dbl> <dbl> <dbl> <dbl>    <int>
#>  1 tC2ynNJu1bT  15.5     8 318     150  2.76       77
#>  2 H5VNwtTWjH   18.1     6 225     105  2.76       51
#>  3 88RRz0I6xj   19.7     6 145     175  3.62       60
#>  4 tC2ynNJu1bT  15.5     8 318     150  2.76       77
#>  5 IWva1BRu6S   32.4     4  78.7    66  4.08       45
#>  6 W1D0c9r5s6o  17.8     6 168.    123  3.92       81
#>  7 7YdSPDer7w   14.3     8 360     245  3.21       23
#>  8 7YdSPDer7w   14.3     8 360     245  3.21       23
#>  9 iNeDExfpi9j  21       6 160     110  3.9        88
#> 10 YBGEfeThj8S  24.4     4 147.     62  3.69       68
#> # ℹ 90 more rows

# グループ間で同じ連番を振るような場合には、.saltを固定する
dplyr::mutate(cars,
  id = ldccr::sqids(.salt = c(10, 20, 30)),
  raw_rank = ldccr::unsqids(id),
  .by = name
)
#> # A tibble: 100 × 8
#>    name               mpg   cyl  disp    hp  drat id       raw_rank
#>    <chr>            <dbl> <dbl> <dbl> <dbl> <dbl> <chr>       <int>
#>  1 Dodge Challenger  15.5     8 318     150  2.76 Q8A34Wfe        1
#>  2 Valiant           18.1     6 225     105  2.76 Q8A34Wfe        1
#>  3 Ferrari Dino      19.7     6 145     175  3.62 Q8A34Wfe        1
#>  4 Dodge Challenger  15.5     8 318     150  2.76 MWnryuRC        2
#>  5 Fiat 128          32.4     4  78.7    66  4.08 Q8A34Wfe        1
#>  6 Merc 280C         17.8     6 168.    123  3.92 Q8A34Wfe        1
#>  7 Duster 360        14.3     8 360     245  3.21 Q8A34Wfe        1
#>  8 Duster 360        14.3     8 360     245  3.21 MWnryuRC        2
#>  9 Mazda RX4         21       6 160     110  3.9  Q8A34Wfe        1
#> 10 Merc 240D         24.4     4 147.     62  3.69 Q8A34Wfe        1
#> # ℹ 90 more rows
```

注意点として、まず、この関数は別に速くはないです。というか、上で紹介していた処理がいずれも思いのほか速かったため、それに比べると相対的に遅いという感じです。また、繰り返しになりますが、`ldccr::sqids()`で生成される文字列は`ldccr::unsqids()`するだけで元のランクを復元できてしまうため、元の順序を復元されてしまうと困るようなIDを仮名化する用途で使ってはいけません。ただ、ほどほどの長さのランダムっぽい文字列で、しかも重複しない仮IDを雑に生成できるという点では、まあまあ便利かなと思います。


## むすび

- 既存の識別子の仮名化をしたい場合、長さを気にしないのなら、opensslのハッシュ関数を適当なソルト（`key`）を指定して使っておくとよいです
- ランダムっぽい文字列のIDを振りたい場合、仮IDを用意するやり方はいろいろ。一見連番のように見えてもかまわないなら、ぶっちゃけ`dplyr::row_number() |> factor() |> forcats::fct_anon()`とかでもよさげ
- 連番っぽい見た目をどうにか隠したいならsqidsも可。`ldccr::sqids()`と`ldccr::unsqids()`については、思ったより便利そうなら、どこかもっと扱いやすいところに移動するかもしれません


[^1]: 参考：[仮名加工情報・匿名加工情報　信頼ある個人情報の利活用に向けて―制度編―](https://www.ppc.go.jp/personalinfo/legal/#officer_report)
