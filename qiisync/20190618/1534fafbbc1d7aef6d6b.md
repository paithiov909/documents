---
ID: 1534fafbbc1d7aef6d6b
Title: RからCaboChaを呼ぶだけのパッケージ書いた
Tags: R,自然言語処理,Cabocha,GoogleColaboratory
Author: Kato Akiru
Private: false
---


[自然言語処理 #2 Advent Calendar2019](https://qiita.com/advent-calendar/2019/nlp2) 23日目です。

[![paithiov909/pipian - GitHub](https://gh-card.dev/repos/paithiov909/pipian.svg)](https://github.com/paithiov909/pipian)

## これは何？

RからCaboChaを呼ぶためのライブラリ。`base::system()`から`cabocha -f3`コマンドを呼んで出力した一時ファイル（XML）を読みに行っている。Rcpp経由ではないためとくに速くはないが、CaboChaとMeCabのパスが通っていれば使えるはずなので導入は楽。**外部コマンドとして叩くだけなので、Windows環境で64bit Rから32bit CaboChaを実行する場合でも問題なく動作する。**

これで使えるようになる。

``` r
remotes::install_github("paithiov909/pipian")
```

## モチベーション

以前つくったRから形態素解析するオレオレパッケージから機能を切り分けたかった。

## 使い方

### XML出力のパース


```r
## ここではUTF-8のIPA辞書を使っています
text <- stringi::stri_enc_toutf8("ふと振り向くと、たくさんの味方がいてたくさんの優しい人間がいることを、わざわざ自分の誕生日が来ないと気付けない自分を奮い立たせながらも、毎日こんな、湖のようななんの引っ掛かりもない、落ちつき倒し、音一つも感じさせない人間でいれる方に憧れを持てたとある25歳の眩しき朝のことでした")
res <- pipian::CabochaTbl(text, force.utf8 = TRUE)
res$tbl
#> # A tibble: 37 x 4
#>    id    link  score     morphs      
#>    <chr> <chr> <chr>     <chr>       
#>  1 0     1     1.287564  ふと        
#>  2 1     36    -2.336376 振り向くと、
#>  3 2     3     1.927252  たくさんの  
#>  4 3     4     0.834422  味方が      
#>  5 4     8     2.020974  いて        
#>  6 5     7     1.913107  たくさんの  
#>  7 6     7     1.773527  優しい      
#>  8 7     8     2.371958  人間が      
#>  9 8     9     3.138138  いる        
#> 10 9     13    0.293884  ことを、    
#> # ... with 27 more rows
```

これで`res$plot()`するとグラフが描ける。

![Rplot.png](https://qiita-image-store.s3.amazonaws.com/0/228173/60b9dc99-954e-82a0-b428-9dba6ffd0520.png)

### XMLをflat XMLとして読み込む

XMLを`flatxml::fxml_importXMLFlat()`で読み込んだflat XMLを返すことができる。


```r
head(pipian::cabochaFlatXML(text, force.utf8 = TRUE))
#>       elem. elemid. attr. value.    level1   level2 level3 level4
#> 1 sentences       1  <NA>   <NA> sentences     <NA>   <NA>   <NA>
#> 2  sentence       2  <NA>   <NA> sentences sentence   <NA>   <NA>
#> 3     chunk       3  <NA>   <NA> sentences sentence  chunk   <NA>
#> 4     chunk       3    id      0 sentences sentence  chunk   <NA>
#> 5     chunk       3  link      1 sentences sentence  chunk   <NA>
#> 6     chunk       3   rel      D sentences sentence  chunk   <NA>
```

### flat XMLの整形

`pipian::cabochaFlatXML(as.tibble = FALSE)`で出力したflat XMLをtibbleに整形できる（IPA辞書と同じ品詞体系の場合にかぎる）。このtibbleは[CabochaR](https://minowalab.org/cabochar/)が出力する形式を参考にしたもので、次のカラムからなる。

- sentence_idx: 文番号
- chunk_idx: 文節のインデックス
- D1: 文節番号
- D2: 係り先の文節の文節番号
- rel:（よくわからない値）
- score: 係り関係のスコア
- head: 主辞の形態素の番号
- func: 機能語の形態素の番号
- tok_idx: 形態素の番号
- ne_value: 固有表現解析の結果の値（`-n 1`オプションを使用している）
- Surface: 表層形
- POS1~POS4: 品詞, 品詞細分類1, 品詞細分類2, 品詞細分類3
- X5StageUse1: 活用型（五段, 下二段...）
- X5StageUse2: 活用形（連用形, 基本形...）
- Original: 原形
- Yomi1~Yomi2: 読み, 発音


```r
res <- pipian::cabochaFlatXML(text, force.utf8 = TRUE) %>%
  pipian::CabochaR()

res$as_tibble()
#> # A tibble: 78 x 20
#>    sentence_idx chunk_idx D1    D2    rel   score head  func  tok_idx ne_value Surface
#>           <int>     <dbl> <chr> <chr> <chr> <chr> <chr> <chr>   <dbl> <chr>    <chr>  
#>  1            1         3 0     1     D     1.28~ 0     0           0 O        ふと   
#>  2            1         5 1     36    D     -2.3~ 1     2           1 O        振り向く~
#>  3            1         5 1     36    D     -2.3~ 1     2           2 O        と     
#>  4            1         5 1     36    D     -2.3~ 1     2           3 O        、     
#>  5            1         9 2     3     D     1.92~ 4     5           4 O        たくさん~
#>  6            1         9 2     3     D     1.92~ 4     5           5 O        の     
#>  7            1        12 3     4     D     0.83~ 6     7           6 O        味方   
#>  8            1        12 3     4     D     0.83~ 6     7           7 O        が     
#>  9            1        15 4     8     D     2.02~ 8     9           8 O        い     
#> 10            1        15 4     8     D     2.02~ 8     9           9 O        て     
#> # ... with 68 more rows, and 9 more variables: POS1 <chr>, POS2 <chr>, POS3 <chr>,
#> #   POS4 <chr>, X5StageUse1 <chr>, X5StageUse2 <chr>, Original <chr>, Yomi1 <chr>,
#> #   Yomi2 <chr>
```

## Google Colaboratoryで試すやり方

Google Colaboratory上で試すことができる。

なお、Colab（この記事を書いた時点だとUbuntu 18.04.3 LTS）にCaboCha（ver.0.69）を公式のGoogleDriveからwgetなどして入れようとすると、何かの確認ダイアログに阻まれるはずなので、古い情報を載せているサイトからスニペットをコピペしてきてもDLできないことがあるが、この記事はそれに対応済みの例。

### CaboChaのセットアップ

まず、MeCabを入れる。

``` bash
%%bash
apt install mecab libmecab-dev mecab-ipadic-utf8
```

次にCRFを入れる。

``` bash
wget "https://drive.google.com/uc?export=download&id=0B4y35FiV1wh7QVR6VXJ5dWExSTQ" -O CRF++-0.58.tar.gz
tar -zxvf CRF++-0.58.tar.gz CRF++-0.58/
cd CRF++-0.58/
./configure
make
make install
ldconfig
cd ../
```

CaboChaを入れる。

``` py
!curl -sc /tmp/cookie "https://drive.google.com/uc?export=download&id=0B4y35FiV1wh7SDd1Q1dUQkZQaUU" > /dev/null
import re
with open('/tmp/cookie') as f:
    contents = f.read()
    print(contents)
    m = re.findall(r'_warning[\S]+[\s]+([a-zA-Z0-9]+)\n', contents)
    code = m[0]
url = f'https://drive.google.com/uc?export=download&confirm={code}&id=0B4y35FiV1wh7SDd1Q1dUQkZQaUU'
!curl -Lb /tmp/cookie "$url" -o cabocha-0.69.tar.bz2
!tar -jxvf cabocha-0.69.tar.bz2 cabocha-0.69/
%cd cabocha-0.69/
!./configure --with-mecab-config=`which mecab-config` --with-charset=UTF8 --enable-utf8-only
!make
!make check
!make install
!ldconfig
%cd ../
```

### pipianのインストール

rpy2経由でRを使えるようにする。

```
%load_ext rpy2.ipython
```

pipianを入れる。

``` r
%%R
remotes::install_github("paithiov909/pipian")
```

使用例。なお、これで`res$plot()`するとigraphを利用して係り受けを図示できるが、Colabは日本語フォントがない環境なのでうまく表示されない。図示したい場合は、日本語フォントを入れたうえで、`res$tbl2graph()`の戻り値であるigraphオブジェクトを利用するなどして自分で頑張ってください。

``` r
%%R
res <- pipian::CabochaTbl("ふつうに動くよ")
res$tbl
#> # A tibble: 2 x 4
#>   id    link  score    morphs  
#>   <chr> <chr> <chr>    <chr>   
#> 1 0     1     0.000000 ふつうに
#> 2 1     -1    0.000000 動くよ  
```

## 参考にしたはずの記事

### igraph

- [R+igraph – Kazuhiro Takemoto](https://sites.google.com/site/kztakemoto/r-seminar-on-igraph---supplementary-information)
- [R の igraph を使ってネットワークを分析する – Qiita](https://qiita.com/tomov3/items/c72e06eaf300b322e99d)

### tidygraph, ggraph, visNetwork

- [Introduction to Network Analysis with R](https://www.jessesadler.com/post/network-analysis-with-r/)
- [Introducing tidygraph · Data Imaginist](https://www.data-imaginist.com/2017/introducing-tidygraph/)

### グラフの中心性について

- [グラフ・ネットワーク分析で遊ぶ(3)：中心性(PageRank, betweeness, closeness, etc.) – 六本木で働くデータサイエンティストのブログ](http://tjo.hatenablog.com/entry/2015/12/09/190000)

### CaboCha

CaboChaが出力するXMLはそのままではパースできないので、適当なルートノードを追記する必要がある。

- [CaboChaで始める係り受け解析 - Qiita](https://qiita.com/nezuq/items/f481f07fc0576b38e81d)
- [CaboChaによってXMLで出力されたファイルをパースする。 – gepuroの日記](http://d.hatena.ne.jp/gepuro/20111014/1318610472)

## セッション情報


```r
sessioninfo::session_info()
#> - Session info ------------------------------------------------------------------------
#>  setting  value                       
#>  version  R version 4.0.3 (2020-10-10)
#>  os       Windows 10 x64              
#>  system   x86_64, mingw32             
#>  ui       RStudio                     
#>  language (EN)                        
#>  collate  Japanese_Japan.932          
#>  ctype    Japanese_Japan.932          
#>  tz       Asia/Tokyo                  
#>  date     2021-01-20                  
#> 
#> - Packages ----------------------------------------------------------------------------
#>  package     * version date       lib source                        
#>  assertthat    0.2.1   2019-03-21 [1] CRAN (R 4.0.2)                
#>  backports     1.2.1   2020-12-09 [1] CRAN (R 4.0.3)                
#>  callr         3.5.1   2020-10-13 [1] CRAN (R 4.0.3)                
#>  cli           2.2.0   2020-11-20 [1] CRAN (R 4.0.3)                
#>  crayon        1.3.4   2017-09-16 [1] CRAN (R 4.0.2)                
#>  curl          4.3     2019-12-02 [1] CRAN (R 4.0.2)                
#>  DBI           1.1.1   2021-01-15 [1] CRAN (R 4.0.3)                
#>  desc          1.2.0   2018-05-01 [1] CRAN (R 4.0.2)                
#>  devtools      2.3.2   2020-09-18 [1] CRAN (R 4.0.2)                
#>  digest        0.6.27  2020-10-24 [1] CRAN (R 4.0.3)                
#>  dplyr         1.0.3   2021-01-15 [1] CRAN (R 4.0.3)                
#>  ellipsis      0.3.1   2020-05-15 [1] CRAN (R 4.0.2)                
#>  evaluate      0.14    2019-05-28 [1] CRAN (R 4.0.2)                
#>  fansi         0.4.2   2021-01-15 [1] CRAN (R 4.0.3)                
#>  flatxml       0.1.1   2020-12-01 [1] CRAN (R 4.0.3)                
#>  fs            1.5.0   2020-07-31 [1] CRAN (R 4.0.2)                
#>  generics      0.1.0   2020-10-31 [1] CRAN (R 4.0.3)                
#>  glue          1.4.2   2020-08-27 [1] CRAN (R 4.0.2)                
#>  hms           1.0.0   2021-01-13 [1] CRAN (R 4.0.3)                
#>  htmltools     0.5.1   2021-01-12 [1] CRAN (R 4.0.3)                
#>  httr          1.4.2   2020-07-20 [1] CRAN (R 4.0.2)                
#>  igraph        1.2.6   2020-10-06 [1] CRAN (R 4.0.3)                
#>  knitr         1.30    2020-09-22 [1] CRAN (R 4.0.2)                
#>  lifecycle     0.2.0   2020-03-06 [1] CRAN (R 4.0.2)                
#>  magrittr      2.0.1   2020-11-17 [1] CRAN (R 4.0.3)                
#>  MASS          7.3-53  2020-09-09 [2] CRAN (R 4.0.3)                
#>  memoise       1.1.0   2017-04-21 [1] CRAN (R 4.0.2)                
#>  pillar        1.4.7   2020-11-20 [1] CRAN (R 4.0.3)                
#>  pipian      * 0.2.4   2021-01-20 [1] local                         
#>  pkgbuild      1.2.0   2020-12-15 [1] CRAN (R 4.0.3)                
#>  pkgconfig     2.0.3   2019-09-22 [1] CRAN (R 4.0.2)                
#>  pkgdown       1.4.1   2020-09-23 [1] Github (r-lib/pkgdown@cdd8340)
#>  pkgload       1.1.0   2020-05-29 [1] CRAN (R 4.0.2)                
#>  prettyunits   1.1.1   2020-01-24 [1] CRAN (R 4.0.2)                
#>  processx      3.4.5   2020-11-30 [1] CRAN (R 4.0.3)                
#>  ps            1.5.0   2020-12-05 [1] CRAN (R 4.0.3)                
#>  purrr         0.3.4   2020-04-17 [1] CRAN (R 4.0.2)                
#>  R.cache       0.14.0  2019-12-06 [1] CRAN (R 4.0.2)                
#>  R.methodsS3   1.8.1   2020-08-26 [1] CRAN (R 4.0.2)                
#>  R.oo          1.24.0  2020-08-26 [1] CRAN (R 4.0.2)                
#>  R.utils       2.10.1  2020-08-26 [1] CRAN (R 4.0.2)                
#>  R6            2.5.0   2020-10-28 [1] CRAN (R 4.0.3)                
#>  readr         1.4.0   2020-10-05 [1] CRAN (R 4.0.3)                
#>  rematch2      2.1.2   2020-05-01 [1] CRAN (R 4.0.2)                
#>  remotes       2.2.0   2020-07-21 [1] CRAN (R 4.0.2)                
#>  rlang         0.4.10  2020-12-30 [1] CRAN (R 4.0.3)                
#>  rmarkdown     2.6     2020-12-14 [1] CRAN (R 4.0.3)                
#>  rprojroot     2.0.2   2020-11-15 [1] CRAN (R 4.0.3)                
#>  rsconnect     0.8.16  2019-12-13 [1] CRAN (R 4.0.2)                
#>  rstudioapi    0.13    2020-11-12 [1] CRAN (R 4.0.3)                
#>  rvest         0.3.6   2020-07-25 [1] CRAN (R 4.0.2)                
#>  sessioninfo   1.1.1   2018-11-05 [1] CRAN (R 4.0.2)                
#>  stringi     * 1.5.3   2020-09-09 [1] CRAN (R 4.0.2)                
#>  stringr       1.4.0   2019-02-10 [1] CRAN (R 4.0.2)                
#>  styler        1.3.2   2020-02-23 [1] CRAN (R 4.0.2)                
#>  testthat      3.0.1   2020-12-17 [1] CRAN (R 4.0.3)                
#>  tibble        3.0.5   2021-01-15 [1] CRAN (R 4.0.3)                
#>  tidyr         1.1.2   2020-08-27 [1] CRAN (R 4.0.2)                
#>  tidyselect    1.1.0   2020-05-11 [1] CRAN (R 4.0.2)                
#>  usethis       2.0.0   2020-12-10 [1] CRAN (R 4.0.3)                
#>  utf8          1.1.4   2018-05-24 [1] CRAN (R 4.0.2)                
#>  vctrs         0.3.6   2020-12-17 [1] CRAN (R 4.0.3)                
#>  withr         2.4.0   2021-01-16 [1] CRAN (R 4.0.3)                
#>  xfun          0.20    2021-01-06 [1] CRAN (R 4.0.3)                
#>  xml2          1.3.2   2020-04-23 [1] CRAN (R 4.0.2)                
#>  yaml          2.2.1   2020-02-01 [1] CRAN (R 4.0.0)                
#> 
#> [1] C:/Users/user/Documents/R/win-library/4.0
#> [2] C:/Program Files/R/R-4.0.3/library
```
