---
title: "Rでの日本語のかな・カナ・ローマ字・漢字のあいだでの変換"
emoji: "↔️"
type: "tech"
topics: ["r"]
published: true
---

## この記事について

Rでの日本語のかな・カナ・ローマ字・漢字のあいだでの変換の方法を紹介する。なお、ここで紹介している変換はいずれもvectorizeされているので、例として与えている`"カッターを買ったうれしかった"`という入力を`c("カッターを買ったうれしかった", "竹やぶ焼けた")`のような長さが2以上のベクトルに置き換えても問題ない。

## かなとカナのあいだでの変換

簡単な変換は`stringi::stri_trans_general()`で実現できる。`stringi::stri_trans_list()`とすると`stringi::stri_trans_general()`で指定できる変換の一覧が見れるが、750件くらいあるわりにほとんどの変換は日本語とは無関係なため、とくにがんばって確認する必要はない。

```r
stringi::stri_trans_general("カッターを買ったうれしかった", "Kana-Hira")
#> [1] "かったあを買ったうれしかった"
stringi::stri_trans_general("カッターを買ったうれしかった", "Hira-Kana")
#> [1] "カッターヲ買ッタウレシカッタ"
```

このあたりは`zipangu::str_conv_*`という名前のラッパーを利用できるので、変換先を覚えていなくてもなんとなく使える。

## かな・カナからローマ字への変換

かな・カナ文字は次のようにしてローマ字に変換できる。

```r
stringi::stri_trans_general("カッターを買ったうれしかった", "Hira-Latn")
#> [1] "カッタ̄wo買ttaureshikatta"
stringi::stri_trans_general("カッターを買ったうれしかった", "Kana-Latn")
#> [1] "kattāを買ったうれしかった"
stringi::stri_trans_general("カッターを買ったうれしかった", "Any-Latn")
#> [1] "kattāwo mǎittaureshikatta"
```

`Any-Latn`では漢字も変換されるが、おそらく中国語読みになる。かな・カナ文字の部分のみを変換したい場合、次のようにするとヘボン式にできる。「ヘボン式は表音的であり、標準的日本語では発音上区別されない「じ・ぢ」「ず・づ」「お・を」などを書き分けない」（[Wikipedia](https://ja.wikipedia.org/wiki/%E3%83%98%E3%83%9C%E3%83%B3%E5%BC%8F%E3%83%AD%E3%83%BC%E3%83%9E%E5%AD%97)より）点に注意。

```r
stringi::stri_trans_general("カッターを買ったうれしかった", "ja_Hrkt-ja_Latn/BGN")
#> [1] "kattāo買ttaureshikatta"
```

「を」は「wo」になってほしいといった場合には、`audubon::strj_romanize(config = "traditional hepburn")`を使うとよい。この関数では`config`引数から他のいくつかの表記法も選べるのだが、いずれの変換においても漢字などは変換時に無視され、結果には表示されない。

```r
audubon::strj_romanize("カッターを買ったうれしかった", config = "traditional hepburn")
#> [1] "kattāwottaureshikatta"
```

## 漢字混じりからカナへの変換

次のようにしてかなやカナに変換できるが、これだと漢字は変換されない。

```r
stringi::stri_trans_general("カッターを買ったうれしかった", "Any-Hira")
#> [1] "かったあを買ったうれしかった"
stringi::stri_trans_general("カッターを買ったうれしかった", "Any-Kana")
#> [1] "カッターヲ買ッタウレシカッタ"
```

とりあえず全部カナにしたいという場合には、固有名詞を含まないような一般的な文であれば、MeCabで読みを取ると手軽に実現できる。

```r
kana <-
  gibasa::tokenize("カッターを買ったうれしかった") |>
  gibasa::prettify(col_select = "Yomi1") |>
  gibasa::pack(Yomi1, .collapse = "") |>
  dplyr::pull(text)

kana
#> [1] "カッターヲカッタウレシカッタ"
```

この結果は漢字を含まないので、次のようにローマ字に変換できる。

```r
stringi::stri_trans_general(kana, "Any-Latn")
#> [1] "kattāwokattaureshikatta"
```

## ローマ字からかな・カナへの変換

こんなことをしたいシーンはない気がするが、いちおうできる。

```r
stringi::stri_trans_general("kattāwokattaureshikatta", "Latn-Hira")
#> [1] "かったあをかったうれしかった"
stringi::stri_trans_general("kattāwokattaureshikatta", "Latn-Kana")
#> [1] "カッターヲカッタウレシカッタ"
```

また、たとえば「katta-wokattauresikatta」のような表記からかなにしたい場合には、次のようにして実現できる。ただし、ここでビルドしている辞書はいわゆる「ローマ字入力」の仕方を網羅しているわけではないので、一部は正しく変換できない。

```r
temp_dir <- tempdir()

gibasa::build_sys_dic(
  dic_dir = system.file("latin", package = "gibasa"),
  out_dir = temp_dir,
  encoding = "utf8"
)

file.copy(
  system.file("latin/dicrc", package = "gibasa"),
  temp_dir
)
#> [1] TRUE

gibasa::tokenize("katta-wokattauresikatta", sys_dic = temp_dir) |>
  gibasa::pack(feature, .collapse = "") |>
  dplyr::pull(text)
#> [1] "かったーをかったうれしかった"
```

## かなから漢字交じりへの変換

できらぁ！

```r
if (!requireNamespace("kelpbeds", quietly = TRUE)) {
  install.packages("kelpbeds", repos = c("https://paithiov909.r-universe.dev", "https://cran.r-project.org"))
}

temp_dir <- kelpbeds::prep_skkserv(dic_dir = tempdir())

gibasa::build_sys_dic(
  dic_dir = temp_dir,
  out_dir = temp_dir,
  encoding = "utf8"
)

gibasa::tokenize("かったーをかったうれしかった", sys_dic = temp_dir) |>
  gibasa::pack(feature, .collapse = "") |>
  dplyr::pull(text)
#> [1] "カッターを買ったうれしかった"
```

これは[mecab-skkserv](http://chasen.org/~taku/software/mecab-skkserv/)という古いプログラムに付いていた辞書を使って変換したもの。さすがに古すぎるのと、gibasaはN-best解を取れないので、変換候補を複数出力するということができないため、それほど便利ではないかもしれない。
