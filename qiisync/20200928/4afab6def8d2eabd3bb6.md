---
ID: 4afab6def8d2eabd3bb6
Title: 【R】spacyr・cleanNLPのデモ
Tags: R,Anaconda,reticulate,spaCy
Author: Kato Akiru
Private: false
---

## spacyr

### spacyrについて

[spaCy](https://spacy.io/)を{reticulate}経由で呼ぶRパッケージです。

[Wrapper to the spaCy NLP Library • spacyr](https://spacyr.quanteda.io/)

spaCyは2.3で日本語のモデルが利用できるようになったらしいです。

### Initialize

簡単に導入できるのですが、Windows環境だけの罠として以下の手順はRを管理者権限で実行する必要があります。Rstudioならショートカットを右クリックから「管理者として実行」です。

> For Windows, you need to run R as an administrator to make installation work properly. To do so, right click the RStudio icon (or R desktop icon) and select “Run as administrator” when launching R.

管理者として以下のスクリプトを実行すると、`spacy_condaenv`という名前のcondaenv内にspaCyと日本語の`ja_core_news_md`というモデルがダウンロードされます

``` r
spacyr::spacy_install()
spacyr::spacy_download_langmodel("ja_core_news_md")
```

モデルをロードして使います。


```r
spacyr::spacy_initialize(model = "ja_core_news_md")
#> Found 'spacy_condaenv'. spacyr will use this environment
#> successfully initialized (spaCy Version: 2.3.5, language model: ja_core_news_md)
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

[cleanNLP: A Tidy Data Model for Natural Language Processing | cleanNLP](https://statsmaths.github.io/cleanNLP/)

UDPipeについてはふつうに{udpipe}をバックエンドとして使っています。spaCyとCoreNLPについてはpipでPythonライブラリを別途導入して、それをバックエンドとして使うようです（{spacyr}や{coreNLP}とは無関係）。spaCyについてはcondaenvのなかにすでにモデルをダウンロードしてあるのでそれを利用できないか試してみたのですが、手元ではうまく動かせませんでした。実質的に{udpipe}のラッパーという感じです。

### Initialize

`cleanNLP::cnlp_init_udpipe`だけで動くようになります。脳死でも使えて便利。


```r
cleanNLP::cnlp_init_udpipe(model_name = "japanese")
#> Downloading udpipe model from https://raw.githubusercontent.com/jwijffels/udpipe.models.ud.2.5/master/inst/udpipe-ud-2.5-191206/japanese-gsd-ud-2.5-191206.udpipe to C:/Users/user/Documents/R/win-library/4.0/cleanNLP/extdata/japanese-gsd-ud-2.5-191206.udpipe
#>  - This model has been trained on version 2.5 of data from https://universaldependencies.org
#>  - The model is distributed under the CC-BY-SA-NC license: https://creativecommons.org/licenses/by-nc-sa/4.0
#>  - Visit https://github.com/jwijffels/udpipe.models.ud.2.5 for model license details.
#>  - For a list of all models and their licenses (most models you can download with this package have either a CC-BY-SA or a CC-BY-SA-NC license) read the documentation at ?udpipe_download_model. For building your own models: visit the documentation by typing vignette('udpipe-train', package = 'udpipe')
#> Downloading finished, model stored at 'C:/Users/user/Documents/R/win-library/4.0/cleanNLP/extdata/japanese-gsd-ud-2.5-191206.udpipe'
```

### 使用例

Windows環境なので文字コードをUTF-8に変換して渡す必要があります。


```r
annotation <- cleanNLP::cnlp_annotate(input = iconv("望遠鏡で泳ぐ彼女を見た", to = "UTF-8"))
annotation$token
#> # A tibble: 7 x 11
#>   doc_id   sid tid   token  token_with_ws lemma  upos  xpos  feats tid_source relation
#> *  <int> <int> <chr> <chr>  <chr>         <chr>  <chr> <chr> <chr> <chr>      <chr>   
#> 1      1     1 1     望遠鏡 望遠鏡        望遠鏡 NOUN  NN    <NA>  6          obl     
#> 2      1     1 2     で     で            で     ADP   PS    <NA>  1          case    
#> 3      1     1 3     泳ぐ   泳ぐ          泳ぐ   VERB  VV    <NA>  4          acl     
#> 4      1     1 4     彼女   彼女          彼女   PRON  NP    <NA>  6          obj     
#> 5      1     1 5     を     を            を     ADP   PS    <NA>  4          case    
#> 6      1     1 6     見     見            見る   VERB  VV    <NA>  0          root    
#> 7      1     1 7     た     た            た     AUX   AV    <NA>  6          aux
```

## セッション情報

Windows10、Miniconda3 (Python 3.7.6 64-bit) です。


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
#>  cleanNLP      3.0.3   2020-10-13 [1] CRAN (R 4.0.3)
#>  cli           2.2.0   2020-11-20 [1] CRAN (R 4.0.3)
#>  crayon        1.3.4   2017-09-16 [1] CRAN (R 4.0.2)
#>  data.table    1.13.6  2020-12-30 [1] CRAN (R 4.0.3)
#>  digest        0.6.27  2020-10-24 [1] CRAN (R 4.0.3)
#>  ellipsis      0.3.1   2020-05-15 [1] CRAN (R 4.0.2)
#>  evaluate      0.14    2019-05-28 [1] CRAN (R 4.0.2)
#>  fansi         0.4.2   2021-01-15 [1] CRAN (R 4.0.3)
#>  glue          1.4.2   2020-08-27 [1] CRAN (R 4.0.2)
#>  htmltools     0.5.1   2021-01-12 [1] CRAN (R 4.0.3)
#>  jsonlite      1.7.2   2020-12-09 [1] CRAN (R 4.0.3)
#>  knitr         1.30    2020-09-22 [1] CRAN (R 4.0.2)
#>  lattice       0.20-41 2020-04-02 [2] CRAN (R 4.0.3)
#>  lifecycle     0.2.0   2020-03-06 [1] CRAN (R 4.0.2)
#>  magrittr      2.0.1   2020-11-17 [1] CRAN (R 4.0.3)
#>  Matrix        1.2-18  2019-11-27 [2] CRAN (R 4.0.3)
#>  pillar        1.4.7   2020-11-20 [1] CRAN (R 4.0.3)
#>  pkgconfig     2.0.3   2019-09-22 [1] CRAN (R 4.0.2)
#>  purrr         0.3.4   2020-04-17 [1] CRAN (R 4.0.2)
#>  R.cache       0.14.0  2019-12-06 [1] CRAN (R 4.0.2)
#>  R.methodsS3   1.8.1   2020-08-26 [1] CRAN (R 4.0.2)
#>  R.oo          1.24.0  2020-08-26 [1] CRAN (R 4.0.2)
#>  R.utils       2.10.1  2020-08-26 [1] CRAN (R 4.0.2)
#>  rappdirs      0.3.1   2016-03-28 [1] CRAN (R 4.0.2)
#>  Rcpp          1.0.6   2021-01-15 [1] CRAN (R 4.0.3)
#>  reticulate    1.18    2020-10-25 [1] CRAN (R 4.0.3)
#>  rlang         0.4.10  2020-12-30 [1] CRAN (R 4.0.3)
#>  rmarkdown     2.6     2020-12-14 [1] CRAN (R 4.0.3)
#>  rstudioapi    0.13    2020-11-12 [1] CRAN (R 4.0.3)
#>  sessioninfo   1.1.1   2018-11-05 [1] CRAN (R 4.0.2)
#>  spacyr        1.2.1   2020-03-04 [1] CRAN (R 4.0.2)
#>  stringi       1.5.3   2020-09-09 [1] CRAN (R 4.0.2)
#>  stringr       1.4.0   2019-02-10 [1] CRAN (R 4.0.2)
#>  styler        1.3.2   2020-02-23 [1] CRAN (R 4.0.2)
#>  tibble        3.0.5   2021-01-15 [1] CRAN (R 4.0.3)
#>  udpipe        0.8.5   2020-12-10 [1] CRAN (R 4.0.3)
#>  utf8          1.1.4   2018-05-24 [1] CRAN (R 4.0.2)
#>  vctrs         0.3.6   2020-12-17 [1] CRAN (R 4.0.3)
#>  withr         2.4.0   2021-01-16 [1] CRAN (R 4.0.3)
#>  xfun          0.20    2021-01-06 [1] CRAN (R 4.0.3)
#>  yaml          2.2.1   2020-02-01 [1] CRAN (R 4.0.0)
#> 
#> [1] C:/Users/user/Documents/R/win-library/4.0
#> [2] C:/Program Files/R/R-4.0.3/library
```

