---
ID: 4afab6def8d2eabd3bb6
Title: 【R】spacyr・cleanNLPのデモ
Tags: R,Anaconda,spaCy
Author: Kato Akiru
Private: false
---

## spacyr

### 環境

Windows10、Miniconda3 (Python 3.7.6 64-bit) です。

### spacyrについて

[spaCy](https://spacy.io/)を{reticulate}経由で呼ぶRパッケージです。

- [Wrapper to the spaCy NLP Library • spacyr](https://spacyr.quanteda.io/)

spaCyは2.3で日本語のモデルが利用できるようになったらしいです。

### Initialize

簡単に導入できるのですが、Windows環境だけの罠として以下の手順はRを管理者権限で実行する必要があります。Rstudioならショートカットを右クリックから「管理者として実行」です。

> For Windows, you need to run R as an administrator to make installation work properly. To do so, right click the RStudio icon (or R desktop icon) and select “Run as administrator” when launching R.

管理者として以下のスクリプトを実行すると、`spacy_condaenv`という名前のcondaenv内にspaCyと日本語の`ja_core_news_md`というモデルがダウンロードされます

```r
spacyr::spacy_install()
spacyr::spacy_download_langmodel("ja_core_news_md")
```

モデルをロードして使います。


```r
spacyr::spacy_initialize(model = "ja_core_news_md")
#> Found 'spacy_condaenv'. spacyr will use this environment
#> successfully initialized (spaCy Version: 2.3.2, language model: ja_core_news_md)
#> (python options: type = "condaenv", value = "spacy_condaenv")
```

### 使用例

日本語のモデルではlemmatizationはうまく動かないよという怒られが発生するので`lemma = FALSE`とします。


```r
spacyr::spacy_parse("望遠鏡で泳ぐ彼女を見た", lemma = FALSE)
#>   doc_id sentence_id token_id token  pos entity
#> 1  text1           1        1  望遠 NOUN       
#> 2  text1           1        2    鏡 NOUN       
#> 3  text1           1        3    で  ADP       
#> 4  text1           1        4  泳ぐ VERB       
#> 5  text1           1        5  彼女 PRON       
#> 6  text1           1        6    を  ADP       
#> 7  text1           1        7    見 VERB       
#> 8  text1           1        8    た  AUX
```

UDなので係り受けを出せます。


```r
spacyr::spacy_parse("望遠鏡で泳ぐ彼女を見た", dependency = TRUE, lemma = FALSE, pos = FALSE)
#>   doc_id sentence_id token_id token head_token_id  dep_rel entity
#> 1  text1           1        1  望遠             2 compound       
#> 2  text1           1        2    鏡             4      obl       
#> 3  text1           1        3    で             2     case       
#> 4  text1           1        4  泳ぐ             5      acl       
#> 5  text1           1        5  彼女             7      obj       
#> 6  text1           1        6    を             5     case       
#> 7  text1           1        7    見             7     ROOT       
#> 8  text1           1        8    た             7      aux
```

とくにサイズの大きいモデルを使う場合、モデルをメモリに読み込んだままだとよろしくない場合があるので、使い終わったら以下のおまじないを実行するとよいらしいです。


```r
spacyr::spacy_finalize()
```

## cleanNLP

### cleanNLPについて

UDPipe、spaCy、CoreNLPをtidyに使えるよ！というRパッケージです。

- [cleanNLP: A Tidy Data Model for Natural Language Processing | cleanNLP](https://statsmaths.github.io/cleanNLP/)

UDPipeについてはふつうに{udpipe}をバックエンドとして使っています。spaCyとCoreNLPについてはpipでPythonライブラリを別途導入して、それをバックエンドとして使うようです（{spacyr}や{coreNLP}とは無関係）。spaCyについてはcondaenvのなかにすでにモデルをダウンロードしてあるのでそれを利用できないか試してみたのですが、手元ではうまく動かせませんでした。実質的に{udpipe}のラッパーという感じです。

### Initialize

`cleanNLP::cnlp_init_udpipe`だけで動くようになります。脳死でも使えて便利。


```r
library(cleanNLP)
cleanNLP::cnlp_init_udpipe(model_name = "japanese")
```

### 使用例

Windows環境なので文字コードをUTF-8に変換して渡す必要があります。


```r
annotation <- cleanNLP::cnlp_annotate(input = iconv("望遠鏡で泳ぐ彼女を見た", to = "UTF-8"))
annotation$token
#> # A tibble: 7 x 11
#>   doc_id   sid tid   token  token_with_ws lemma  upos  xpos  feats tid_source relation
#> *  <int> <int> <chr> <chr>  <chr>         <chr>  <chr> <chr> <chr> <chr>      <chr>   
#> 1      1     1 1     望遠鏡 望遠鏡        望遠鏡 NOUN  NN    <NA>  3          obl     
#> 2      1     1 2     で     で            で     ADP   PS    <NA>  1          case    
#> 3      1     1 3     泳ぐ   泳ぐ          泳ぐ   VERB  VV    <NA>  4          acl     
#> 4      1     1 4     彼女   彼女          彼女   PRON  NP    <NA>  6          obj     
#> 5      1     1 5     を     を            を     ADP   PS    <NA>  4          case    
#> 6      1     1 6     見     見            見る   VERB  VV    <NA>  0          root    
#> 7      1     1 7     た     た            た     AUX   AV    <NA>  6          aux
```

## セッション情報


```r
sessioninfo::session_info()
#> - Session info ---------------------------------------------------------------------------
#>  setting  value                       
#>  version  R version 4.0.2 (2020-06-22)
#>  os       Windows 10 x64              
#>  system   x86_64, mingw32             
#>  ui       RStudio                     
#>  language (EN)                        
#>  collate  Japanese_Japan.932          
#>  ctype    Japanese_Japan.932          
#>  tz       Asia/Tokyo                  
#>  date     2020-09-28                  
#> 
#> - Packages -------------------------------------------------------------------------------
#>  ! package      * version date       lib source                             
#>    assertthat     0.2.1   2019-03-21 [1] CRAN (R 4.0.2)                     
#>    backports      1.1.10  2020-09-15 [1] CRAN (R 4.0.2)                     
#>    blob           1.2.1   2020-01-20 [1] CRAN (R 4.0.2)                     
#>    blogdown       0.20    2020-06-23 [1] CRAN (R 4.0.2)                     
#>    bookdown       0.20    2020-06-23 [1] CRAN (R 4.0.2)                     
#>    broom          0.7.0   2020-07-09 [1] CRAN (R 4.0.2)                     
#>    callr          3.4.4   2020-09-07 [1] CRAN (R 4.0.2)                     
#>    cellranger     1.1.0   2016-07-27 [1] CRAN (R 4.0.2)                     
#>    cleanNLP     * 3.0.2   2020-03-08 [1] CRAN (R 4.0.2)                     
#>    cli            2.0.2   2020-02-28 [1] CRAN (R 4.0.2)                     
#>    colorspace     1.4-1   2019-03-18 [1] CRAN (R 4.0.2)                     
#>    conifer        0.0.4   2020-09-26 [1] local                              
#>    crayon         1.3.4   2017-09-16 [1] CRAN (R 4.0.2)                     
#>    data.table     1.13.0  2020-07-24 [1] CRAN (R 4.0.2)                     
#>    DBI            1.1.0   2019-12-15 [1] CRAN (R 4.0.2)                     
#>    dbplyr         1.4.4   2020-05-27 [1] CRAN (R 4.0.2)                     
#>    desc           1.2.0   2018-05-01 [1] CRAN (R 4.0.2)                     
#>    devtools       2.3.2   2020-09-18 [1] CRAN (R 4.0.2)                     
#>    digest         0.6.25  2020-02-23 [1] CRAN (R 4.0.2)                     
#>    dplyr        * 1.0.2   2020-08-18 [1] CRAN (R 4.0.2)                     
#>    ellipsis       0.3.1   2020-05-15 [1] CRAN (R 4.0.2)                     
#>    evaluate       0.14    2019-05-28 [1] CRAN (R 4.0.2)                     
#>    fansi          0.4.1   2020-01-08 [1] CRAN (R 4.0.2)                     
#>    fastmatch      1.1-0   2017-01-28 [1] CRAN (R 4.0.0)                     
#>    flatxml        0.1.0   2020-07-24 [1] CRAN (R 4.0.2)                     
#>    forcats      * 0.5.0   2020-03-01 [1] CRAN (R 4.0.2)                     
#>    fs             1.5.0   2020-07-31 [1] CRAN (R 4.0.2)                     
#>    generics       0.0.2   2018-11-29 [1] CRAN (R 4.0.2)                     
#>    ggplot2      * 3.3.2   2020-06-19 [1] CRAN (R 4.0.2)                     
#>    glue           1.4.2   2020-08-27 [1] CRAN (R 4.0.2)                     
#>    googledrive    1.0.1   2020-05-05 [1] CRAN (R 4.0.2)                     
#>    gtable         0.3.0   2019-03-25 [1] CRAN (R 4.0.2)                     
#>    haven          2.3.1   2020-06-01 [1] CRAN (R 4.0.2)                     
#>    hms            0.5.3   2020-01-08 [1] CRAN (R 4.0.2)                     
#>    htmltools      0.5.0   2020-06-16 [1] CRAN (R 4.0.2)                     
#>    httr           1.4.2   2020-07-20 [1] CRAN (R 4.0.2)                     
#>    igraph         1.2.5   2020-03-19 [1] CRAN (R 4.0.2)                     
#>    jsonlite       1.7.1   2020-09-07 [1] CRAN (R 4.0.2)                     
#>    knitr          1.30    2020-09-22 [1] CRAN (R 4.0.2)                     
#>    lattice        0.20-41 2020-04-02 [2] CRAN (R 4.0.2)                     
#>    lifecycle      0.2.0   2020-03-06 [1] CRAN (R 4.0.2)                     
#>    lubridate      1.7.9   2020-06-08 [1] CRAN (R 4.0.2)                     
#>    magrittr       1.5     2014-11-22 [1] CRAN (R 4.0.2)                     
#>    Matrix         1.2-18  2019-11-27 [2] CRAN (R 4.0.2)                     
#>    memoise        1.1.0   2017-04-21 [1] CRAN (R 4.0.2)                     
#>    modelr         0.1.8   2020-05-19 [1] CRAN (R 4.0.2)                     
#>    munsell        0.5.0   2018-06-12 [1] CRAN (R 4.0.2)                     
#>    pillar         1.4.6   2020-07-10 [1] CRAN (R 4.0.2)                     
#>    pipian         0.2.3   2020-08-11 [1] Github (paithiov909/pipian@7097360)
#>    pkgbuild       1.1.0   2020-07-13 [1] CRAN (R 4.0.2)                     
#>    pkgconfig      2.0.3   2019-09-22 [1] CRAN (R 4.0.2)                     
#>    pkgload        1.1.0   2020-05-29 [1] CRAN (R 4.0.2)                     
#>    prettyunits    1.1.1   2020-01-24 [1] CRAN (R 4.0.2)                     
#>    processx       3.4.4   2020-09-03 [1] CRAN (R 4.0.2)                     
#>    ps             1.3.4   2020-08-11 [1] CRAN (R 4.0.2)                     
#>    purrr        * 0.3.4   2020-04-17 [1] CRAN (R 4.0.2)                     
#>    quanteda       2.1.2   2020-09-23 [1] CRAN (R 4.0.2)                     
#>    R.cache        0.14.0  2019-12-06 [1] CRAN (R 4.0.2)                     
#>    R.methodsS3    1.8.1   2020-08-26 [1] CRAN (R 4.0.2)                     
#>    R.oo           1.24.0  2020-08-26 [1] CRAN (R 4.0.2)                     
#>    R.utils        2.10.1  2020-08-26 [1] CRAN (R 4.0.2)                     
#>    R6             2.4.1   2019-11-12 [1] CRAN (R 4.0.2)                     
#>    Rcpp           1.0.5   2020-07-06 [1] CRAN (R 4.0.2)                     
#>  D RcppParallel   5.0.2   2020-06-24 [1] CRAN (R 4.0.2)                     
#>    RcppProgress   0.4.2   2020-02-06 [1] CRAN (R 4.0.2)                     
#>    readr        * 1.3.1   2018-12-21 [1] CRAN (R 4.0.2)                     
#>    readtext       0.80    2020-09-22 [1] CRAN (R 4.0.2)                     
#>    readxl         1.3.1   2019-03-13 [1] CRAN (R 4.0.2)                     
#>    remotes        2.2.0   2020-07-21 [1] CRAN (R 4.0.2)                     
#>    reprex         0.3.0   2019-05-16 [1] CRAN (R 4.0.2)                     
#>    reticulate     1.16    2020-05-27 [1] CRAN (R 4.0.2)                     
#>  D rJava          0.9-13  2020-07-06 [1] CRAN (R 4.0.2)                     
#>    rlang          0.4.7   2020-07-09 [1] CRAN (R 4.0.2)                     
#>    rmarkdown      2.3     2020-06-18 [1] CRAN (R 4.0.2)                     
#>    rprojroot      1.3-2   2018-01-03 [1] CRAN (R 4.0.2)                     
#>    rstudioapi     0.11    2020-02-07 [1] CRAN (R 4.0.2)                     
#>    rvest          0.3.6   2020-07-25 [1] CRAN (R 4.0.2)                     
#>    scales         1.1.1   2020-05-11 [1] CRAN (R 4.0.2)                     
#>    sessioninfo    1.1.1   2018-11-05 [1] CRAN (R 4.0.2)                     
#>    spacyr         1.2.1   2020-03-04 [1] CRAN (R 4.0.2)                     
#>    stopwords      2.0     2020-04-14 [1] CRAN (R 4.0.2)                     
#>    stringdist     0.9.6   2020-07-16 [1] CRAN (R 4.0.2)                     
#>    stringi        1.5.3   2020-09-09 [1] CRAN (R 4.0.2)                     
#>    stringr      * 1.4.0   2019-02-10 [1] CRAN (R 4.0.2)                     
#>    styler         1.3.2   2020-02-23 [1] CRAN (R 4.0.2)                     
#>    tangela        0.0.3   2020-07-26 [1] local                              
#>    testthat       2.3.2   2020-03-02 [1] CRAN (R 4.0.2)                     
#>    textmineR      3.0.4   2019-04-18 [1] CRAN (R 4.0.2)                     
#>    tibble       * 3.0.3   2020-07-10 [1] CRAN (R 4.0.2)                     
#>    tidyr        * 1.1.2   2020-08-27 [1] CRAN (R 4.0.2)                     
#>    tidyselect     1.1.0   2020-05-11 [1] CRAN (R 4.0.2)                     
#>    tidyverse    * 1.3.0   2019-11-21 [1] CRAN (R 4.0.2)                     
#>    udpipe         0.8.3   2019-07-05 [1] CRAN (R 4.0.2)                     
#>    usethis        1.6.3   2020-09-17 [1] CRAN (R 4.0.2)                     
#>    utf8           1.1.4   2018-05-24 [1] CRAN (R 4.0.2)                     
#>    vctrs          0.3.4   2020-08-29 [1] CRAN (R 4.0.2)                     
#>    viridisLite    0.3.0   2018-02-01 [1] CRAN (R 4.0.2)                     
#>    withr          2.3.0   2020-09-22 [1] CRAN (R 4.0.2)                     
#>    xfun           0.17    2020-09-09 [1] CRAN (R 4.0.2)                     
#>    xml2           1.3.2   2020-04-23 [1] CRAN (R 4.0.2)                     
#>    yaml           2.2.1   2020-02-01 [1] CRAN (R 4.0.0)                     
#> 
#> [1] C:/Users/user/Documents/R/win-library/4.0
#> [2] C:/Program Files/R/R-4.0.2/library
#> 
#>  D -- DLL MD5 mismatch, broken installation.
```
