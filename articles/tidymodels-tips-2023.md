---
title: "tidymodelsを使うとき知っておくとよさそうなこと"
emoji: "📒"
type: "tech"
topics: ["r"]
published: true
--

## はじめに

[R言語 Advent Calendar 2023](https://qiita.com/advent-calendar/2023/rlang)の6日目の記事、その2です。最近、[mlpack](https://www.mlpack.org/doc/stable/r_documentation.html)の関数をtidymodelsの枠組みのなかで使えるようにするためのラッパーを書いています。

https://github.com/paithiov909/baritsu

tidymodelsは私はいまだに全容がよくわからないですが、わからないなりにその気持ちがちょっとわかってきたような気もします。この記事では、「tidymodelsのコードって、なんかこんな感じで書くとよさそうだよね」というtipsのようなものを3点だけ紹介します。


## parallel::detectCores()はやめよう

とりあえず布教したいこととして、できれば`parallel::detectCores()`は`parallelly::availableCores()`に置き換えましょうという話をします。これはtidymodelsとは関係ない話ですが、何はともあれ、tidymodelsを読み込んでデータを用意します。


```r
suppressPackageStartupMessages({
  require(tidymodels)
})
ames_split <- initial_split(modeldata::ames, strata = Sale_Price)
ames_train <- training(ames_split)
ames_test <- testing(ames_split)
```

ここで、たとえば、次のようなコードを書いたとしましょう。

```r
num_trees <- 500L
ames_spec <-
  rand_forest(
    mtry = ncol(ames_train) - 1, # 目的変数を除いた特徴量の数
    trees = num_trees
  ) |>
  set_mode("regression") |>
  set_engine("ranger") |>
  set_args(
    num.threads = parallel::detectCores() - 1, # 全コア使うのは嫌なので、とりあえず1つ減らしている
    seed = 1234
  )
```

このコードは間違いではありません。多くのケースでは、問題なく動きます。しかし、このコードは稀に動かない環境があることが知られており、コードの可搬性を考えるならできれば避けたい書き方です。

というのも、`parallel::detectCores()`は、呼ばれた環境によっては`NA`を返すことがあります。そのため、ここでは`num.threads`に`NA - 1`が渡されてしまい、いざrangerの学習をはじめたときに謎のエラーが返ることになります。たちが悪いポイントが、とくにこの文脈では、実際にrangerの学習をはじめてみるまで、エラーが出るかは確認できないという点です。あと、当たり前ですが、利用できるコアが1つしかない環境でも`1 - 1`が渡されてしまって、謎のエラーが返ったりします。

こうした罠を知っている人は、`parallel::detectCores() - 1`を`max(1, parallel::detectCores() - 1, na.rm = TRUE)`というふうに書きます。長いですね。私自身、このスニペットはよく使いますが、やはり長いと思います。そこで、`parallelly::availableCores()`の出番です。

`parallely::availableCores()`を使うと、同等のコードが以下のように書けます。

```r
num_trees <- 500L
ames_spec <-
  rand_forest(
    mtry = ncol(ames_train) - 1,
    trees = num_trees
  ) |>
  set_mode("regression") |>
  set_engine("ranger") |>
  set_args(
    num.threads = parallelly::availableCores(omit = 1),
    seed = 1234
  )
```

便利ですね。というわけですので、`parallel::detectCores()`はできるだけ`parallelly::availableCores()`に置き換えていきましょう。

- [Enhancing the parallel Package • parallelly](https://parallelly.futureverse.org/)


## model_specに変数を与えるときはインジェクションしたほうがいいよ

それでは、先ほどのコードは晴れて問題なくなったのかというと、まだ微妙なところがあります。

というのは、先ほどのコードは`ames_train`や`num_trees`といった、手もとにある変数の中身に依存しているからです。実際、こういう書き方は次のようなケースで機能しなくなるのでやめたほうがいいよと、[parsnipのドキュメント](https://parsnip.tidymodels.org/reference/model_spec.html#:~:text=The%20consequence%20of,do%20not%20exist.)のなかでも案内されています。

> The consequence of this strategy is that any data required to get the parameter values must be available when the model is fit. The two main ways that this can fail is if:
>
> 1. The data have been modified between the creation of the model specification and when the model fit function is invoked.
> 2. If the model specification is saved and loaded into a new session where those same data objects do not exist.

じゃあどうすればいいのかというと、次のように書きます。


```r
num_trees <- 500L
ames_spec <-
  rand_forest(
    mtry = .cols(), # こういうときはdescriptorを使う
    trees = !!num_trees # これはインジェクションしたほうがいい
  ) |>
  set_mode("regression") |>
  set_engine("ranger") |>
  set_args(
    num.threads = !!parallelly::availableCores(omit = 1), # こっちをインジェクションすべきかはケースによる
    seed = 1234
  )
```

`.cols()`は[descriptors](https://parsnip.tidymodels.org/reference/descriptors.html)と呼ばれているparsnipの関数で、fitしたときに与えられているデータを見て値が決まるプレースホルダーのようなものです。また、その他の手もとにある変数（Rセッションをまたいだときに中身が変わる可能性があるオブジェクト）については、`!!`を使うことでインジェクションしてしまうのが安全です。

このあたりの考え方は、学習したモデルをどこかにデプロイしたかったりとか、何かのかたちで再利用したいときにとくに重要になります。複数のRセッション（あるいは実行環境）をまたいでも同様に動作するモデルをつくるには、descriptorsや`!!`を使うんだよねという感じで覚えておきましょう。


## add_model()にはformulaを渡すことができる

それでは、適当にレシピをつくってモデルを学習してみましょう……となったときに、ここではちょっと凝った（？）モデリングをしたいとします。

たとえば、こういう雰囲気のやつです。


```r
try({
  ames_rec <-
    recipe(log1p(Sale_Price) ~ ., data = ames_train) |>
    step_nzv(all_predictors()) |>
    step_select(
      all_outcomes(),
      all_numeric_predictors(),
      !starts_with("Year"), !ends_with("tude")
    ) |>
    step_normalize(all_numeric_predictors())
})
#> Error in inline_check(formula) :
#>   ✖ No in-line functions should be used here.
#> ℹ The following function was found: `log1p`.
#> ℹ Use steps to do transformations instead.
#> ℹ If your modeling engine uses special terms in formulas, pass that formula to
#>   workflows as a model formula.
```

しかし、このコードは動きません。エラーメッセージの下のインフォメーションを読むと、おそらくこうしろと言っています。

```r
ames_rec <-
  recipe(Sale_Price ~ ., data = ames_train) |>
  step_mutate(Sale_Price = log1p(Sale_Price)) |>
  step_nzv(all_predictors()) |>
  step_select(
    all_outcomes(),
    all_numeric_predictors(),
    !starts_with("Year"), !ends_with("tude")
  ) |>
  step_normalize(all_numeric_predictors())
```

しかし、じゃあ私たちはこれがしたかったのかというと、たぶんそういうことではないだろうと思います。こういうことがしたい人というのは、おそらく次のようなことがしたいのでしょう。


```r
ames_rec <-
  recipe(Sale_Price ~ ., data = ames_train) |>
  step_nzv(all_predictors()) |>
  step_select(
    all_outcomes(),
    all_numeric_predictors(),
    !starts_with("Year"), !ends_with("tude")
  ) |>
  step_normalize(all_numeric_predictors())

base_wf <-
  workflow() |>
  add_recipe(ames_rec) |>
  add_model(ames_spec, formula = log1p(Sale_Price) ~ .) |>
  fit(ames_train)
```

実は、これはちゃんと動きます。

この書き方はtidymodels本にも出てきていなかった気がするので、あまり知られていないかもしれませんが、実は`workflows::add_model()`にはこのように`formula`引数を渡すことができるのです。いやいや、formulaならもうrecipeに渡したはずじゃないと思ったかもしれません。ちがいます。というか、あれはたしかにformulaだったのですが、私たちがよく使っていたformulaとは、実のところ別のものです。

tidymodelsの枠組みの中では、formulaには2つの種類があります。ひとつはrecipeに渡されるやつで、`preprocessing formula`と呼ばれるもの。もうひとつはfitに渡されるやつで、`model formula`と呼ばれるものです。

前者は、前処理を経てfitに渡されるべき変数を記述するために用いられ、後者は、実際にfitするモデルを記述するために用いられます。私たちがformulaと聞いて思い浮かべるだろうやつは、おそらく後者の役割を担うものです。にもかかわらずと言いますか、何を隠そう、後者がとくに指定されない場合には、なんと前者が流用されています。

注意点として、当たり前ですが、こうして`model formula`を明示的に指定した場合、モデルの予測値は`log1p`されているため、RMSEとかを計算したいなーというときには次のような感じの書き方になります。


```r
pred <-
  augment(
    extract_fit_parsnip(base_wf),
    new_data = prep(ames_rec) |> bake(new_data = ames_test),
  ) |>
  dplyr::select(Sale_Price, .pred)

pred
#> # A tibble: 733 × 2
#>    Sale_Price .pred
#>         <int> <dbl>
#>  1     215000  12.1
#>  2     105000  11.7
#>  3     244000  12.2
#>  4     189900  12.1
#>  5     236500  12.5
#>  6     180400  12.1
#>  7     171500  12.2
#>  8     149000  11.9
#>  9     105500  11.5
#> 10     306000  12.6
#> # ℹ 723 more rows

pred |>
  dplyr::mutate(.pred = expm1(.pred)) |>
  metrics(truth = Sale_Price, estimate = .pred)
#> # A tibble: 3 × 3
#>   .metric .estimator .estimate
#>   <chr>   <chr>          <dbl>
#> 1 rmse    standard   26654.
#> 2 rsq     standard       0.878
#> 3 mae     standard   16397.
```

これが便利なのかはやりたいことによるでしょう。いちおう、生の予測値に対するRMSEで調整しておいて、得られたハイパーパラメータをつっこむときにformulaの左辺でin-line functionsを使いながらfitするとかもできます。


```r
kielmc <- foreign::read.dta("http://fmwww.bc.edu/ec-p/data/wooldridge/kielmc.dta") |>
  dplyr::as_tibble()

kielmc_spec <-
  linear_reg(penalty = tune()) |>
  set_mode("regression") |>
  set_engine("glmnet")

kielmc_grid <-
  workflow() |>
  add_formula(rprice ~ .) |> # 前処理はしないがpreprocessing formulaは指定しなければいけないとき、`add_formula`を使う
  add_model(
    kielmc_spec,
    formula = rprice ~ nearinc + y81 + nearinc * y81 + age + I(age^2) + log(intst) + log(land) + log(area) + rooms + baths
  ) |>
  tune_grid(
    resamples = vfold_cv(kielmc, strata = rprice, v = 5),
    grid = grid_latin_hypercube(penalty(), size = 5),
    metrics = metric_set(rmse)
  )

best_wf <-
  workflow() |>
  add_formula(rprice ~ .) |>
  add_model(
    kielmc_spec,
    formula = log(rprice) ~ nearinc + y81 + nearinc * y81 + age + I(age^2) + log(intst) + log(land) + log(area) + rooms + baths
  ) |>
  finalize_workflow(select_best(kielmc_grid))

fit(best_wf, kielmc) |>
  tidy()
#> # A tibble: 11 × 3
#>    term          estimate       penalty
#>    <chr>            <dbl>         <dbl>
#>  1 (Intercept)  7.64      0.00000000219
#>  2 nearinc      0.0290    0.00000000219
#>  3 y81          0.160     0.00000000219
#>  4 age         -0.00809   0.00000000219
#>  5 I(age^2)     0.0000360 0.00000000219
#>  6 log(intst)  -0.0586    0.00000000219
#>  7 log(land)    0.0983    0.00000000219
#>  8 log(area)    0.350     0.00000000219
#>  9 rooms        0.0474    0.00000000219
#> 10 baths        0.0965    0.00000000219
#> 11 nearinc:y81 -0.127     0.00000000219
```

このコードは次の資料を参考に書きました。こういうことをしたいときにわざわざtidymodelsを使いたいのかはよくわからないですが、こういう引数があったなと覚えておくと役に立つこともあるかもしれません。

- [chapter: 7 Rによる重回帰分析 | データ分析入門](https://yamamoto-masashi.github.io/DSlec/chapter07.html#log%E3%82%92%E3%81%A8%E3%81%A3%E3%81%9F%E5%A0%B4%E5%90%88)


## まとめ

この記事では次の3つのtipsを紹介しました。

- parallel::detectCores()はやめよう
- model_specに変数を与えるときはインジェクションしたほうがいいよ
- add_model()には実はformulaを渡すことができる

もうすぐtidymodels本が出版されてから1年ほど経つようですが、私はあいかわらず「tidymodelsなんもわからん」という気持ちです。まあ、わからないなりになんとなくでいいと思うので、使っていきましょう。
