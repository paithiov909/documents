---
ID: 1534fafbbc1d7aef6d6b
Title: 'pipianの紹介: RからCaboChaを呼ぶだけのパッケージ書いた'
Tags: R,自然言語処理,Cabocha
Author: Kato Akiru
Private: false
---

## これは何？

[paithiov909/pipian](https://github.com/paithiov909/pipian)は、日本語係り受け解析器の[CaboCha](https://taku910.github.io/cabocha/)を利用するためのRパッケージです。

内部的には、`cabocha -f3 -n1`コマンドを呼んで出力した一時ファイル（XML）を読み込み、Rcpp側でデータフレームにパースしています。

## モチベーション

- [RCaBoCha](http://rmecab.jp/wiki/index.php?RCaBoCha)や[CabochaR](https://minowalab.org/cabochar/)の（特にWindows環境向けの）代替として
  - Rcppの内部でCaboChaのC/C++ APIを利用しようとすると、本来は、Windows環境では64bit向けにビルドされたCaboChaバイナリが必要になります。しかし、CaboCha本体の64bit対応はかなり難しいため、pipianはプログラム的にCaboChaを利用するのではなく、外部コマンドとして実行します。
  - 外部コマンドとして呼ぶだけなので、Windows環境で64bit Rから32bit CaboChaを実行する場合であっても問題なく動作するというメリットがあります。
- エンコーディングの厳格化
  - pipianは、Windows環境であっても、テキストエンコーディングはすべてUTF-8で扱います。このため、MeCabとCaboChaの辞書もあらかじめUTF-8でコンパイルしたものを用意しておく必要があります。

## 使い方

### インストール

GitHubからインストールするため、Windowsでは[Rtools](https://cran.r-project.org/bin/windows/Rtools/)が必要です。

```r
remotes::install_github("paithiov909/pipian")
```

自分でビルドしない場合、[R-universe](https://paithiov909.r-universe.dev/)からバイナリパッケージをインストールすることもできます。

```r
# Enable universe(s) by paithiov909
options(repos = c(
    paithiov909 = "https://paithiov909.r-universe.dev",
    CRAN = "https://cloud.r-project.org"))

# Install some packages
install.packages("pipian")
```


### 係り受け構造のグラフ化

`pipian::ppn_cabocha`に解析したい文字列を渡して、係り受け解析をおこないます。`pipian::ppn_cabocha`は、渡された文書を一時ファイルに書き出し、テキストファイルに対して解析をおこなった結果のXMLファイルへのパスを返します
（このため、`pipian::ppn_cabocha`を実行するためには、MeCabとCaboChaの実行ファイルにパスが通っている必要があります）。

解析結果のXMLファイルは`pipian::ppn_parse_xml`でデータフレームにパースできます。また、この戻り値のデータフレームは`pipian::ppn_make_graph`で有向グラフ（igraphオブジェクト）にして確認することができます。

```r
sentence <- "ふと振り向くと、たくさんの味方がいてたくさんの優しい人間がいることを、わざわざ自分の誕生日が来ないと気付けない自分を奮い立たせながらも、毎日こんな、湖のようななんの引っ掛かりもない、落ちつき倒し、音一つも感じさせない人間でいれる方に憧れを持てたとある25歳の眩しき朝のことでした"

df <- sentence %>% 
  pipian::ppn_cabocha() %>% 
  pipian::ppn_parse_xml()

head(df)
#>   doc_id sentence_id chunk_id token_id    token chunk_link chunk_score
#> 1      1           1        1        0     ふと          2    1.287564
#> 2      1           1        2        1 振り向く         37   -2.336376
#> 3      1           1        2        2       と         37   -2.336376
#> 4      1           1        2        3       、         37   -2.336376
#> 5      1           1        3        4 たくさん          4    1.927252
#> 6      1           1        3        5       の          4    1.927252
#>   chunk_head chunk_func POS1     POS2 POS3 POS4      X5StageUse1 X5StageUse2
#> 1          1          0 副詞     一般 <NA> <NA>             <NA>        <NA>
#> 2          2          2 動詞     自立 <NA> <NA> 五段・カ行イ音便      基本形
#> 3          2          2 助詞 接続助詞 <NA> <NA>             <NA>        <NA>
#> 4          2          2 記号     読点 <NA> <NA>             <NA>        <NA>
#> 5          5          5 名詞 副詞可能 <NA> <NA>             <NA>        <NA>
#> 6          5          5 助詞   連体化 <NA> <NA>             <NA>        <NA>
#>   Original    Yomi1    Yomi2 entity
#> 1     ふと     フト     フト   <NA>
#> 2 振り向く フリムク フリムク   <NA>
#> 3       と       ト       ト   <NA>
#> 4       、       、       、   <NA>
#> 5 たくさん タクサン タクサン   <NA>
#> 6       の       ノ       ノ   <NA>

g <- df %>% 
  pipian::ppn_make_graph()

print(g)
#> IGRAPH d102f78 DN-- 38 38 -- 
#> + attr: name (v/c), tokens (v/c), pos (v/c), score (e/n)
#> + edges from d102f78 (vertex names):
#>  [1] 111 ->112  112 ->1137 113 ->114  114 ->115  115 ->119  116 ->118 
#>  [7] 117 ->118  118 ->119  119 ->1110 1110->1114 1111->1114 1112->1113
#> [13] 1113->1114 1114->1118 1115->1116 1116->1117 1117->1118 1118->1132
#> [19] 1119->1120 1120->1130 1121->1122 1122->1123 1123->1124 1124->1130
#> [25] 1125->1127 1126->1127 1127->1128 1128->1129 1129->1130 1130->1132
#> [31] 1131->1132 1132->1136 1133->1134 1134->1136 1135->1136 1136->1137
#> [37] 1137->110  110 ->110
```

```r
pagerank <- igraph::page.rank(g, directed = TRUE)

plot(
  g,
  vertex.size = pagerank$vector * 50,
  vertex.color = "steelblue",
  vertex.label = igraph::V(g)$tokens,
  vertex.label.cex = 0.8,
  vertex.label.color = "black",
  edge.width = 0.4,
  edge.arrow.size = 0.4,
  edge.color = "gray80",
  layout = igraph::layout_as_tree(g, mode = "in", flip.y = FALSE)
)
```

![README-deps-1.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/228173/51055aa7-84e4-9aef-773f-99661508ca7b.png)
