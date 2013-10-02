---
layout: post
title: "scalikejdbc asyncを試してみた"
description: ""
category: 
tags: []
---
{% include JB/setup %}

scalikejdbc-asyncでノンブロッキングなMySQLへのアクセスができると聞いて試してみた。
PostgreSQLだとplay-sampleがうまく動くんだけどMySQLのときはinsertがうまく動かない。
作者の瀬良さんに相談してみると、auto の代わりに localTxを使ったほうがいいとのことだったので
試してみたらそれでいけた。

こんなかんじに書いていたのを
```scala
def create(createdAt: DateTime = DateTime.now)(
implicit session: AsyncDBSession = AsyncDB.sharedSession, cxt: EC = ECGlobal): Future[Log] = {
  for {
    id <- withSQL {
      insert.into(Log).namedValues(column.createdAt -> createdAt)
    }.updateAndReturnGeneratedKey.future()
  } yield Log(id = id, createdAt = createdAt)
}
```


こういうふうに変更した。
```scala
def create(createdAt: DateTime = DateTime.now) = AsyncDB.localTx { implicit s =>
  for {
    id <- withSQL {
      insert.into(Log).namedValues(column.createdAt -> createdAt)
    }.updateAndReturnGeneratedKey.future()
  } yield Log(id = id, createdAt = createdAt)
}
```
試してみたコードはここにアップしといた。

https://github.com/wshino/scalikejdbc-async-mysql


同じ条件でscalikejdbcを使ったplay2を動かしてみて、abをかけてどの程度さばける多さが違うのか試してみた。
abの結果を10回取得してみて平均値を算出してみた。
↓

    /usr/local/apache/bin/ab -n 1000 -c 50 http://172.19.8.122:9000/

結果としてscalikejdbcの場合は  
* Requests per second:  554.473
scalikejdbc-asyncの場合は
* Requests per second:  919.9


おおむね1.6倍ほど高速になったので満足。
プロダクトに使うかどうかは未定だけどわりといいパフォーマンスは出てるんではなかろうか。


