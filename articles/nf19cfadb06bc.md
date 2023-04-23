---
title: "短歌におけるテキストマイニングとその展望"
emoji: "️🐼"
type: "idea"
topics: ["r","nlp"]
published: true
---

## 短歌のテキストマイニング

ずばり「短歌のテキストマイニング」と題した記事を書いた。記事といっても、短歌を分析したRのコードにコメントを添えたみたいなものだけど。

https://github.com/paithiov909/wabbitspunch/blob/master/content/posts/shinabitanori-tanka.Rmd

この記事ではatilika/kuromojiをrJava経由で呼んで自作短歌の形態素解析（IPA辞書）をおこない、出現する語彙の簡単な要約を試みている。具体的には、Webでよく見られる「テキストマイニングをやってみた」系の記事でよくあるような図を描いてみたりしている。

**全作品を通じて頻出する語彙（ストップワードを使用して語彙を削っている。動詞は原型をカウント）**

![&#x753B;&#x50CF;1](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24797219/picture_pc_f417f4b7fd181b48e4ddc529a642decd.png)

**ワードクラウド**

![&#x753B;&#x50CF;2](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24797447/picture_pc_2567ceb682f77daf5c1f17a1a84f6db9.png)

ただ、こういう語彙の頻度に注目した要約は短歌の場合とくに必ずしも重要ではないような気もする。今回はとりあえず動くコードを書きあげることを目標にしていたので自作短歌は436首だけを投入したこともあり、そのなかで相対的に多く出現するものをピックアップしたといっても頻度はたかが知れている。もちろんこういうことができるというおもしろさはあって見た目にもわかりやすいのだが、だからなんだという感は否めない。

やや珍しいだろう分析としては、名詞で助詞の「の」が挟まれている「AのB」のようなかたちの表現を探索して、こういった表現が短歌のどの部分に出現するかを概観しようとした。短歌は57577の定型詩であるため、音数（モーラ数）に注目して要約すると便利だと考え、名詞はモーラ数に置き換えながらこのかたちの表現をマイニングした。

**同じモーラ数の表現ごとに頻度をカウントしたもの**

![&#x753B;&#x50CF;4](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798034/picture_pc_1fd29461b7936be4a76fc3c51bfe3ef9.png)

このカウントが多かったものから順に、3の2から4の4まで、短歌のなかでの実際の位置を図示した。

![&#x753B;&#x50CF;3](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798003/picture_pc_20bac58b5a13effc19228ef72249102b.png)

![&#x753B;&#x50CF;5](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798622/picture_pc_9f01af82dee0b6ed3c9feaef548b5931.png)

![&#x753B;&#x50CF;6](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798633/picture_pc_6dcf67c2083e23c2b3f6f7ce830a1441.png)

![&#x753B;&#x50CF;7](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798644/picture_pc_e3f1b93efd1f28f191db80b5ee53ffb1.png)

![&#x753B;&#x50CF;8](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798656/picture_pc_666739f81ad0f452262655d865287548.png)

![&#x753B;&#x50CF;9](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798714/picture_pc_5a382f656b5f9dd7022536c209bd78de.png)

![&#x753B;&#x50CF;10](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798727/picture_pc_a010556341e9645e9c55c0f81b9030dd.png)

![&#x753B;&#x50CF;11](https://d2l930y2yx77uc.cloudfront.net/production/uploads/images/24798745/picture_pc_fd651ba130a2e89e3c297ea95f09ef41.png)

必ずしも分かりやすい傾向は確認できないかもしれないが、少なくとも4の3や4の2のような4モーラの語が先行する表現は初句のあたりに出現がかたよっているほか、3の3という表現は結句に出現がかたよっているらしいことが確認できるだろう。

## 短歌の計量的分析の先例

笠井康平が𐮷田恭大『光と私語』のテキストマイニングをおこなった結果を公開している。

https://note.com/inunosenakaza/m/m1301c2435627

また、私も「うたの日」という短歌投稿サイトの作品を独自に収集して形態素解析した結果を要約した記事をいくつか書いている。

https://note.com/shinabitanori/n/nb8d2758a5ef7

## 「批評ニューウェーブ」を巡る議論

西巻真が流れをブログ記事にまとめている。

http://cocoatalk.blog94.fc2.com/blog-entry-725.html

私からとくに目新しいコメントはないが、テキストマイニングによる短歌の特徴量化とその要約はあくまでもデータであって、「批評」以前の「情報」でしかないという点には注意したい。先ほど自作短歌のワードクラウドを掲載したが、あれを描いてみたところでだからなんだというのはつまりそういうことだし、私の「データでわかるうたの日」なども情報の要約を試みたものであって、それ自体はとくに批評的な活動を意図したものではない。

## 短歌の構文マイニングの展望

展望として、短歌の構文の特徴量化をやってみたいと考えている。短歌における「構文」の癖はときどきその人の文体的特徴（作家性）のひとつと見なされて話題にのぼることがあるが、そうした短歌における「構文」について定量的に分析した例はおそらくこれまでに存在しない。今回の「AのB」のようなかたちの表現の探索も、頻度をカウントしたいというよりも、むしろ自作の「構文」の癖を確認することができるかを探るための手がかりとしてやってみたものだった。

一般に係り受け解析の結果は構文木として解釈でき、ある文の「構文」は、ある語句から特定の語句を根とするパス（部分木）が存在するかのブール値として特徴量化できると考えられる。また、動詞に係る名詞を含む文節に注目し、動詞の格情報にもとづいて特徴量をつくるのもおもしろそうだと思う。
