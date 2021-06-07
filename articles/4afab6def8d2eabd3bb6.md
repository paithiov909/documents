---
title: '【R】spacyr・cleanNLPのデモ'
emoji: '🌿'
type: 'tech'
topics: ['r','spacy','sudachi','ginza']
published: true
author: 'paithiov909'
canonical: 'https://zenn.dev/paithiov909/articles/4afab6def8d2eabd3bb6'
---

## spacyr

### spacyrについて

[spaCy](https://spacy.io/)をreticulate経由で呼ぶRパッケージです。

https://spacyr.quanteda.io/

基本的にpipからインストールできるモデルを使ってPOS tagging・依存関係ラベリング・固有表現抽出をやるためのものです。

自分でpipelineをいじったり、モデルの再学習したり、自前のモデルを読み込んだりとかは今のところできません。~~そういうのは自分でPython書いてやろうね~~

### メモ

- [ここのリスト](http://spacyr.quanteda.io/articles/using_spacyr.html#using-other-language-models)にはないですが、spaCyは2.3で日本語のモデルが公式に追加されたので、それ以降のバージョンでは日本語のモデルも使えます。
- [このissue](http://spacyr.quanteda.io/reference/spacy_install.html#spacy-version-issues)はすでにクローズされているので、たぶんふつうに最新版のspaCyを入れても大丈夫です。
- `nlp.pipe(n_process=2)`みたいなマルチプロセスの設定は渡せません。Python側でマルチスレッド対応はしています（実体としてはspaCyをラップしたPythonスクリプトをitertoolsで回すことになる）。
- spaCyは適当にセットアップして`spacy.pefer.gpu()`とかするとGPUを使えるのですが、spacyrからGPU利用はできないっぽいです（そもそも`spacyr::spacy_install`からだとGPU使うやつは現状入れることができない）。

### 使い方

#### (1) condaで入れる場合

ふつうにMiniconda環境でやるとPythonライブラリはぜんぶcondaでインストールされます（それはそう）。

簡単に導入できるのですが、Windows環境だけの罠として、以下の手順はRを管理者権限で実行する必要があります。RStudioならショートカットを右クリックから「管理者として実行」です。

管理者として以下のスクリプトを実行すると、`spacy_condaenv`という名前のcondaenv内にspaCyと日本語の`ja_core_news_sm`というモデルがダウンロードされます（[日本語モデル](https://spacy.io/models/ja)は_sm, _md, _lgがある）。

なお、Windows以外ではvirtualenvで環境を切ることもできるので、その場合は`spacy_install`を`spacy_install_virtualenv`に読み替えましょう。

```r
spacyr::spacy_install(lang_models = "ja_core_news_sm")
```

モデルをロードして使います。

```r
spacyr::spacy_initialize(model = "ja_core_news_sm")
```

これは余談ですが、spaCy>2.3では日本語のtokenizerがfugashiからsudachipy（sudachidict-core）に移行しているので、spaCyを入れると自動的にsudachipyも入ります（spaCyにはnatto-pyへの依存もあるが、これは韓国語のためのもので、日本語のモデルの利用にMeCabへの依存はない、はず）。

とくにサイズの大きいモデルを使う場合、モデルをメモリに読み込んだままだとよろしくない場合があるので、使い終わったら以下のおまじないを実行するとよいらしいです。

```r
spacyr::spacy_finalize()
# 一度finalizeすると、再度initializeするにはRセッションを起ちあげ直す必要があるっぽい
```

#### (2) GiNZAとあわせてpipで入れる場合

[GiNZA](https://megagonlabs.github.io/ginza/)のモデルを使いたいときなど、conda環境であってもpipを使いたい場合があります。condaとpipを混ぜるとMiniconda環境そのものが壊れることがあるという報告がたびたび聞かれるのでオススメはしませんが、環境を切ってからpipだけを使うようにしてcondaと混ぜなければいちおう動かすことができます。

spacyrの最新のCRANリリース（1.2.1）ではspaCyのv2.xを指定して入れることができないので、GiNZAのモデルを使いたい場合は、次のようにします。

注意点として、Rはやはり管理者権限で実行しなれけばならないのと、Windows環境でpipから入れる場合だと依存する何かをビルドするのに[Visual C++](https://visualstudio.microsoft.com/ja/downloads/)が必要になります。また、すでに`spacy_condaenv`をcondaで切っていると環境が混ざってしまうので、その場合はその環境ごと消してまっさらな状態にするか、適宜別の名前の環境に読み替えてください。

```r
require(reticulate)
conda_create("spacy_condaenv", python_version = "3.9")
conda_install("spacy_condaenv", packages = "spacy<3", pip = TRUE)

require(spacyr)
spacy_download_langmodel("ja_core_news_sm")

conda_install("spacy_condaenv", packages = "ginza", pip = TRUE)
```

これでGiNZAのモデルをロードできるようになります。`reticulate::import("ginza")`とかせずにいきなり呼んで大丈夫です。

```r
spacyr::spacy_initialize(model = "ja_ginza")
```

使い終わったらfinalizeするのも同様です。

```r
spacyr::spacy_finalize()
```

#### 解析する

`ja_ginza`をロードしたうえで使ってみます。taggerのレイヤだけ呼ぶのはこうです。


```r
spacyr::spacy_tokenize(
  c("望遠鏡で泳ぐ彼女を見た", "頭が赤い魚を食べる猫", "外国人参政権")
)
#> $text1
#> [1] "望遠鏡" "で"     "泳ぐ"   "彼女"   "を"     "見"     "た"    
#> 
#> $text2
#> [1] "頭"     "が"     "赤い"   "魚"     "を"     "食べる" "猫"    
#> 
#> $text3
#> [1] "外国人参政権"
```

ラベリングする場合は`spacy_parse`という関数を呼びます。日本語のモデルではlemmatizationはうまく動かないよという怒られが発生するので`lemma = FALSE`とします。


```r
spacyr::spacy_parse(
  c("望遠鏡で泳ぐ彼女を見た", "頭が赤い魚を食べる猫", "外国人参政権"),
  lemma = FALSE
)
#>    doc_id sentence_id token_id        token  pos         entity
#> 1   text1           1        1       望遠鏡 NOUN               
#> 2   text1           1        2           で  ADP               
#> 3   text1           1        3         泳ぐ VERB               
#> 4   text1           1        4         彼女 PRON               
#> 5   text1           1        5           を  ADP               
#> 6   text1           1        6           見 VERB               
#> 7   text1           1        7           た  AUX               
#> 8   text2           1        1           頭 NOUN  Animal_Part_B
#> 9   text2           1        2           が  ADP               
#> 10  text2           1        3         赤い  ADJ Nature_Color_B
#> 11  text2           1        4           魚 NOUN               
#> 12  text2           1        5           を  ADP               
#> 13  text2           1        6       食べる VERB               
#> 14  text2           1        7           猫 NOUN       Mammal_B
#> 15  text3           1        1 外国人参政権 NOUN
```

'Universal Dependencies'なので係り受けを出せます。


```r
spacyr::spacy_parse(
  c("望遠鏡で泳ぐ彼女を見た", "頭が赤い魚を食べる猫", "外国人参政権"),
  dependency = TRUE,
  lemma = FALSE,
  pos = FALSE
)
#>    doc_id sentence_id token_id        token head_token_id dep_rel         entity
#> 1   text1           1        1       望遠鏡             3     obl               
#> 2   text1           1        2           で             1    case               
#> 3   text1           1        3         泳ぐ             4     acl               
#> 4   text1           1        4         彼女             6     obj               
#> 5   text1           1        5           を             4    case               
#> 6   text1           1        6           見             6    ROOT               
#> 7   text1           1        7           た             6     aux               
#> 8   text2           1        1           頭             3   nsubj  Animal_Part_B
#> 9   text2           1        2           が             1    case               
#> 10  text2           1        3         赤い             4     acl Nature_Color_B
#> 11  text2           1        4           魚             6     obj               
#> 12  text2           1        5           を             4    case               
#> 13  text2           1        6       食べる             7     acl               
#> 14  text2           1        7           猫             7    ROOT       Mammal_B
#> 15  text3           1        1 外国人参政権             1    ROOT
```

`noun_chunks`を抽出するやつとNamed Entity Recognitionはそれ用の関数があります。
`spacyr::spacy_parse`の戻り値はデータフレームなのでdplyrなどを使ってR側でやってもよいのですが、Python側に任せたほうが速い環境ではこちらが便利かもしれません。


```r
spacyr::spacy_parse(
  c("望遠鏡で泳ぐ彼女を見た", "頭が赤い魚を食べる猫", "外国人参政権"),
  lemma = FALSE,
  entity = TRUE
) %>%
  spacyr::entity_extract()
#>   doc_id sentence_id entity entity_type
#> 1  text2           1     頭      Animal
#> 2  text2           1   赤い      Nature
#> 3  text2           1     猫      Mammal
```

#### 依存関係のグラフ表示

[spacy-streamlit](https://github.com/explosion/spacy-streamlit)とかで出せるようなグラフはないんですかと思うじゃないですか。あります。

[textplot](https://github.com/bnosac/textplot)というパッケージを使います。ふつうにCRANから入れられます。

`textplot::textplot_dependencyparser`は本来はudpipeの結果のデータフレームを引数に取りますが、ぜんぶのカラムはいらないので、ちょっと加工するとspacyrの結果もプロットできます。


```r
spacyr::spacy_parse(
  c("望遠鏡で泳ぐ彼女を見た", "頭が赤い魚を食べる猫", "外国人参政権"),
  dependency = TRUE,
  lemma = FALSE,
  pos = TRUE # UDにおけるupos
) %>%
  dplyr::rename(upos = pos) %>% # ここでuposにrenameする
  dplyr::filter(doc_id == "text1") %>% # 1文ごとに渡す必要がある
  textplot::textplot_dependencyparser()
```

![spacy_plot-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/228173/f8ee77ef-3faa-f2b7-7b91-a364cc9529c0.png)

## cleanNLP

### cleanNLPについて

UDPipe、spaCy、CoreNLPをtidyに使えるよ！というRパッケージです。

https://statsmaths.github.io/cleanNLP/

UDPipeについてはふつうに[udpipe](https://github.com/bnosac/udpipe)をバックエンドとして使っています。spaCyとCoreNLPについてはpipでPythonライブラリを別途導入して、それをバックエンドとして使うようです（spacyrや[coreNLP](https://github.com/statsmaths/coreNLP)とは無関係）。実質的にはudpipeのラッパーです。

### 使い方

`cleanNLP::cnlp_init_udpipe`で動くようになります。Windows環境では文字コードをUTF-8に変換して渡す必要があります。

```r
cleanNLP::cnlp_init_udpipe(model_name = "japanese")
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

ぶっちゃけudpipeとたいして変わらない気がしますが、cleanNLPではいちおうロードしたモデルをpackageのnamespace内にある環境で管理してくれます。

udpipeだと以下に相当するので、ちょっとだけ楽に書けます。

```r
model_path <- udpipe::udpipe_download_model("japanese")
model <- udpipe::udpipe_load_model(model_path$file_model)
udpipe::udpipe(iconv("これをこうやって使う", to = "UTF-8"), model)
```

## どれがなんのモデルなのか的な話

たぶん2021年5月現在。日本語UDそのものについては[これ](https://www.jstage.jst.go.jp/article/jnlp/26/1/26_3/_article/-char/ja/)を読むとよいです。

|              | 学習コーパス               | 語数    | モデルのライセンス |
| ------------ | ----------------------- | ------ | ----------- |
| spaCy>2.3    | UD Japanese-GSD v2.6-NE | 186k   | CC BY-SA    |
| GiNZA>4      | UD Japanese-BCCWJ v2.6  | 1,098k | MIT         |
| UDPipe 1     | UD Japanese-GSD v2.5    | 186k   | CC BY-NC-SA |

- ここの「語数」は元の学習コーパスのサイズのことで、モデルの語彙とかではない。
- GiNZA v4のNERモデルには別のアノテーション済みコーパス（[GSK2014-A (2019) BCCWJ edition](https://github.com/megagonlabs/ginza#gsk2014-a-2019-bccwj-edition)）が利用されているらしい。
- UDPipeについてはUD 2.6で学習したモデルがすでに公開されているが、Rパッケージからは利用できない。

## 参考

- [Using Tidytext and SpacyR in R to do Sentiment Analysis on the COVID-19 Update Speeches by the President of Ghana. | Ghana Data Stuff](https://ghanadatastuff.com/post/text_analysis_covid/)
- [spaCy からたどる最近の日本語自然言語処理ライブラリの調査 | ハカセノオト](https://hakasenote.hnishi.com/2020/20200810-spacy-japanese-nlp/)
- [固有表現抽出のアノテーションデータについて - NLP太郎のブログ](https://kzinmr.hatenablog.com/entry/2020/10/06/162659)

## セッション情報

Windows10, Miniconda3（Minicondaそのものの環境はPython=3.7.9, conda=4.9.2）で試しています。

```r
sessioninfo::session_info()
#> - Session info --------------------------------------------------------------------------------
#>  setting  value                       
#>  version  R version 4.0.2 (2020-06-22)
#>  os       Windows 10 x64              
#>  system   x86_64, mingw32             
#>  ui       RStudio                     
#>  language (EN)                        
#>  collate  Japanese_Japan.932          
#>  ctype    Japanese_Japan.932          
#>  tz       Asia/Tokyo                  
#>  date     2021-05-24                  
#> 
#> - Packages ------------------------------------------------------------------------------------
#>  package      * version date       lib source        
#>  assertthat     0.2.1   2019-03-21 [1] CRAN (R 4.0.2)
#>  backports      1.2.1   2020-12-09 [1] CRAN (R 4.0.3)
#>  bslib          0.2.4   2021-01-25 [1] CRAN (R 4.0.3)
#>  cli            2.5.0   2021-04-26 [1] CRAN (R 4.0.5)
#>  colorspace     2.0-1   2021-05-04 [1] CRAN (R 4.0.5)
#>  crayon         1.4.1   2021-02-08 [1] CRAN (R 4.0.2)
#>  data.table     1.14.0  2021-02-21 [1] CRAN (R 4.0.2)
#>  DBI            1.1.1   2021-01-15 [1] CRAN (R 4.0.3)
#>  digest         0.6.27  2020-10-24 [1] CRAN (R 4.0.3)
#>  dplyr          1.0.6   2021-05-05 [1] CRAN (R 4.0.2)
#>  ellipsis       0.3.2   2021-04-29 [1] CRAN (R 4.0.2)
#>  evaluate       0.14    2019-05-28 [1] CRAN (R 4.0.2)
#>  fansi          0.4.2   2021-01-15 [1] CRAN (R 4.0.3)
#>  farver         2.1.0   2021-02-28 [1] CRAN (R 4.0.2)
#>  generics       0.1.0   2020-10-31 [1] CRAN (R 4.0.3)
#>  ggforce        0.3.3   2021-03-05 [1] CRAN (R 4.0.4)
#>  ggplot2        3.3.3   2020-12-30 [1] CRAN (R 4.0.3)
#>  ggraph         2.0.5   2021-02-23 [1] CRAN (R 4.0.4)
#>  ggrepel        0.9.1   2021-01-15 [1] CRAN (R 4.0.3)
#>  glue           1.4.2   2020-08-27 [1] CRAN (R 4.0.3)
#>  graphlayouts   0.7.1   2020-10-26 [1] CRAN (R 4.0.4)
#>  gridExtra      2.3     2017-09-09 [1] CRAN (R 4.0.2)
#>  gtable         0.3.0   2019-03-25 [1] CRAN (R 4.0.2)
#>  highr          0.9     2021-04-16 [1] CRAN (R 4.0.5)
#>  htmltools      0.5.1.1 2021-01-22 [1] CRAN (R 4.0.3)
#>  igraph         1.2.6   2020-10-06 [1] CRAN (R 4.0.3)
#>  jquerylib      0.1.4   2021-04-26 [1] CRAN (R 4.0.5)
#>  jsonlite       1.7.2   2020-12-09 [1] CRAN (R 4.0.3)
#>  knitr          1.33    2021-04-24 [1] CRAN (R 4.0.5)
#>  labeling       0.4.2   2020-10-20 [1] CRAN (R 4.0.3)
#>  lattice        0.20-41 2020-04-02 [2] CRAN (R 4.0.2)
#>  lifecycle      1.0.0   2021-02-15 [1] CRAN (R 4.0.2)
#>  magrittr     * 2.0.1   2020-11-17 [1] CRAN (R 4.0.3)
#>  MASS           7.3-54  2021-05-03 [1] CRAN (R 4.0.5)
#>  Matrix         1.3-3   2021-05-04 [1] CRAN (R 4.0.2)
#>  munsell        0.5.0   2018-06-12 [1] CRAN (R 4.0.2)
#>  pillar         1.6.0   2021-04-13 [1] CRAN (R 4.0.5)
#>  pkgconfig      2.0.3   2019-09-22 [1] CRAN (R 4.0.2)
#>  png            0.1-7   2013-12-03 [1] CRAN (R 4.0.3)
#>  polyclip       1.10-0  2019-03-14 [1] CRAN (R 4.0.3)
#>  purrr          0.3.4   2020-04-17 [1] CRAN (R 4.0.2)
#>  R.cache        0.15.0  2021-04-30 [1] CRAN (R 4.0.5)
#>  R.methodsS3    1.8.1   2020-08-26 [1] CRAN (R 4.0.3)
#>  R.oo           1.24.0  2020-08-26 [1] CRAN (R 4.0.3)
#>  R.utils        2.10.1  2020-08-26 [1] CRAN (R 4.0.3)
#>  R6             2.5.0   2020-10-28 [1] CRAN (R 4.0.3)
#>  rappdirs       0.3.3   2021-01-31 [1] CRAN (R 4.0.3)
#>  Rcpp           1.0.6   2021-01-15 [1] CRAN (R 4.0.3)
#>  rematch2       2.1.2   2020-05-01 [1] CRAN (R 4.0.2)
#>  reticulate     1.20    2021-05-03 [1] CRAN (R 4.0.5)
#>  rlang          0.4.11  2021-04-30 [1] CRAN (R 4.0.5)
#>  rmarkdown      2.8     2021-05-07 [1] CRAN (R 4.0.5)
#>  sass           0.3.1   2021-01-24 [1] CRAN (R 4.0.3)
#>  scales         1.1.1   2020-05-11 [1] CRAN (R 4.0.2)
#>  sessioninfo    1.1.1   2018-11-05 [1] CRAN (R 4.0.2)
#>  spacyr         1.2.1   2020-03-04 [1] CRAN (R 4.0.2)
#>  stringi        1.6.1   2021-05-10 [1] CRAN (R 4.0.2)
#>  stringr        1.4.0   2019-02-10 [1] CRAN (R 4.0.2)
#>  styler         1.4.1   2021-03-30 [1] CRAN (R 4.0.5)
#>  textplot       0.1.4   2020-10-10 [1] CRAN (R 4.0.5)
#>  tibble         3.1.1   2021-04-18 [1] CRAN (R 4.0.2)
#>  tidygraph      1.2.0   2020-05-12 [1] CRAN (R 4.0.4)
#>  tidyr          1.1.3   2021-03-03 [1] CRAN (R 4.0.4)
#>  tidyselect     1.1.1   2021-04-30 [1] CRAN (R 4.0.2)
#>  tweenr         1.0.2   2021-03-23 [1] CRAN (R 4.0.2)
#>  utf8           1.2.1   2021-03-12 [1] CRAN (R 4.0.2)
#>  vctrs          0.3.8   2021-04-29 [1] CRAN (R 4.0.2)
#>  viridis        0.6.0   2021-04-15 [1] CRAN (R 4.0.5)
#>  viridisLite    0.4.0   2021-04-13 [1] CRAN (R 4.0.5)
#>  withr          2.4.2   2021-04-18 [1] CRAN (R 4.0.5)
#>  xfun           0.22    2021-03-11 [1] CRAN (R 4.0.4)
#>  yaml           2.2.1   2020-02-01 [1] CRAN (R 4.0.0)
#> 
#> [1] C:/Users/user/Documents/R/win-library/4.0
#> [2] C:/Program Files/R/R-4.0.2/library
```

