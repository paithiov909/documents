---
title: 'CRANにRパッケージを投稿するときの資料メモ'
emoji: '🗒'
type: 'idea'
topics: ['r']
published: true
author: 'paithiov909'
---

## このメモについて

CRANに[パッケージ](https://github.com/paithiov909/audubon)を投稿するときに参考にした資料のまとめ。

https://twitter.com/paithiov909/status/1494881031820050432

個人的な感覚としては「CRANに投稿しよう」というのをすごく勧めているわけではない。なかなかめんどくさいし、やっぱりめんどくさいなと思ったらやらないほうがよいと思う。

## 全体の流れ

基本的に、[ThinkR-open/prepare-for-cran](https://github.com/ThinkR-open/prepare-for-cran)で紹介されているTL;DRそのままの流れでやればよさそう。

```r
# Prepare for CRAN ----

# Update dependencies in DESCRIPTION
attachment::att_amend_desc()

# Run tests and examples
devtools::test()
devtools::run_examples()
# autotest::autotest_package(test = TRUE)

# Check package as CRAN
rcmdcheck::rcmdcheck(args = c("--no-manual", "--as-cran"))

# Check content
# remotes::install_github("ThinkR-open/checkhelper")
checkhelper::find_missing_tags()

# Check spelling
# usethis::use_spell_check()
spelling::spell_check_package()

# Check URL are correct
# remotes::install_github("r-lib/urlchecker")
urlchecker::url_check()
urlchecker::url_update()

# check on other distributions
# _rhub
devtools::check_rhub()
rhub::check_on_windows(check_args = "--force-multiarch")
rhub::check_on_solaris()
# _win devel
devtools::check_win_devel()

# Check reverse dependencies
# remotes::install_github("r-lib/revdepcheck")
usethis::use_git_ignore("revdep/")
usethis::use_build_ignore("revdep/")

devtools::revdep()
library(revdepcheck)
# In another session
id <- rstudioapi::terminalExecute("Rscript -e 'revdepcheck::revdep_check(num_workers = 4)'")
rstudioapi::terminalKill(id)
# See outputs
revdep_details(revdep = "pkg")
revdep_summary()                 # table of results by package
revdep_report() # in revdep/
# Clean up when on CRAN
revdep_reset()

# Update NEWS
# Bump version manually and add list of changes

# Add comments for CRAN
usethis::use_cran_comments(open = rlang::is_interactive())

# Upgrade version number
usethis::use_version(which = c("patch", "minor", "major", "dev")[1])

# Verify you're ready for release, and release
devtools::release()
```

`devtools::check_rhub()`については、`results <- devtools::check_rhub()`とかして結果を保持しておいて`results$cran_summary()`とするとcran-comments.mdに書く用のマークダウンを出力できるので、たぶんそうしたほうがよい。

パッケージ製作全体を通じたより詳しい流れとしては、次の記事がオススメ。

- [How to write your own R package and publish it on CRAN | Methods Bites](https://www.mzes.uni-mannheim.de/socialsciencedatalab/article/r-package/)
- [CRAN にパッケージを初投稿する手順 | Atusy's blog](https://blog.atusy.net/2019/06/28/cran-submission/)

[johnmackintosh/CRANt-touch-this](https://github.com/johnmackintosh/CRANt-touch-this)では、次の資料が挙げられている（どれもだいたい似たようなことが書かれている）。

- [PIPING HOT DATA: Getting started with unit testing in R](https://www.pipinghotdata.com/posts/2021-11-23-getting-started-with-unit-testing-in-r/)
- [Chapter 20 Releasing a package | R Packages](https://r-pkgs.org/release.html)
- [Checklist for R package (re-)submissions on CRAN](https://www.marinedatascience.co/blog/2020/01/09/checklist-for-r-package-re-submissions-on-cran/)
- [Preparing Your Package for for Submission](https://johnmuschelli.com/neuroc/getting_ready_for_submission/index.html)

## あんまり書かれてなさそうなこと

### 細々としたポイント

- [DavisVaughan/extrachecks](https://github.com/DavisVaughan/extrachecks)

### ライセンス周り

このへんを読もう。

- [Chapter 9 Licensing | R Packages](https://r-pkgs.org/license.html)
- [Licensing R](https://thinkr-open.github.io/licensing-r/)

前提として、CRANに登録できるパッケージは「[標準ライセンス（“official” licenses）](https://thinkr-open.github.io/licensing-r/rworld.html#classifying-the-11-official-licenses)」でライセンスしておくのが無難。

標準ライセンスはDESCRIPTIONに所定の書き方で指定するだけでよく、逆に`LICENSE`というファイル名でライセンスの全文コピーを含めてはいけないとされている。標準ライセンスの全文コピーをパッケージに含める場合、`LICENSE.md`として保存して`.Rbuildignore`で無視させる。

要点を簡単に指摘すると、まず、CRANポリシーには、著作権保持者は全員DESCRIPTIONなどでわかるように列挙してね、という決まりが書かれている。これはデータセットについても同様で、仮に著作権がない（著作権が消滅していたりといった）データを含める場合であってもドキュメントにその旨を書かないとツッコまれるっぽい。

また、PRなどを通じてコミットされたコードでなく、バンドルされたコードを含める場合、同様にpermissiveなライセンスであっても統合できるライセンスとできないライセンスがある点に注意。たとえば、MITライセンスは修正BSDに統合できるが、修正BSDをMITに統合することはできないらしい。そのへんのライセンスの「強さ」関係については、[このあたり](https://en.wikipedia.org/wiki/License_compatibility#Compatibility_of_FOSS_licenses)などに書かれている。
