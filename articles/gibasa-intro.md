---
title: "RMeCabみたいなRパッケージをCRANに投稿した話"
emoji: '🎉'
type: 'idea'
topics: ['r']
published: true
author: 'paithiov909'
---

## gibasaパッケージについて

[RMeCab](https://github.com/IshidaMotohiro/RMeCab)みたいなことができる、gibasaというRパッケージを個人で開発しています。先日CRANに投稿できたので、2023年4月20日現在では、`install.packages("gibasa")`とするだけでもインストールできるようになっています。

https://github.com/paithiov909/gibasa

モチベーションとしては、`tidytext::unnest_tokens`と同じような処理をMeCabを利用しつつできるようにしたいということで開発しています。また、とくに最近は、より簡単に利用をはじめられるようにしようと、すこしずつ改善を続けています。

## 開発の背景

RからMeCabを利用できるRパッケージとしては、すでにRMeCabがあります。徳島大学の石田基広先生が開発されているもので、わりと昔からあるパッケージです。

RMeCabは便利なパッケージですが、残念ながら、CRANには登録されていません。

技術的な話をすると、RMeCabはMeCabのダイナミックライブラリを用いて形態素解析をおこないます。そのため、ビルドするには`libmecab.dll`などのバイナリファイルが必要です。一方で、CRANポリシーは基本的にそうしたバイナリファイルをソースパッケージに含めることを禁止しています。おそらくですが、そういった事情から、RMeCabはCRANに登録することが困難だったのだろうと思われます。

同様にMeCabのダイナミックライブラリを利用するRパッケージとして、[RcppMeCab](https://github.com/junhewk/RcppMeCab)というパッケージがCRANにあります。しかし、RcppMeCabは、2023年4月現在ではすでにアクティブにメンテナンスされていないように見えます。MeCabを利用するパッケージをCRANに登録し、複数のプラットフォーム上で動作するようにメンテンナンスするのはなかなか難しかったようで、実際、そのようなRパッケージはこれまでCRANにはありませんでした。

その点、gibasaはCRANからインストールできるものとしてはおそらくはじめてともいえる、外部のバイナリファイルなしで形態素解析が可能なRパッケージです。gibasaはv0.5.0からMeCabのソースコードをソースパッケージ内に含んでいるため、ビルドするのにMeCabのバイナリファイルを必要としません。

## インストールの仕方

gibasaを実際に利用するには、MeCabの辞書とその場所が書かれた設定ファイル（`mecabrc`）が必要です。逆に、それらがあればMeCabのバイナリはなくても動かせます。よくわからないという人は、とりあえずふつうにMeCabもインストールしておくと動かせると思います。

たとえば、WindowsでMeCabをインストール済みであれば、以下のようにするだけで利用できるようになるはずです。

```r
install.packages("gibasa")
```

インストール後に`gibasa::dictionary_info()`して、辞書のパスが含まれているデータフレームが返ってくれば上手くいっています。


## 使い方

形態素解析をするには、次のように`tokenize`という関数を使います。

```r
gibasa::tokenize(c("頭が赤い魚を食べた猫", "望遠鏡で泳ぐ彼女を見た"))
```

この関数には、次のようにデータフレームを渡すこともできます。

```r
data.frame(
  doc_id = c("a", "b"),
  text = c("頭が赤い魚を食べた猫", "望遠鏡で泳ぐ彼女を見た")
) |>
  gibasa::tokenize()
```

詳しい使い方は、[ここ](https://paithiov909.github.io/gibasa/)や[これ](https://paithiov909.github.io/textmining-ja/)を見てください。
