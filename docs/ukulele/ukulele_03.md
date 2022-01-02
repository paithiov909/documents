# 「スイミング・スクール」を読む（A4横置き・縦書き40字30行PDF）

## モチベーション

A4縦書き40字30行の印刷はMS Wordの有料ライセンスがないとふつうは詰む。代替策としてCloud LaTeXでPDF出力した例を紹介する。

https://github.com/lyrikuso/bouquet-article

## その他の代替ソフト

### MS Office Word（無料プラン）

縦書きできない。

### Google ドキュメント

縦書きできない。

### LibreOffice（Writer）

縦書きできるが、字数・行ともに既定では36が最大値であるため、行あたり40字に設定できない。

### TATEditor

自力で設定を調整すればできる。ただ、基本的にただのテキストエディタなので、構造化された文章を書くには非力に感じられる。

## 実際の使用環境

- [Typora](https://typora.io/)
- [Cloud LaTeX](https://cloudlatex.io/ja)（＋VS Code拡張）

文章の下書きはTyporaを使ってマークダウンで書くことが多い。1行40字の縦書きに組む場合、まずTyporaの組み込みの機能でTeXにエクスポートする（要Pandoc）。これをCloud LaTeX（uplatex）に持ちこみ、プリアンブルを書き換えたり適宜修正したりしてから、PDFに出力する。

以下はCloud LaTeXの縦書きテンプレートをもとにして適当に書き換えたプリアンブルの例。実際にはかなり余白が出るが、ちゃんと文字サイズを計算して割り付けるのは苦行なのでやらないほうがいい。

```tex
% A4 段組なし 用紙は横置きにする
\documentclass[uplatex,a4paper,oneside,landscape]{jsarticle}

% フッターにノンブル
\pagestyle{plain}

% 見出しレベル
\setcounter{secnumdepth}{0}

\usepackage[sourcehan-otc]{pxchfon}
\usepackage{lltjp-geometry}

\makeatletter
% 縦組にする
\AtBeginDocument{\tate\adjustbaseline}

% \textheight と \textwidth を逆転
\setlength{\@tempdima}{\textheight}
\setlength{\textheight}{\textwidth}
\setlength{\textwidth}{\@tempdima}

% ヘッダの左右を逆転
\let\@temp\@evenhead
\let\@evenhead\@oddhead
\let\@oddhead\@temp
\fullwidth\textheight

% \maketitle の出力様式を微修正
\def\@maketitle{%
  \newpage\null
  \vskip 2em
  \begin{center}%
    {\huge \gtfamily \@title \par}%
    \vskip 1.5em
    {\large
      \lineskip .5em
        \@author
      \par}%
  \end{center}%
  \par\vskip 1.5em
}

\let\plainifnotempty\relax

% 40字30行 geometryに任せるので30行にならないことがある
\usepackage[truedimen,textwidth=40zw,lines=30,centering]{geometry}

\makeatother

\title{}
\author{}
\date{}

```

## PDF

この設定だと次の点は上手くいっていない。

- 一部の全角記号が期待どおり倒立しない（「‐」（U+FF0D、全角ハイフン）「≒」（U+2252、ほとんど等しい）など）
- URL文字列が含まれる場合、割り付けが均等割り付けになり字間が開いてしまう（これは適当なパッケージを利用すれば比較的簡単に解決できる）

