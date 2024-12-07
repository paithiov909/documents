---
title: "Rで麻雀の手牌を顔の表情で表す"
emoji: "😊"
type: "idea"
topics: ["r"]
published: true
---

## これは何？

「Rで変な図を描いたらおもしろいのでは？」というネタです。次のような画像をつくりました。

![手牌を表情で表したグラフ](https://storage.googleapis.com/zenn-user-upload/727d20f233d1-20241207.jpg)

あるゲームにおける、各プレイヤーの手牌の状態を顔の表情で表しています。それぞれのパネルが各プレイヤーを表します。縦軸はラウンドのIDで「第n局m本場」のnまたはmが増えると1増えます。横軸はイベントのIDで、その時点で何巡目かみたいなことです。シャンテン数（テンパイで`0`、和了すると`-1`になる数）が減るほど顔の表情の口角が上がり、テンパイすると鼻が付きます。また、引くとシャンテン数が減るような、受け入れ可能な牌の種類が多いほど、眉が釣り上がります。

## モチベーション

最近、麻雀のデータを扱うためのRパッケージをつくりました。

- [paithiov909/shikakusphere: Miscellaneous functions for Japanese mahjong](https://github.com/paithiov909/shikakusphere)
- [paithiov909/convlog: Read 'tenhou.net/6' format files into tibbles](https://github.com/paithiov909/convlog)

次のYouTube動画で紹介しています。動画中でめくっているスライドは[これ](https://paithiov909.github.io/shiryo/tidy-up-mahjong-data/)です。

https://youtu.be/PjNkkrJF_aM

動画のなかで言及しているように、本来は「牌譜を機械学習して当たり牌を予測するぜ！」みたいなことがやりたくて書いた気がするのですが、実際に機械学習までしてみる体力が足りないので、とりあえず何かの図を描いてみることにしました。実用性はさておき、何か変わった図を描いてみたくて、手牌の状態が顔の表情でわかったらネタとしてわかりやすくておもしろいだろうと考え、上の図にたどり着きました。

## やったこと

上で紹介した自作Rパッケージを使いつつ、適当にコードを書きました。まず、牌譜のサンプルデータを読み込んで、整形します。

```r
library(shikakusphere)

json_log <-
  convlog::read_tenhou6(
    system.file("testdata/output_log.example.json", package = "convlog")
  )

paifu <-
  json_log[["paifu"]] |>
  dplyr::filter(
    type %in% c("tsumo", "dahai", "chi", "pon", "daiminkan", "kakan", "ankan", "reach")
  ) |>
  dplyr::mutate(
    # ツモ切りかを判定する場合、あらかじめ`NA`を`FALSE`で埋めておく
    tsumogiri = dplyr::if_else(is.na(tsumogiri), FALSE, tsumogiri),
    # 牌の表現の変換
    pai = trans_tile(pai)
  ) |>
  dplyr::mutate(
    # 副露メンツの変換
    pai = mjai_conv(type, pai, consumed, mjai_target(actor, target)),
    # ツモ切りなら末尾に"_"を付ける
    pai = dplyr::if_else(tsumogiri,
      paste0(pai, "_"),
      pai
    ),
    # リーチ宣言牌なら末尾に"*"を付ける
    pai = dplyr::if_else(
      dplyr::lag(type, default = "") == "reach",
      paste0(pai, "*"),
      pai
    ),
    .by = c(game_id, round_id, actor)
  )
paifu
#> # A tibble: 1,009 × 12
#>    game_id round_id event_id type  actor target pai   tsumogiri consumed
#>      <int>    <int>    <int> <chr> <int>  <int> <chr> <lgl>     <list>
#>  1       1        1        1 tsumo     0     NA s3    FALSE     <NULL>
#>  2       1        1        2 dahai     0     NA z7    FALSE     <NULL>
#>  3       1        1        3 tsumo     1     NA m9    FALSE     <NULL>
#>  4       1        1        4 dahai     1     NA z4    FALSE     <NULL>
#>  5       1        1        5 tsumo     2     NA p5    FALSE     <NULL>
#>  6       1        1        6 dahai     2     NA z1    FALSE     <NULL>
#>  7       1        1        7 tsumo     3     NA z2    FALSE     <NULL>
#>  8       1        1        8 dahai     3     NA z7    FALSE     <NULL>
#>  9       1        1        9 tsumo     0     NA p6    FALSE     <NULL>
#> 10       1        1       10 dahai     0     NA m7    FALSE     <NULL>
#> # ℹ 999 more rows
#> # ℹ 3 more variables: dora_marker <chr>, deltas <list>, ura_markers <list>

qipai <-
  json_log[["round_info"]] |>
  dplyr::rowwise() |>
  dplyr::reframe(
    game_id = game_id,
    round_id = round_id,
    actor = 0:3,
    tehais
  ) |>
  dplyr::mutate(
    qipai = list(trans_tile(as.character(tehais))),
    .by = c(game_id, round_id, actor),
    .keep = "unused"
  )
qipai
#> # A tibble: 40 × 4
#>    game_id round_id actor qipai
#>      <int>    <int> <int> <list>
#>  1       1        1     0 <chr [13]>
#>  2       1        1     1 <chr [13]>
#>  3       1        1     2 <chr [13]>
#>  4       1        1     3 <chr [13]>
#>  5       1        2     0 <chr [13]>
#>  6       1        2     1 <chr [13]>
#>  7       1        2     2 <chr [13]>
#>  8       1        2     3 <chr [13]>
#>  9       1        3     0 <chr [13]>
#> 10       1        3     1 <chr [13]>
#> # ℹ 30 more rows
```

ここまではREADMEに書いているコード例と同じです。次に、すべての打牌イベント後について、その時点での手牌の状態を再現してまとめます。これについては、`shikakusphere::proceed()`がそれなりに重い処理であるため、少し時間がかかります。

```r
# めちゃめちゃwarningが出るが、ここでは無害なので無視してOK
tehais <- paifu |>
  dplyr::left_join(qipai, by = dplyr::join_by(game_id, round_id, actor)) |>
  dplyr::group_by(game_id, round_id, actor) |>
  dplyr::group_modify(~ {
    event_idx <- .x |>
      dplyr::filter(type == "dahai") |>
      dplyr::pull(event_id)
    purrr::map(event_idx, \(id) {
      dplyr::filter(.x, event_id <= id) |>
        dplyr::reframe(
          qipai = qipai,
          zimo = list(pai[which(type %in% c("tsumo", "chi", "pon", "daiminkan"))]),
          dapai = list(pai[which(type %in% c("dahai", "kakan", "ankan"))])
        ) |>
        dplyr::reframe(
          event_id = id,
          last_state = proceed(qipai, zimo, dapai)
        )
    }) |>
      dplyr::bind_rows()
  }) |>
  dplyr::ungroup() |>
  dplyr::distinct()

tehais
#> # A tibble: 495 × 5
#>    game_id round_id actor event_id last_state
#>      <int>    <int> <int>    <int> <paistr>
#>  1       1        1     0        2 <13>'m1134457p67s35z66'
#>  2       1        1     0       10 <13>'m113445p667s35z66'
#>  3       1        1     0       18 <13>'m1133445p66s35z66'
#>  4       1        1     0       26 <13>'m1133445p66s35z66'
#>  5       1        1     0       30 <13>'m1133445p66s3,z666-'
#>  6       1        1     0       38 <13>'m113345p66s34,z666-'
#>  7       1        1     0       46 <13>'m113345p66s34,z666-'
#>  8       1        1     0       54 <13>'m113345p66s34,z666-'
#>  9       1        1     0       62 <13>'m113345p66s34,z666-'
#> 10       1        1     0       70 <13>'m113345p66s34,z666-'
#> # ℹ 485 more rows
```

これをもとに、さらにいい感じに整形して、次のようなデータをつくります。

```r
scores <-
  json_log[["round_info"]] |>
  dplyr::select(game_id, round_id, scores) |>
  dplyr::mutate(
    rank = purrr::map(scores, \(x) 5 - rank(x, ties.method = "min")),
    .keep = "unused"
  ) |>
  tidyr::unnest_longer(rank, values_to = "rank") |>
  dplyr::mutate(actor = 0:3, .by = c(game_id, round_id))

dat <-
  tehais |>
  dplyr::left_join(scores, by = dplyr::join_by(game_id, round_id, actor)) |>
  dplyr::mutate(
    id = dplyr::consecutive_id(game_id, round_id),
    game_id = game_id,
    round_id = round_id,
    event_id = event_id,
    actor = factor(actor + 1),
    rank = rank,
    n_xiangting = calc_xiangting(last_state)[["num"]],
    n_tingpai = collect_tingpai(last_state) |> lengths(),
    is_tenpai = n_xiangting < 1,
    .keep = "none"
  ) |>
  dplyr::mutate(event_id = dplyr::consecutive_id(event_id), .by = c(game_id, round_id, actor))

dat
#> # A tibble: 495 × 9
#>    game_id round_id actor event_id  rank    id n_xiangting n_tingpai is_tenpai
#>      <int>    <int> <fct>    <int> <dbl> <int>       <int>     <int> <lgl>
#>  1       1        1 1            1     4     1           2         5 FALSE
#>  2       1        1 1            2     4     1           2        11 FALSE
#>  3       1        1 1            3     4     1           1         3 FALSE
#>  4       1        1 1            4     4     1           1         3 FALSE
#>  5       1        1 1            5     4     1           1         4 FALSE
#>  6       1        1 1            6     4     1           1         5 FALSE
#>  7       1        1 1            7     4     1           1         5 FALSE
#>  8       1        1 1            8     4     1           1         5 FALSE
#>  9       1        1 1            9     4     1           1         5 FALSE
#> 10       1        1 1           10     4     1           1         5 FALSE
#> # ℹ 485 more rows
```

肝心の図示には、[ggChernoff](https://github.com/Selbosh/ggChernoff)というパッケージを使いました。いわゆる「チャーノフの顔グラフ」にインスパイアされた`ggplot2::geom_point()`の拡張みたいなものらしく、x, yにくわえて、口の上がり具合・眉の角度・目のあいだの距離の3変数と、鼻の有無（logical）とをまとめて図示することができます。

次のようにして描いたグラフをリサイズしたものが、冒頭の画像です。

```r
library(ggplot2)
library(ggChernoff)

dat |>
  dplyr::mutate(
    n_xiangting = 5 - n_xiangting,
    n_tingpai = dplyr::if_else(n_tingpai >= 16, 16, n_tingpai)
  ) |>
  ggplot() +
  aes(
    x = event_id,
    y = id,
    colour = actor,
    smile = n_xiangting,
    brow = n_tingpai,
    nose = is_tenpai
  ) +
  geom_chernoff(size = 8) +
  scale_y_reverse(breaks = c(1, 5, 10)) +
  scale_smile_continuous(name = "n_xiangting", labels = rev(-1:5)) +
  scale_color_manual(values = khroma::color("okabe ito")(4)) +
  facet_wrap(~ actor, ncol = 1) +
  theme_bw() +
  labs(x = element_blank(), y = element_blank())

ggsave("test.png", device = ragg::agg_png(), scale = 1.4, height = 3600,
       limitsize = FALSE, units = "px")
```
