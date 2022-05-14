---
title: 'Rで日本語単語ベクトルchiVeを使う'
emoji: '🥬'
type: 'tech'
topics: ['r','自然言語処理','chive','magnitude']
published: true
author: 'paithiov909'
---

## モチベーション

Pythonで単語ベクトルを扱う方法はたくさんあります。一方で、RはPythonほど自然言語処理によく採用されるわけではないため、Rで単語ベクトルを扱う方法は調べてもややわかりにくいものです。

Rで分散表現を学習する手段には[EmilHvitfeldt/wordsalad](https://github.com/EmilHvitfeldt/wordsalad)といったパッケージがありますが、こうしたパッケージは分散表現をデータフレームとしてオンメモリで扱うため、大規模な分散表現を扱うにはメモリをたくさん積んでいる必要があります。この記事では、あまりメモリに余裕がない環境で分散表現を扱うことを念頭に、Rでmagnitude形式のchiVeを扱う例を紹介します。

## chiVeとMagnitudeについて

### chiVe

[chive](https://github.com/WorksApplications/chiVe)はWorksApplicationsが開発している日本語単語ベクトルです。コーパスにはNWJCが用いられています。また、形態素解析器としてSudachiを利用することで、複数粒度の分割単位について同時に分散表現を得ているようです。

実態としては300次元のword2vecモデルで、素のword2vec形式のほか、gensimのKeyedVectors形式と、[plasticityai/magnitude](https://github.com/plasticityai/magnitude)のmagnitude形式のデータが頒布されています。

### Magnitude

Magnitudeは、[Plasticity](https://github.com/plasticityai)というOrganizationが開発した単語ベクトルを扱うためのライブラリです。gensimなどで学習した分散表現をSQLiteファイルに変換し、便利に扱うことができます。

ライブラリとしてのMagnitudeの特徴は、まず、そのやや独特な未知語処理（out-of-vacabulary lookups）にあります。未知語については、分散表現の変換時にあらかじめ格納しておいた3～6文字の文字ngramをもとに、その未知語と字面が似ている単語ベクトルを引っ張ってきて、ランダムなベクトルと組み合わせることによってそれっぽいベクトルを割り当てるというものです。

また、KeyedVectors形式よりも高速・省メモリに動作することを目指しており、一度呼び出したベクトルのキャッシュ機能や、リモートにあるmagnitudeファイルへのストリーミングアクセス機能（streaming of large models over HTTP）といった工夫がなされています。

日本語情報としては、以下のブログ記事などが参考になります。

- [Embeddingを高速に取り出すMagnitude - Technical Hedgehog](https://kamujun.hatenablog.com/entry/2019/02/28/160616)
- [単語埋め込みにおけるout-of-vocabularyの対応 - magnitudeの初期化 - Out-of-the-box](https://yag-ays.github.io/project/out-of-vocab-magnitude/)

注意点として、Magnitudeは現在すでにアクティブに開発されていないように見えます。とくにgensimなどは比較的頻繁にAPIが変更されるため、依存しているライブラリのバージョンを合わせないと、Magnitudeでの変換は上手く動かないことがあります。このあたりを考慮して依存ライブラリを減らした[davebulaval/magnitude-light](https://github.com/davebulaval/magnitude-light)というforkもあるようです。

## Rでmagnitudeを扱う

### apportitaパッケージ

magnitudeファイルは実態としてはただのSQLiteファイルなので、Pythonでなくても扱えます。キャッシュ機能などを実装せず、標準化された分散表現がほしいというだけなら、`magnitude`というテーブル中に格納されているベクトルをもとにすこし計算するだけで、簡単に取り出すことができます。

このあたりの処理を手軽にやるために[paithiov909/apportita](https://github.com/paithiov909/apportita)というRパッケージを書いたので、これを使うことで、Rでもmagnitudeファイルから単語ベクトルを引っ張ってくることができます。

apportitaは、次のようにあらかじめダウンロードしておいたmagnitudeファイルへのコネクションを張って使います。


```r
library(apportita)

conn <- apportita::magnitude("./magnitude/chive-1.1-mc5-aunit.magnitude")

dim(conn)
#> [1] 322094    300
```

apportitaは、本家のMagnitudeとは異なるものの、簡単な未知語処理を実装しているので、未知語であっても単語ベクトルを返します。たとえば、次の例では、chiVeではSudachiによる文字列正規化がおこなわれているため「すだち」は未知語なのですが、「すだち」に対してもそれっぽい単語ベクトルが返されます。


```r
apportita::has_exact(conn, c("すだち", "酢橘", "徳島"))
#> # A tibble: 3 × 2
#>   keys   exists
#>   <chr>  <lgl> 
#> 1 すだち FALSE 
#> 2 酢橘   TRUE  
#> 3 徳島   TRUE

apportita::query(conn, c("すだち", "酢橘", "徳島"))
#> # A tibble: 3 × 301
#>   key      dim_0   dim_1  dim_2   dim_3    dim_4    dim_5    dim_6    dim_7    dim_8
#>   <chr>    <dbl>   <dbl>  <dbl>   <dbl>    <dbl>    <dbl>    <dbl>    <dbl>    <dbl>
#> 1 徳島   -0.0876 0.0158  0.0813 -0.0527 -0.132   -0.00482  0.0315   0.0108  -0.170  
#> 2 酢橘    0.0310 0.0141  0.0515 -0.0443 -0.0844  -0.0314  -0.0696  -0.0414  -0.00600
#> 3 すだち -0.0319 0.00989 0.0318 -0.0161 -0.00614 -0.0231   0.00332 -0.00974 -0.0162 
#> # … with 291 more variables: dim_9 <dbl>, dim_10 <dbl>, dim_11 <dbl>, dim_12 <dbl>,
#> #   dim_13 <dbl>, dim_14 <dbl>, dim_15 <dbl>, dim_16 <dbl>, dim_17 <dbl>, dim_18 <dbl>,
#> #   dim_19 <dbl>, dim_20 <dbl>, dim_21 <dbl>, dim_22 <dbl>, dim_23 <dbl>, dim_24 <dbl>,
#> #   dim_25 <dbl>, dim_26 <dbl>, dim_27 <dbl>, dim_28 <dbl>, dim_29 <dbl>, dim_30 <dbl>,
#> #   dim_31 <dbl>, dim_32 <dbl>, dim_33 <dbl>, dim_34 <dbl>, dim_35 <dbl>, dim_36 <dbl>,
#> #   dim_37 <dbl>, dim_38 <dbl>, dim_39 <dbl>, dim_40 <dbl>, dim_41 <dbl>, dim_42 <dbl>,
#> #   dim_43 <dbl>, dim_44 <dbl>, dim_45 <dbl>, dim_46 <dbl>, dim_47 <dbl>, …
```

ただし、apportitaはMagnitudeの完全な移植ではないため、提供しているAPIが異なります。たとえば、`apportita::most_simmilar`はMagnitudeの`most_similar`とは異なり、次のように使います。


```r
apportita::most_similar(conn, "徳島市", c("すだち", "酢橘", "徳島"), n = 3)
#> # A tibble: 3 × 2
#>   keys   similarity
#>   <chr>       <dbl>
#> 1 徳島        0.774
#> 2 酢橘        0.342
#> 3 すだち      0.279
```

### RcppHNSWによる近似最近傍探索

本家のMagnitudeの`most_similar`のように、特定のベクトルと似ているベクトルを探索して返したい場合には、最近傍探索をする必要があります。

やり方はいろいろ考えられますが、ここでは、はじめに探索対象とする分散表現をmagnitudeファイルから読み込みます。


```r
vectors <- apportita::slice_n(conn, n = 50000)
head(vectors)
#> # A tibble: 6 × 301
#>   key    dim_0   dim_1   dim_2    dim_3  dim_4    dim_5   dim_6   dim_7  dim_8   dim_9
#>   <chr>  <dbl>   <dbl>   <dbl>    <dbl>  <dbl>    <dbl>   <dbl>   <dbl>  <dbl>   <dbl>
#> 1 の    0.0533 -0.0517 -0.0466 -0.0661  0.0366 -0.0323  -0.0470 -0.0193 0.0439 -0.0450
#> 2 、    0.0746 -0.0678 -0.0885 -0.0643  0.0419  0.0240  -0.0469 -0.0337 0.0312 -0.0203
#> 3 て    0.112  -0.0476 -0.0432 -0.0225  0.0607  0.00685 -0.0114 -0.0659 0.0236 -0.0103
#> 4 に    0.0885 -0.0638 -0.0464 -0.0564  0.0954  0.0491  -0.0520 -0.0576 0.0328 -0.0806
#> 5 だ    0.0773  0.0117 -0.0612  0.00233 0.0147 -0.0114  -0.0357  0.0110 0.0719  0.0177
#> 6 た    0.0642 -0.0270 -0.0399 -0.0287  0.0711  0.0488  -0.0300 -0.0257 0.0308 -0.0302
#> # … with 290 more variables: dim_10 <dbl>, dim_11 <dbl>, dim_12 <dbl>, dim_13 <dbl>,
#> #   dim_14 <dbl>, dim_15 <dbl>, dim_16 <dbl>, dim_17 <dbl>, dim_18 <dbl>, dim_19 <dbl>,
#> #   dim_20 <dbl>, dim_21 <dbl>, dim_22 <dbl>, dim_23 <dbl>, dim_24 <dbl>, dim_25 <dbl>,
#> #   dim_26 <dbl>, dim_27 <dbl>, dim_28 <dbl>, dim_29 <dbl>, dim_30 <dbl>, dim_31 <dbl>,
#> #   dim_32 <dbl>, dim_33 <dbl>, dim_34 <dbl>, dim_35 <dbl>, dim_36 <dbl>, dim_37 <dbl>,
#> #   dim_38 <dbl>, dim_39 <dbl>, dim_40 <dbl>, dim_41 <dbl>, dim_42 <dbl>, dim_43 <dbl>,
#> #   dim_44 <dbl>, dim_45 <dbl>, dim_46 <dbl>, dim_47 <dbl>, dim_48 <dbl>, …
```

次に、アナロジーを求めたいベクトルを用意します。ここでは、「阿波 + 高知 - 徳島」のベクトルをつくります。


```r
tosa <-
  apportita::query(conn, "阿波")[, -1] +
  apportita::query(conn, "高知")[, -1] -
  apportita::query(conn, "徳島")[, -1]
```

そのうえで、近似最近傍探索をします。ここでは[RcppHNSW](https://github.com/jlmelville/rcpphnsw)を使ってみます。


```r
library(RcppHNSW)

ann <- new(HnswCosine, ncol(vectors) - 1, nrow(vectors), 16, 200)
ann$addItems(as.matrix(vectors[, -1]))

(nn <- ann$getAllNNsList(as.matrix(tosa), k = 10, include_distnces = TRUE))
#> $item
#>       [,1] [,2] [,3]  [,4]  [,5]  [,6]  [,7]  [,8]  [,9] [,10]
#> [1,] 17449 6494 9405 48595 26865 26419 17760 23774 44484 28294
#> 
#> $distance
#>          [,1]      [,2]      [,3]     [,4]      [,5]      [,6]     [,7]      [,8]
#> [1,] 0.212158 0.3214518 0.3684468 0.401971 0.4369003 0.4479579 0.455144 0.4572788
#>           [,9]     [,10]
#> [1,] 0.4663377 0.4811183

vectors[nn$item[1, ], ]
#> # A tibble: 10 × 301
#>    key          dim_0   dim_1  dim_2    dim_3   dim_4     dim_5    dim_6    dim_7   dim_8
#>    <chr>        <dbl>   <dbl>  <dbl>    <dbl>   <dbl>     <dbl>    <dbl>    <dbl>   <dbl>
#>  1 阿波      -0.115    0.0373 0.0513 -0.0358  -0.138  -0.0788    0.0180  -0.0241  -0.156 
#>  2 高知      -0.0509   0.0266 0.127  -0.0422  -0.184   0.0447    0.00445  0.0162  -0.133 
#>  3 土佐      -0.0407   0.0373 0.102  -0.0967  -0.145   0.0569    0.0123   0.00952 -0.113 
#>  4 よさこい…  0.00640  0.0955 0.0680 -0.0414  -0.0592  0.000781  0.0307  -0.0245  -0.0438
#>  5 安芸      -0.115   -0.127  0.0549 -0.00738 -0.130   0.0130   -0.0389  -0.0104  -0.183 
#>  6 伊予      -0.113    0.0471 0.0643 -0.0276  -0.119   0.0325    0.0443   0.0229  -0.144 
#>  7 よさこい   0.0146   0.0957 0.0775  0.0213  -0.101  -0.0689    0.0757  -0.00640 -0.0457
#>  8 四万十    -0.100   -0.0126 0.0279 -0.0410  -0.201   0.0913   -0.0704  -0.0404  -0.0927
#>  9 桂浜      -0.0516   0.0622 0.0831 -0.0363  -0.189   0.0127   -0.0335   0.0121  -0.0880
#> 10 宇和島    -0.0588   0.0554 0.0551  0.0161  -0.150  -0.00970   0.0102   0.0553  -0.145 
#> # … with 291 more variables: dim_9 <dbl>, dim_10 <dbl>, dim_11 <dbl>, dim_12 <dbl>,
#> #   dim_13 <dbl>, dim_14 <dbl>, dim_15 <dbl>, dim_16 <dbl>, dim_17 <dbl>, dim_18 <dbl>,
#> #   dim_19 <dbl>, dim_20 <dbl>, dim_21 <dbl>, dim_22 <dbl>, dim_23 <dbl>, dim_24 <dbl>,
#> #   dim_25 <dbl>, dim_26 <dbl>, dim_27 <dbl>, dim_28 <dbl>, dim_29 <dbl>, dim_30 <dbl>,
#> #   dim_31 <dbl>, dim_32 <dbl>, dim_33 <dbl>, dim_34 <dbl>, dim_35 <dbl>, dim_36 <dbl>,
#> #   dim_37 <dbl>, dim_38 <dbl>, dim_39 <dbl>, dim_40 <dbl>, dim_41 <dbl>, dim_42 <dbl>,
#> #   dim_43 <dbl>, dim_44 <dbl>, dim_45 <dbl>, dim_46 <dbl>, dim_47 <dbl>, …
```

### WRD（Word Rotator's Distance）の計算

apportitaの実験的な機能として、単語列間のWRD（Word Rotator's Distance）を計算することができます。

WRDは、Word Mover's Distanceをもとに改良された自然言語文間の非類似度を測るための指標です。日本語の説明は以下などを読んでください。

- [横井ほか「超球面上での最適輸送に基づく文類似性尺度」(PDF)](https://www.anlp.jp/proceedings/annual_meeting/2020/pdf_dir/D1-2.pdf)
- [WRD（Word Rotator's Distance）で文書間の距離（類似度）を計算する - Qiita](https://qiita.com/kenta1984/items/bad7e2f68331849d0053)
- [Word Rotator’s Distance - 由々しき](https://mytache.hatenablog.com/entry/2020/06/13/032100)

既存のPython実装はEarth Mover's Distanceの計算に[POT](https://pythonot.github.io/)の[ot.emb2](https://pythonot.github.io/all.html?highlight=emd#ot.emd2)というメソッドを使っているようですが、apportitaでは[transport](https://CRAN.R-project.org/package=transport)というRパッケージにある`transport::wasserstein`を使っています。


```r
Sys.setenv(SUDACHI_DICT_PATH = "./dict/system_full.dic")

apportita::calc_wrd(
  conn,
  washoku::tokenize("醤油とコーラって似ていますよね", mode = "A")$normalized,
  washoku::tokenize("ソースとサイダーも似ていますよ", mode = "A")$normalized
)
#> [1] 0.1462258

apportita::calc_wrd(
  conn,
  washoku::tokenize("老人と海", mode = "A")$normalized,
  washoku::tokenize("風に舞いあがるビニールシート", mode = "A")$normalized
)
#> [1] 0.6577369
```

WRDを計算するだけなら`apportita::calc_wrd`に適当な単語列をベクトルで与えるだけでよいのですが、ここで使っているchiVeはSudachiによるA単位の分かち書きをベースにしているため、いちおうSudachiによって正規化された形の単語列を使っています。

RでSudachiPyを利用するには[sudachir]( https://CRAN.R-project.org/package=sudachir)というパッケージもありますが、ここでは[paithiov909/washoku](https://github.com/paithiov909/washoku)というパッケージを使っています。このリポジトリは[uribo/washoku](https://github.com/uribo/washoku)からforkしたもので、[yutannihilation/fledgingr](https://github.com/yutannihilation/fledgingr)から引っ張ってきた[sudachi.rs](https://github.com/WorksApplications/sudachi.rs)のラッパー関数を実装しています。[Sudachiの辞書](https://github.com/WorksApplications/SudachiDict)はあらかじめ手元にダウンロードしておく必要があります。

## さいごに

magnitudeファイルへのコネクションは、使い終わったら忘れずに閉じましょう。


```r
close(conn)
```

GitHubリポジトリにスターをもらえるととてもうれしいです。

- [paithiov909/apportita: Utility for Handling ‘magnitude’ Word Embeddings](https://github.com/paithiov909/apportita)
