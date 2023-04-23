---
title: "Rで日本語の単語分散表現を使ったテキスト分類をやる"
emoji: "🥬"
type: "tech"
topics: ["r","自然言語処理","chive","sudachi"]
published: true
---

## この記事でやること

Emil Hvitfeldt氏による[Textrecipes series: Pretrained Word Embedding](https://www.emilhvitfeldt.com/post/textrecipes-series-pretrained-word-embeddings/)を参考に、日本語テキストについて単語分散表現を用いながら2値の分類問題を解きます。

単語分散表現は、[chiVe](https://github.com/WorksApplications/chiVe)のA単位語のみについて学習した資源のひとつである`chive-1.1-mc5-aunit.magnitude`を利用します。

データセットとしては、[JRTEコーパス](https://github.com/megagonlabs/jrte-corpus)の`rhr.tsv`というデータを使います。これは、「また是非利用したいと思いました。」「近くにコンビニもあります。」といったようなごく短いテキスト列に対して、ホテルについての評判であるかどうかのラベルが付与されているデータです。

## 準備

JRTEコーパスは、[ldccr](https://github.com/paithiov909/ldccr)で読み込むことができます。


```r
rhr <- ldccr::read_jrte(keep_rhr = TRUE)
#> Parsing rte.nlp2020_base.tsv...
#> Parsing rte.nlp2020_append.tsv...
#> Parsing rte.lrec2020_surf.tsv...
#> Parsing rte.lrec2020_sem_short.tsv...
#> Parsing rte.lrec2020_sem_long.tsv...
#> Parsing rte.lrec2020_me.tsv...
#> Done.
rhr <- rhr$rhr |>
  dplyr::select(example_id, label, text, usage)

summary(rhr)
#>   example_id                   label          text             usage
#>  Length:5553        reputation    :2069   Length:5553        dev  :1122
#>  Class :character   not_reputation:3484   Class :character   test : 553
#>  Mode  :character                         Mode  :character   train:3878
```

magnitude形式の分散表現は、[apportita](https://github.com/paithiov909/apportita)で読み込むことができます。ここでは、先頭から10,000語の分散表現をtibbleとして切り出して利用することにします。


```r
require(apportita)
#>  要求されたパッケージ apportita をロード中です

conn <- apportita::magnitude("magnitude/chive-1.1-mc5-aunit.magnitude")
dim(conn)
#> [1] 322094    300

embeddings <- apportita::slice_n(conn, 10000)

close(conn)
```

chiVeは[Sudachi](https://github.com/WorksApplications/SudachiPy)の言語資源をもとに学習されているため、テキストはSudachiの辞書に収録されている単語のかたちに寄せて分かち書きにする必要があります。

そのため、形態素解析には、SudachiのRust実装である[sudachi.rs](https://github.com/WorksApplications/sudachi.rs)のラッパーを提供している[fledgingr](https://github.com/yutannihilation/fledgingr)というパッケージを利用します。

`textrecipes::step_tokenize`の`custom_token`引数に指定するために、次のようなラッパー関数を用意しておきます。


```r
tokenize <- \(x) {
  fledgingr::tokenize(x, mode = "A") |>
    dplyr::group_by(id) |>
    dplyr::group_map(~ {
      .x$normalized_form
    })
}

tokenize("新しい朝が来た")
#> [[1]]
#> [1] "新しい" "朝"     "が"     "来る"   "た"
```

## モデルの学習

データにはもともとdev/test/trainに分けられるラベルが付いていますが、ここではそれは使わずに、あらためてtrain/testに分割します。


```r
require(tidymodels)
#>  要求されたパッケージ tidymodels をロード中です
#> ── Attaching packages ─────────────────────────────────────── tidymodels 1.0.0 ──
#> ✔ broom        1.0.0     ✔ recipes      1.0.1
#> ✔ dials        1.0.0     ✔ rsample      1.0.0
#> ✔ dplyr        1.0.9     ✔ tibble       3.1.8
#> ✔ ggplot2      3.3.6     ✔ tidyr        1.2.0
#> ✔ infer        1.0.2     ✔ tune         1.0.0
#> ✔ modeldata    1.0.0     ✔ workflows    1.0.0
#> ✔ parsnip      1.0.0     ✔ workflowsets 1.0.0
#> ✔ purrr        0.3.4     ✔ yardstick    1.0.0
#> ── Conflicts ────────────────────────────────────────── tidymodels_conflicts() ──
#> ✖ purrr::discard() masks scales::discard()
#> ✖ dplyr::filter()  masks stats::filter()
#> ✖ dplyr::lag()     masks stats::lag()
#> ✖ recipes::step()  masks stats::step()
#> • Dig deeper into tidy modeling with R at https://www.tmwr.org
require(textrecipes)
#>  要求されたパッケージ textrecipes をロード中です
tidymodels::tidymodels_prefer()

set.seed(5553)
rhr_split <- initial_split(rhr, strata = label)
rhr_train <- training(rhr_split)
rhr_test <- testing(rhr_split)
```

workflowを定義します。


```r
rhr_rec <-
  recipe(label ~ text, data = rhr_train) |>
  step_text_normalization(text, normalization_form = "nfkc") |>
  step_tokenize(text, custom_token = tokenize) |>
  step_word_embeddings(text, embeddings = embeddings)

rhr_spec <-
  logistic_reg(penalty = tune(), mixture = 1) |>
  set_engine("glmnet")

rhr_wflow <-
  workflow() |>
  add_recipe(rhr_rec) |>
  add_model(rhr_spec)
```

ペナルティを探索します。


```r
rhr_grid <-
  tune_grid(
    rhr_wflow,
    resamples = bootstraps(rhr_train, times = 10),
    grid = grid_regular(penalty(), levels = 30)
  )

autoplot(rhr_grid)
```

![grid](https://storage.googleapis.com/zenn-user-upload/ae0debebda56-20220723.png)


```r
rhr_grid |>
  show_best("roc_auc")
#> # A tibble: 5 × 7
#>    penalty .metric .estimator  mean     n std_err .config
#>      <dbl> <chr>   <chr>      <dbl> <int>   <dbl> <chr>
#> 1 0.00174  roc_auc binary     0.921    10 0.00202 Preprocessor1_Model22
#> 2 0.000788 roc_auc binary     0.921    10 0.00192 Preprocessor1_Model21
#> 3 0.00386  roc_auc binary     0.918    10 0.00205 Preprocessor1_Model23
#> 4 0.000356 roc_auc binary     0.918    10 0.00215 Preprocessor1_Model20
#> 5 0.000161 roc_auc binary     0.914    10 0.00210 Preprocessor1_Model19
```

`last_fit`します。


```r
rhr_wflow <- finalize_workflow(rhr_wflow, select_best(rhr_grid, "roc_auc"))
rhr_last_res <- last_fit(rhr_wflow, rhr_split)
```

試しにモデルを学習してみることが目的なのでいちいち評価しませんが、ROC曲線を描いてみます。


```r
rhr_last_res |>
  collect_predictions() |>
  roc_curve(label, .pred_reputation) |>
  autoplot()
```

![roc_curve](https://storage.googleapis.com/zenn-user-upload/396149364f2d-20220723.png)

混同行列も描いてみます。


```r
rhr_last_res |>
  collect_predictions() |>
  conf_mat(label, .pred_class) |>
  autoplot(type = "heatmap")
```

![conf_mat](https://storage.googleapis.com/zenn-user-upload/93a72360f499-20220723.png)

## セッション情報


```r
sessioninfo::session_info()
#> ─ Session info ────────────────────────────────────────────────────────────────
#>  setting  value
#>  version  R version 4.2.1 (2022-06-23 ucrt)
#>  os       Windows 10 x64 (build 19043)
#>  system   x86_64, mingw32
#>  ui       RStudio
#>  language (EN)
#>  collate  Japanese_Japan.utf8
#>  ctype    Japanese_Japan.utf8
#>  tz       Asia/Tokyo
#>  date     2022-07-23
#>  rstudio  2022.07.0+548 Spotted Wakerobin (desktop)
#>  pandoc   2.18 @ C:/Program Files/RStudio/bin/quarto/bin/tools/ (via rmarkdown)
#>
#> ─ Packages ────────────────────────────────────────────────────────────────────
#>  package      * version        date (UTC) lib source
#>  apportita    * 0.0.1          2022-05-10 [1] https://paithiov909.r-universe.dev (R 4.2.0)
#>  assertthat     0.2.1          2019-03-21 [1] CRAN (R 4.2.0)
#>  backports      1.4.1          2021-12-13 [1] CRAN (R 4.2.0)
#>  bit            4.0.4          2020-08-04 [1] CRAN (R 4.2.0)
#>  bit64          4.0.5          2020-08-30 [1] CRAN (R 4.2.0)
#>  blob           1.2.3          2022-04-10 [1] CRAN (R 4.2.0)
#>  broom        * 1.0.0          2022-07-01 [1] CRAN (R 4.2.0)
#>  bslib          0.4.0          2022-07-16 [1] RSPM (R 4.2.0)
#>  cachem         1.0.6          2021-08-19 [1] CRAN (R 4.2.0)
#>  class          7.3-20         2022-01-16 [2] CRAN (R 4.2.1)
#>  cleanrmd       0.1.0          2022-06-14 [1] RSPM (R 4.2.0)
#>  cli            3.3.0          2022-04-25 [1] CRAN (R 4.2.0)
#>  codetools      0.2-18         2020-11-04 [2] CRAN (R 4.2.1)
#>  colorspace     2.0-3          2022-02-21 [1] CRAN (R 4.2.0)
#>  conflicted     1.1.0          2021-11-26 [1] CRAN (R 4.2.0)
#>  crayon         1.5.1          2022-03-26 [1] CRAN (R 4.2.0)
#>  DBI            1.1.3          2022-06-18 [1] CRAN (R 4.2.0)
#>  dbplyr         2.2.1          2022-06-27 [1] CRAN (R 4.2.0)
#>  dials        * 1.0.0          2022-06-14 [1] CRAN (R 4.2.0)
#>  DiceDesign     1.9            2021-02-13 [1] CRAN (R 4.2.0)
#>  digest         0.6.29         2021-12-01 [1] CRAN (R 4.2.0)
#>  dplyr        * 1.0.9          2022-04-28 [1] CRAN (R 4.2.0)
#>  ellipsis       0.3.2          2021-04-29 [1] CRAN (R 4.2.0)
#>  evaluate       0.15           2022-02-18 [1] CRAN (R 4.2.0)
#>  fansi          1.0.3          2022-03-24 [1] CRAN (R 4.2.0)
#>  farver         2.1.1          2022-07-06 [1] CRAN (R 4.2.1)
#>  fastmap        1.1.0.9000     2022-05-15 [1] https://fastverse.r-universe.dev (R 4.2.0)
#>  fledgingr      0.0.0.9003     2022-07-18 [1] https://yutannihilation.r-universe.dev (R 4.2.1)
#>  foreach        1.5.2          2022-02-02 [1] CRAN (R 4.2.0)
#>  furrr          0.3.0          2022-05-04 [1] CRAN (R 4.2.0)
#>  future         1.27.0         2022-07-22 [1] CRAN (R 4.2.1)
#>  future.apply   1.9.0          2022-04-25 [1] CRAN (R 4.2.0)
#>  generics       0.1.3          2022-07-05 [1] CRAN (R 4.2.0)
#>  ggplot2      * 3.3.6          2022-05-03 [1] CRAN (R 4.2.0)
#>  glmnet       * 4.1-4          2022-04-15 [1] CRAN (R 4.2.0)
#>  globals        0.15.1         2022-06-24 [1] CRAN (R 4.2.0)
#>  glue           1.6.2          2022-02-24 [1] CRAN (R 4.2.0)
#>  gower          1.0.0          2022-02-03 [1] CRAN (R 4.2.0)
#>  GPfit          1.0-8          2019-02-08 [1] CRAN (R 4.2.0)
#>  gtable         0.3.0          2019-03-25 [1] CRAN (R 4.2.0)
#>  hardhat        1.2.0          2022-06-30 [1] CRAN (R 4.2.0)
#>  highr          0.9            2021-04-16 [1] CRAN (R 4.2.0)
#>  hms            1.1.1          2021-09-26 [1] CRAN (R 4.2.0)
#>  htmltools      0.5.3          2022-07-18 [1] CRAN (R 4.2.1)
#>  infer        * 1.0.2          2022-06-10 [1] CRAN (R 4.2.0)
#>  ipred          0.9-13         2022-06-02 [1] CRAN (R 4.2.0)
#>  iterators      1.0.14         2022-02-05 [1] CRAN (R 4.2.0)
#>  jquerylib      0.1.4          2021-04-26 [1] CRAN (R 4.2.0)
#>  jsonlite       1.8.0          2022-02-22 [1] CRAN (R 4.2.0)
#>  knitr          1.39           2022-04-26 [1] CRAN (R 4.2.0)
#>  labeling       0.4.2          2020-10-20 [1] CRAN (R 4.2.0)
#>  lattice        0.20-45        2021-09-22 [2] CRAN (R 4.2.1)
#>  lava           1.6.10         2021-09-02 [1] CRAN (R 4.2.0)
#>  ldccr          0.0.9.20220709 2022-07-09 [1] https://paithiov909.r-universe.dev (R 4.2.1)
#>  lhs            1.1.5          2022-03-22 [1] CRAN (R 4.2.0)
#>  lifecycle      1.0.1          2021-09-24 [1] CRAN (R 4.2.0)
#>  listenv        0.8.0          2019-12-05 [1] CRAN (R 4.2.0)
#>  lubridate      1.8.0.9000     2022-05-14 [1] https://fastverse.r-universe.dev (R 4.2.0)
#>  magrittr       2.0.3.9000     2022-05-13 [1] https://fastverse.r-universe.dev (R 4.2.0)
#>  MASS           7.3-57         2022-04-22 [2] CRAN (R 4.2.1)
#>  Matrix       * 1.4-1          2022-03-23 [2] CRAN (R 4.2.1)
#>  memoise        2.0.1          2021-11-26 [1] CRAN (R 4.2.0)
#>  modeldata    * 1.0.0          2022-07-01 [1] CRAN (R 4.2.1)
#>  munsell        0.5.0          2018-06-12 [1] CRAN (R 4.2.0)
#>  nnet           7.3-17         2022-01-16 [2] CRAN (R 4.2.1)
#>  parallelly     1.32.1         2022-07-21 [1] CRAN (R 4.2.1)
#>  parsnip      * 1.0.0          2022-06-16 [1] CRAN (R 4.2.0)
#>  pillar         1.8.0          2022-07-18 [1] CRAN (R 4.2.1)
#>  pkgconfig      2.0.3          2019-09-22 [1] CRAN (R 4.2.0)
#>  prodlim        2019.11.13     2019-11-17 [1] CRAN (R 4.2.0)
#>  purrr        * 0.3.4          2020-04-17 [1] CRAN (R 4.2.0)
#>  R.cache        0.16.0         2022-07-21 [1] CRAN (R 4.2.1)
#>  R.methodsS3    1.8.2          2022-06-13 [1] CRAN (R 4.2.0)
#>  R.oo           1.25.0         2022-06-12 [1] CRAN (R 4.2.0)
#>  R.utils        2.12.0         2022-06-28 [1] CRAN (R 4.2.0)
#>  R6             2.5.1          2021-08-19 [1] CRAN (R 4.2.0)
#>  rappdirs       0.3.3          2021-01-31 [1] CRAN (R 4.2.0)
#>  Rcpp           1.0.9          2022-07-08 [1] CRAN (R 4.2.1)
#>  RcppSimdJson   0.1.7          2022-02-18 [1] CRAN (R 4.2.0)
#>  readr          2.1.2          2022-01-30 [1] CRAN (R 4.2.0)
#>  recipes      * 1.0.1          2022-07-07 [1] CRAN (R 4.2.0)
#>  rlang          1.0.4          2022-07-12 [1] RSPM (R 4.2.0)
#>  rmarkdown      2.14           2022-04-25 [1] CRAN (R 4.2.0)
#>  rpart          4.1.16         2022-01-24 [2] CRAN (R 4.2.1)
#>  rsample      * 1.0.0          2022-06-24 [1] CRAN (R 4.2.0)
#>  RSQLite        2.2.15         2022-07-17 [1] RSPM (R 4.2.0)
#>  rstudioapi     0.13           2020-11-12 [1] CRAN (R 4.2.0)
#>  sass           0.4.2          2022-07-16 [1] RSPM (R 4.2.0)
#>  scales       * 1.2.0          2022-04-13 [1] CRAN (R 4.2.0)
#>  sessioninfo    1.2.2          2021-12-06 [1] CRAN (R 4.2.0)
#>  shape          1.4.6          2021-05-19 [1] CRAN (R 4.2.0)
#>  stringi      * 1.7.8          2022-07-11 [1] RSPM (R 4.2.0)
#>  stringr        1.4.0.9000     2022-05-14 [1] https://fastverse.r-universe.dev (R 4.2.0)
#>  styler         1.7.0          2022-03-13 [1] CRAN (R 4.2.0)
#>  survival       3.3-1          2022-03-03 [2] CRAN (R 4.2.1)
#>  textrecipes  * 1.0.0          2022-07-02 [1] CRAN (R 4.2.1)
#>  tibble       * 3.1.8          2022-07-22 [1] CRAN (R 4.2.1)
#>  tidymodels   * 1.0.0          2022-07-13 [1] CRAN (R 4.2.1)
#>  tidyr        * 1.2.0          2022-02-01 [1] CRAN (R 4.2.0)
#>  tidyselect     1.1.2          2022-02-21 [1] CRAN (R 4.2.0)
#>  timeDate       4021.104       2022-07-19 [1] RSPM (R 4.2.0)
#>  tune         * 1.0.0          2022-07-07 [1] CRAN (R 4.2.0)
#>  tzdb           0.3.0          2022-03-28 [1] CRAN (R 4.2.0)
#>  utf8           1.2.2          2021-07-24 [1] CRAN (R 4.2.0)
#>  vctrs          0.4.1          2022-04-13 [1] CRAN (R 4.2.0)
#>  vroom          1.5.7          2021-11-30 [1] CRAN (R 4.2.0)
#>  withr          2.5.0          2022-03-03 [1] CRAN (R 4.2.0)
#>  workflows    * 1.0.0          2022-07-05 [1] CRAN (R 4.2.0)
#>  workflowsets * 1.0.0          2022-07-12 [1] RSPM (R 4.2.0)
#>  xfun           0.31           2022-05-10 [1] CRAN (R 4.2.0)
#>  yaml           2.3.5          2022-02-21 [1] CRAN (R 4.2.0)
#>  yardstick    * 1.0.0          2022-06-06 [1] CRAN (R 4.2.0)
#>
#>  [1] C:/R/win-library/4.2
#>  [2] C:/Program Files/R/R-4.2.1/library
#>
#> ───────────────────────────────────────────────────────────────────────────────
```
