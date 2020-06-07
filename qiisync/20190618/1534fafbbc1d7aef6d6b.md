---
ID: 1534fafbbc1d7aef6d6b
Title: RからCaboChaを呼ぶだけのパッケージ書いた
Tags: R,自然言語処理,Cabocha,GoogleColaboratory
Author: Kato Akiru
Private: false
---

## これは何？

RからCaboChaを呼ぶためのRパッケージ。`system()`から`cabocha -f3`コマンドを呼んで出力した一時ファイル（XML）を読みに行っている。Rcpp経由ではないためとくに速くはないが、CaboChaとMeCabのパスが通っていれば使えるはずなので導入は楽。mecabrcは渡せるようにした気がする。

>  [paithiov909/pipian: Tiny Interface to CaboCha](https://github.com/paithiov909/pipian)

これで使えるようになる。

```R
remotes::install_github("paithiov909/pipian")
```

## モチベーション

以前つくったRから形態素解析するオレオレパッケージから機能を切り分けたかった。

## 使い方

### XML出力のパース

```{usage1.R}
res <- pipian::CabochaTbl("ふと振り向くと、たくさんの味方がいてたくさんの優しい人間がいることを、わざわざ自分の誕生日が来ないと気付けない自分を奮い立たせながらも、毎日こんな、湖のようななんの引っ掛かりもない、落ちつき倒し、音一つも感じさせない人間でいれる方に憧れを持てたとある25歳の眩しき朝のことでした")
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

```{usage2.R}
head(pipian::cabochaFlatXML("ふと振り向くと、たくさんの味方がいてたくさんの優しい人間がいることを、わざわざ自分の誕生日が来ないと気付けない自分を奮い立たせながらも、毎日こんな、湖のようななんの引っ掛かりもない、落ちつき倒し、音一つも感じさせない人間でいれる方に憧れを持てたとある25歳の眩しき朝のことでした"))
#>       elem. elemid. attr. value.    level1   level2 level3 level4
#> 1 sentences       1  <NA>   <NA> sentences     <NA>   <NA>   <NA>
#> 2  sentence       2  <NA>   <NA> sentences sentence   <NA>   <NA>
#> 3     chunk       3  <NA>   <NA> sentences sentence  chunk   <NA>
#> 4     chunk       3    id      0 sentences sentence  chunk   <NA>
#> 5     chunk       3  link      1 sentences sentence  chunk   <NA>
#> 6     chunk       3   rel      D sentences sentence  chunk   <NA>
```

### flat XMLの整形

`pipian::cabochaFlatXML(as.tibble = FALSE)`で出力したflat XMLをtibbleに整形できる。このtibbleは[CabochaR](https://minowalab.org/cabochar/)が出力する形式を参考にしたもので、次のカラムからなる。

- sentence_idx: 文番号
- chunk_idx: 文節のインデックス.
- D1: 文節番号
- D2: 係り先の文節の文節番号
- rel:（よくわからない値）
- score: 係り関係のスコア
- head: 主辞の形態素の番号
- func: 機能語の形態素の番号
- tok_idx: 形態素の番号
- ne_value: 固有表現解析の結果の値（`-n 1`オプションを使用している）
- word: 表層形
- POS1~POS4: 品詞, 品詞細分類1, 品詞細分類2, 品詞細分類3
- X5StageUse1: 活用形
- X5StageUse2: 活用型
- Original: 原形
- Yomi1~Yomi2: 読み, 発音

```{usage3.R}
library(magrittr)
res <- pipian::cabochaFlatXML("ふと振り向くと、たくさんの味方がいてたくさんの優しい人間がいることを、わざわざ自分の誕生日が来ないと気付けない自分を奮い立たせながらも、毎日こんな、湖のようななんの引っ掛かりもない、落ちつき倒し、音一つも感じさせない人間でいれる方に憧れを持てたとある25歳の眩しき朝のことでした") %>%
  pipian::CabochaR()

res$morphs[[1]]
#> # A tibble: 78 x 21
#>   chunk_idx tok_idx ne_value word  POS1  POS2  POS3  POS4  X5StageUse1 X5StageUse2
#>       <dbl>  <dbl> <chr>    <chr> <chr> <chr> <chr> <chr> <chr>       <chr>      
#>  1        3      0 O        ふと  副詞  一般  *     *     *           *          
#>  2        5      1 O        振り向く~ 動詞  自立  *     *     五段・カ行イ音便~ 基本形     
#>  3        5      2 O        と    助詞  接続助詞~ *     *     *           *          
#>  4        5      3 O        、    記号  読点  *     *     *           *          
#>  5        9      4 O        たくさん~ 名詞  副詞可能~ *     *     *           *          
#>  6        9      5 O        の    助詞  連体化~ *     *     *           *          
#>  7       12      6 O        味方  名詞  サ変接続~ *     *     *           *          
#>  8       12      7 O        が    助詞  格助詞~ 一般  *     *           *          
#>  9       15      8 O        い    動詞  自立  *     *     一段        連用形     
#> 10       15      9 O        て    助詞  接続助詞~ *     *     *           *          
#> # ... with 68 more rows, and 11 more variables: Original <chr>, Yomi1 <chr>,
#> #   Yomi2 <chr>, sentence_id <int>, chunk_id1 <dbl>, D1 <dbl>, D2 <dbl>, rel <chr>,
#> #   score <dbl>, head <dbl>, func <dbl>

res$as_tibble()
#> # A tibble: 78 x 20
#>    sentence_idx chunk_idx D1    D2    rel   score head  func  tok_idx ne_value
#>           <int>     <dbl> <chr> <chr> <chr> <chr> <chr> <chr>   <dbl> <chr>   
#>  1            1         3 0     1     D     1.28~ 0     0           0 O       
#>  2            1         5 1     36    D     -2.3~ 1     2           1 O       
#>  3            1         5 1     36    D     -2.3~ 1     2           2 O       
#>  4            1         5 1     36    D     -2.3~ 1     2           3 O       
#>  5            1         9 2     3     D     1.92~ 4     5           4 O       
#>  6            1         9 2     3     D     1.92~ 4     5           5 O       
#>  7            1        12 3     4     D     0.83~ 6     7           6 O       
#>  8            1        12 3     4     D     0.83~ 6     7           7 O       
#>  9            1        15 4     8     D     2.02~ 8     9           8 O       
#> 10            1        15 4     8     D     2.02~ 8     9           9 O       
#> # ... with 68 more rows, and 10 more variables: word <chr>, POS1 <chr>,
#> #   POS2 <chr>, POS3 <chr>, POS4 <chr>, X5StageUse1 <chr>, X5StageUse2 <chr>,
#> #   Original <chr>, Yomi1 <chr>, Yomi2 <chr>
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
%%bash
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
!./configure --with-mecab-config=`which mecab-config` --with-charset=UTF8
!make
!make check
!make install
!ldconfig
%cd ../
```

### {pipian}のインストール

rpy2経由でRを使えるようにする。

```
%load_ext rpy2.ipython
```

{pipian}を入れる。

``` r
%%R
remotes::install_github("paithiov909/pipian")
```

### 使用例

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

- [CaboChaで始める係り受け解析 - Qiita](https://qiita.com/nezuq/items/f481f07fc0576b38e81d)

CaboChaが出力するXMLはそのままではパースできないので、適当なルートノードを追記する必要がある。

- [CaboChaによってXMLで出力されたファイルをパースする。 – gepuroの日記](http://d.hatena.ne.jp/gepuro/20111014/1318610472)
