---
title: "base::use()でRパッケージ中の一部の関数だけ読み込む"
emoji: "📖"
type: "idea"
topics: ["r"]
published: false
---

## R-4.5.0から（？）名前付きインポートができるらしい

例年、4月下旬くらいになると、Rのアップデートが来ます。とくに、春のアップデートでは、マイナーバージョンやメジャーバージョンが繰り上がることから、例年と同じようなペースで行けば、この2025年4月にはR-4.5.0がリリースされるはずです。

R-4.5.0になる予定のR-develのリリースノート（になるはずのドキュメント）を眺めていたところ、「C23がRパッケージインストール時のデフォルトのコンパイラになる」「C++コードのコンパイル時に`-DR_NO_REMAP`が自動的に有効になる」といった情報のほかに、次のような項目が目にとまりました。

> New function `use()` to use packages in R scripts with full control over what gets added to the search path. (Actually already available since `R 4.4.0`.)

`use()`、知らなかったですよね？

たぶん、ふつうの人は気づいていなかっただろうと思います。というか、なんと、すでにリリースされているR-4.4.xでも使えるということらしいので、試しに使ってみました。

## `base::use()`による名前付きインポート

`base::use()`のヘルプを確認すると、次のように書かれています（R-4.4.2で確認しています）。

> useR DocumentationUse Packages
>
> Description
>
> Use packages in R scripts by loading their namespace and attaching a
> package environment including (a subset of) their exports to the
> search path.
>
> Usage
>
> use(package, include.only)
>
> Arguments
>
> package
>
> a character string given the name of a package.
>
> include.only
>
> character vector of names of objects to include in the attached
> environment frame. If missing, all exports are included.
>
> Details
>
> This is a simple wrapper around library which always uses
> attach.required = FALSE, so that packages listed in the Depends clause
> of the DESCRIPTION file of the package to be used never get attached
> automatically to the search path.
>
> This therefore allows to write R scripts with full control over what
> gets found on the search path. In addition, such scripts can easily be
> integrated as package code, replacing the calls to use by the
> corresponding ImportFrom directives in ‘NAMESPACE’ files.
>
> Value
>
> (invisibly) a logical indicating whether the package to be used is
> available.
>
> Note
>
> This functionality is still experimental: interfaces may change in
> future versions.

「Use packages in R scripts by loading their namespace and attaching a package environment including (a subset of) their exports to the search path.」というのは、つまり「**パッケージの名前空間を読み込み、パッケージ中の一部のオブジェクトだけを含むパッケージ環境をサーチパスに追加できる**」ということだと思います。

この説明だけ読んでもわかりにくいですが、ようするに、たとえば次のようにすると、

```r
use("dplyr", c("filter", "mutate"))
#>
#> 次のパッケージを付け加えます: 'dplyr'
#> 以下のオブジェクトは 'package:stats' からマスクされています:
#>
#>     filter
```

dplyrパッケージが提供する関数のうち、`filter()`と`mutate()`だけを読み込むことができるということです。このような機能を一般的に何と呼ぶべきなのかよくわからないですが、ここでは、JavaScriptにおける[import](https://developer.mozilla.org/ja/docs/Web/JavaScript/Reference/Statements/import#import_%E5%AE%A3%E8%A8%80%E3%81%AE%E5%BD%A2)の呼び方にならって、これを「名前付きインポート」と呼ぶことにします。

:::details 名前付きインポート？？
「名前付きインポート（名前付き import）」という日本語もなんだかわかりにくいですが、これは、ある名前が付けられているものをそのままその名前に束縛するみたいなことです。Pythonにおける`import numpy as np`ように、名前空間を（また別な名前の）名前空間に束縛するみたいなやつは、上のページでは「名前空間の import」と呼んでいます。
:::

ここでは、dplyrから`filter()`と`mutate()`だけを名前付きインポートしたので、次のように`filter()`と`mutate()`は`::`なしで呼べますが、`dplyr::select()`を意図して`select()`を呼ぼうとするとエラーになります。

```r
dat <- mtcars |>
  filter(mpg > 20) |>
  mutate(mpg = mpg * 2)
dat
#>                 mpg cyl  disp  hp drat    wt  qsec vs am gear carb
#> Mazda RX4      42.0   6 160.0 110 3.90 2.620 16.46  0  1    4    4
#> Mazda RX4 Wag  42.0   6 160.0 110 3.90 2.875 17.02  0  1    4    4
#> Datsun 710     45.6   4 108.0  93 3.85 2.320 18.61  1  1    4    1
#> Hornet 4 Drive 42.8   6 258.0 110 3.08 3.215 19.44  1  0    3    1
#> Merc 240D      48.8   4 146.7  62 3.69 3.190 20.00  1  0    4    2
#> Merc 230       45.6   4 140.8  95 3.92 3.150 22.90  1  0    4    2
#> Fiat 128       64.8   4  78.7  66 4.08 2.200 19.47  1  1    4    1
#> Honda Civic    60.8   4  75.7  52 4.93 1.615 18.52  1  1    4    2
#> Toyota Corolla 67.8   4  71.1  65 4.22 1.835 19.90  1  1    4    1
#> Toyota Corona  43.0   4 120.1  97 3.70 2.465 20.01  1  0    3    1
#> Fiat X1-9      54.6   4  79.0  66 4.08 1.935 18.90  1  1    4    1
#> Porsche 914-2  52.0   4 120.3  91 4.43 2.140 16.70  0  1    5    2
#> Lotus Europa   60.8   4  95.1 113 3.77 1.513 16.90  1  1    5    2
#> Volvo 142E     42.8   4 121.0 109 4.11 2.780 18.60  1  1    4    2

try(select(dat, mpg))
#> Error in select(dat, mpg) :
#>   関数 "select" を見つけることができませんでした
```

## 解説

「Use packages in R scripts by loading their namespace and attaching a package environment including (a subset of) their exports to the search path.」の意味するところについて、もう少し詳しく説明しましょう。

R言語における環境（environment）については、HadleyのAdvanced Rの7章で解説されているので、よければそちらも参照してください。

- [7 Environments \| Advanced R](https://adv-r.hadley.nz/environments.html)

さて、まず、環境というのは、雑にいうと、変数が存在できる箱みたいなものです。Rで`dat <- mtcars`とかすると、`mtcars`の中身は`.GlobalEnv`という名前の環境のなかで`dat`という変数に束縛されます。

私たちが`library(dplyr)`などとすると、一見、dplyrパッケージの提供する関数がまとめて手元の環境に追加されたかのように見えます。ただし、これはdplyrが提供する`filter`といった関数が`.GlobalEnv`において新たに束縛されているわけではありません。実際、私たちは`.GlobalEnv`に`filter`という名前の関数をまた別につくることができますし、そうしてつくられた関数は、dplyrのfilterよりも優先的に呼び出されます。

これはなぜかというと、イメージとしては、環境が入れ子のようになっているからです。`.GlobalEnv`は入れ子のもっとも内側の環境であり、その外側には`library()`によって読み込まれたパッケージ環境（package environment）が取り巻いています。内側の環境に存在しない変数は、その一つ外側の親環境で探され、一つ外側の親にも存在しない変数は、そのまた一つ外側の親にあたる環境で探されます。

つまり、`library()`というのは、`.GlobalEnv`の一つ外側に、指定したパッケージ環境を用意することに相当します。この点について、Advanced Rは次のように説明しています。

> Each package attached by library() or require() becomes one of the parents of the global environment. The immediate parent of the global environment is the last package you attached, the parent of that package is the second to last package you attached, …

> If you follow all the parents back, you see the order in which every package has been attached. This is known as the search path because all objects in these environments can be found from the top-level interactive workspace.

サーチパス（search path）上にあるパッケージ環境の一覧は、`base::search()`を使って確認できます。先ほどまでのコードを実行した時点で実際に`search()`を呼んでみると、`.GlobalEnv`の直接の親環境は次のように`package:dplyr`となっています。

```r
search()
#>  [1] ".GlobalEnv"        "package:dplyr"     "package:stats"
#>  [4] "package:graphics"  "package:grDevices" "package:utils"
#>  [7] "package:datasets"  "package:methods"   "Autoloads"
#> [10] "package:base"
```

ところで、個々のパッケージ環境の中身は、私たちがアクセスできるサーチパスとはまた別の環境から派生しています。それらはパッケージの名前空間（namespace）といいます。名前空間は、その名前空間に含まれるオブジェクトにアクセスしただけでまるごと読み込まれますが、`library()`しないかぎり、パッケージ空間がサーチパスに追加されるわけではありません。Rの話をするときには、前者の「名前空間が読み込まれる」ことをロード（load）されると表現し、後者の「パッケージ空間がサーチパス上に付け加えられる」ことを指して追加（attach）されるという言い方をします。

実際に確かめてみましょう。まず、`package:dplyr`は先ほどサーチパス上にあったことからも、dplyrのパッケージ空間はすでにサーチパスに追加されています。このセッションでロードされているdplyrのパッケージ環境には`rlang::pkg_env("dplyr")`でアクセスできますが、この環境には`filter`, `mutate`はあっても、`select`はないことが確認できます。

```r
rlang::is_attached("package:dplyr")
#> [1] TRUE
rlang::env_has(rlang::pkg_env("dplyr"), c("filter", "mutate", "select"))
#> filter mutate select
#>   TRUE   TRUE  FALSE
```

## わかりにくいところ

ここで、あらためて`library(dplyr)`してみるとどうなるでしょうか？

```r
library(dplyr)
rlang::env_has(rlang::pkg_env("dplyr"), c("filter", "mutate", "select"))
#> filter mutate select
#>   TRUE   TRUE  FALSE
```

変化しないですね。どうやら、現状の仕様？としてはこういう感じで、一度選択的に名前付きインポートした場合、あらためて`library()`や`use()`を使っても、同名のパッケージ空間の中身を更新することはできないようです。たとえば`dplyr::select`だけあらためてサーチパスに追加したいといった場合、一度`package:dplyr`を`detach()`する必要があります。

```r
detach("package:dplyr")
use("dplyr", "select")
rlang::env_has(rlang::pkg_env("dplyr"), c("filter", "mutate", "select"))
#> filter mutate select
#>  FALSE  FALSE   TRUE
```

あと、これはふつうに`library()`のラッパーらしいので、関数の中で使った場合でも外側のサーチパスに影響します。

```r
rlang::is_attached("package:tidyr")
#> [1] FALSE
use_tidyr <- \() {
  use("tidyr", c("pivot_longer", "pivot_wider"))
}
use_tidyr()
rlang::is_attached("package:tidyr")
#> [1] TRUE
```

ヘルプにはRパッケージを書くときに`@importFrom`の代わりに使えるかもみたいに書かれていますが、別にふつうの関数のなかだけにスコープを切ってアタッチできるみたいなものではないです。したがって、[box::use()](https://klmr.me/box/reference/use.html)とか[import::here()](https://import.rticulate.org/reference/importfunctions.html)とかが滅ぶわけではなさそう。そういうことがしたい場合、次の記事などをあわせて読んでみてください。

- [【R】スコープを限定してパッケージをインポートする \| terashim.com](https://terashim.com/posts/r-scoped-pkg-import/)

## まとめ

- R-4.5.0から`base::use()`でRパッケージ中の一部の関数だけ読み込むことができるようになるらしい
- 覚えておくと便利なシーンも、あるのか？ あるかもしれない……？
- 覚えておこう
