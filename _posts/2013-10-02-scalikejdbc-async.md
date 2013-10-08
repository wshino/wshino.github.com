---
layout: post
title: "scalikejdbc asyncを試してみた"
description: ""
category: 
tags: []
---
{% include JB/setup %}

scalikejdbc-asyncでノンブロッキングなMySQLへのアクセスができると聞いて試してみた。
<https://github.com/seratch/scalikejdbc-async>

seratch さんが Activate にインスパイアされて作ったそうだ。

業務でPlay2を使うことがあるので今回の検証には

* Playframework 2.1.3
* MySQL 5.6.13(homebrewでいれたやつ)

を使ってみた。githubにはplayで使う場合のsampleもあって親切。

コネクションプールまわりは初期値とMaxを100にして新しい生成コストが発生しないようにした。

```
db.default.poolInitialSize=100
db.default.maxPoolSize=100
db.default.maxIdleMillis=1000
db.default.maxQueueSize=99
db.default.connectionTimeoutMillis=3000
```

PostgreSQLだとplay-sampleがうまく動くんだけどMySQLのときはinsertがうまく動かない。
作者のseratchさんに相談してみると、auto の代わりに localTxを使ったほうがいいとのことだったので
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

<https://github.com/wshino/scalikejdbc-async-mysql>


同じ条件でscalikejdbcを使ったplay2を動かしてみて、abをかけてどの程度さばける多さが違うのか試してみた。
abの結果を10回取得してみて平均値を算出してみた。以下のようなかんじ。

```
/usr/local/apache/bin/ab -n 1000 -c 50 http://172.19.8.122:9000/
```

結果としてscalikejdbcの場合は

* Requests per second:  554.473

scalikejdbc-asyncの場合は

* Requests per second:  919.9


おおむね1.6倍ほど高速になったので満足。
プロダクトに使うかどうかは未定だけどわりといいパフォーマンスは出てるんではなかろうか。
そのうちtypesafeが作ってるslickもasyncになるんじゃないかって期待してる。

