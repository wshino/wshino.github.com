---
layout: post
title: "git filter branch"
description: ""
category: 
tags: []
---
{% include JB/setup %}

6年くらいCVSにあったデザイナーさん向けプロジェクトをGithub Enterpriseに移行した。
cloneしてみるとやたら時間がかかる。4GBって書いてある。でも実ファイルは500MBしかない。
よくよくみてみると.git配下が肥大化している。contentというディレクトリが実ファイルのサイズのうちほとんどでしかもバイナリファイル。
まずはここを別レポジトリに切り出す。
切り出したのち、リリースジョブを修正して問題がないことを確認したので元のレポジトリからcontentディレクトリを削除。そしてpush。
.gitを減らす方法をいろいろ調べていたところ、filter-branchコマンドを使ってレポジトリの歴史からファイルを抹消することができるとのこと。
cloneしてきたディレクトリで以下のコマンドを実行

git filter-branch -f --index-filter "git rm -rf --cached --ignore-unmatch content/*" --prune-empty -- --all

ディレクトリサイズを調べてみる

pc-7424% du -sm .git
4183	.git

あまり小さくなっていない、なので git gc する

rm -rf .git/refs/original/

reflogを破棄する

git reflog expire --expire=now --all

git gc してみる

git gc --aggressive --prune=now

あれ、増えた？

pc-7424% du -sm .git
5219	.git

なんか失敗したぽい…

新しいレポジトリ作って6年分の履歴を全部削除しましたとさ。