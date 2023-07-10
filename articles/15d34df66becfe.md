---
title: "Cloud Spanner における各種トランザクションの使い分け"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudspanner","gcp","spanner"]
published: false
---
# Cloud Spanner における各種トランザクションの使い分け

## 概要
Cloud Spanner にはトランザクションの実行方法が数種類あります。これらの各方式には使うべき場所やメリット・デメリットがあるため、これらの各方式についての差分を学ぶことでより効率的に Cloud Spanner を利用できます。

Cloud Spanner で利用可能なトランザクションは分類すると以下のパターンがあります。

- 読み取り/書き込みトランザクション
- 読み取り専用トランザクション
  -  強い整合性読み取り
  - ステイル読み取り
- パーティション化 DML

特に断りがない場合、単一リージョン(リージョナル)構成での Cloud Spanner の動作を前提としています。
コードサンプルは Ruby と [Ruby Spanner Client](https://github.com/googleapis/ruby-spanner) での動作を確認しています。

## 読み取り / 書き込みトランザクション(Read/Write transaction)

名前の通り、読み込みと書き込みを含む一連の処理を行うときのトランザクションです。一般的な RDBMS でのトランザクションと同じく複数の処理をまとめたアトミック(原子的)な処理が可能です。

トランザクション中で読み取った範囲に対してセル単位での読み取り共有ロックを、書き込みには書き込み共有ロックを取得します。また、単一トランザクションで変更できる内容について大きさに一定の上限(40000 ミューテーション)があります。この上限を超える処理にはパーティション化 DML を使うことが適しています。

例:
```ruby
client.transaction do |transaction|
  player = "100"
  results = transaction.query("SELECT quantity FROM player_items WHERE player_id=@player AND item_id=@item", 
                               params: { player: player, item: item })
  quantity = results.rows.first[:quantity] + 1
  transaction.execute("UPDATE player_items SET quantity=@quantity WHERE player_id=@player AND item_id=@item",
                            params: { player: player, item: item, quantity: quantity })
end
```

## 読み取り専用トランザクション(Read only transaction)
複数の読み取り処理で一貫した読み取り結果が必要な場合に使うトランザクションです。単発の SQL クエリを実行して結果を得たいだけの場合は、単一読み取りメソッドを使うことも可能です。 

読み取り専用トランザクションでは**ロックを必要としません**。つまり読み取り / 書き込みトランザクションも含めて**その他のトランザクションをブロックしません**。

読み取り専用トランザクションと単一読み取りメソッドは実行時にタイムスタンプバウンド(タイムスタンプの境界)を指定できます。この指定方法により強い整合性読み取りかステイル読み取りのいずれかになります。単一読み取りメソッドにもタイムバウンドを指定することが可能で、効果はトランザクションのときと同様です。

### 強い整合性読み取り(Strong read)
タイムスタンプバウンドを指定しないあるいは0と指定した場合、強い整合性のある読み取りとなりデータベースに対して行われた最新の更新を反映した内容が読み取られます。

強い整合性読み取りはトランザクション開始次点の、更新直後に実行されても更新された内容を含んだ結果が取得可能です。読み取り内容の反映に**遅延が許されない場合**にご利用ください。
### ステイル読み取り(Stale read)
タイムスタンプバウンドに正の数を指定した場合、ステイル読み取りとなります。英語の Stale とは「新鮮ではない」という意味です。


タイムスタンプバウンドの指定には2種類あり、現在時刻から正確に過去の秒数を指定する方法(**ExactStaleness**)と現在時刻から指定した秒数までの過去のうち最新の結果を取得する方法(**MaxStaleness**)があります。

![](/images/staleness.png)

例:

```ruby
client.snapshot(exact_staleness: 10) do |snap|
  results = snap.query("SELECT quantity FROM player_items WHERE player_id=@player AND item_id=@item", 
                       params: { player: player, item: item })
  puts results.rows.first[:quantity] 
end
```
この実行例では現在時刻よりきっかり 10秒前のスナップショットをとり、それに対して後続のブロックで読み取りを行うという処理です。複数のクエリを実行しても同一のスナップショットに対して実行されるため、同じデータの断面へのクエリ結果が得られます。

ステイル読み取りの使いどころとしては MySQL や PostgreSQL のリードレプリカを使っている場合、リードレプリカで処理可能な読み取り処理を代替できるとイメージいただくと良いと思います。MySQL や PostgreSQL のリードレプリカはその実現方法から、レプリカソースで実行された更新内容が遅延して反映されるためリードレプリカ上での内容も遅延します。リードレプリは遅延が許容される処理、例えば EC サイトの商品一覧ページなどのようにアクセスが多いが遅延が多少は許容される場面で利用されているかと思います。このような処理をステイル読み取りとして読み替えると効果的です。

ステイル読み取りは読み取り専用トランザクションの一種であるためロックを取りませんし、後述の仕組みによりパフォーマンス上もメリットがあります。
## パーティション化 DML(Partitioned DML)

パーティション化 DML とはデータ更新の DML (UPDATE, DELETE など)を一括して実行するための仕組みです。DML の内容がべき等であることなどが必要ですが、RW トランザクションでの一括更新できない規模の大きなテーブルの全体に対しても実行可能です。

```ruby
row_count = db.execute_partition_update \
  "UPDATE users SET friends = NULL WHERE active = false",
  request_options: request_options
```


簡単に言うと対象のテーブルを細かい範囲で分割し、その領域ごとに DML を実行するという仕組みです。そのため、処理はアトミックではないことにご留意ください。そのため、厳密にはトランザクションではありません。
## ステイル読み取りはどうして効率的に実行できるのか
Cloud Spanner はデータをスプリットという単位で管理します。各スプリットは3つのゾーンに複製されます。3つの複製のうち、いずれかの一つがリーダー(Leader)として選ばれます。残りの2つはレプリカとなります。

強い整合性読み取りを行った場合、読み取りのリクエストは通常は同じゾーンにある最寄りのレプリカが処理を担当します。レプリカは読み取り対象の行が最新であるかをリーダーに確認を行います。レプリカが最新の内容を持っていることが確認できればレプリカ自身の持っている情報で処理が行われ、クライアントへ結果が返されます。レプリカの持っている内容が最新ではない場合、最新への更新を待って結果が返されます。

一方で、ステイル読み取りの場合は最新から一定の時間過去のデータであることを許容します。そのため、レプリカ上のデータで処理が完結することが可能となる場合があります。これにより*ステイル読み取りは強い整合性読み取りと比べてより短い時間で応答される*ことが期待されます。

では、タイムスタンプバウンドとしてどれぐらいを指定すると読み取りレイテンシの短縮に効果があるでしょうか。リーダーは10秒間隔で最新のタイムスタンプで更新するため、これよりも長い時間を指定することは安定した効果が期待できます。余裕をもって15秒のタイムスタンプバウンドを指定すると効果が大きいでしょう。

読み取りレイテンシの短縮効果は単一リージョン構成でも実感できますが、マルチリージョン構成ではその効果はより大きくなります。これはリーダーとレプリカが地理的に離れた場所に在ることが多く、WAN 越しでのリーダーへの通信を必要とせずレプリカのみで処理が完結することの恩恵が大きいためです。Cloud Spanner を**マルチリージョン構成を利用される場合は特にステイル読み取りを活用**されることをお勧め致します。

参考資料
- https://cloud.google.com/blog/topics/developers-practitioners/what-cloud-spanner?hl=en
- https://cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes?hl=ja
- https://cloud.google.com/blog/topics/developers-practitioners/demystifying-cloud-spanner-multi-region-configurations?hl=en


## どのトランザクションを使えば良いかのフローチャート
```mermaid
graph LR
	A["読み取り処理のみですか？"] -->|Yes| B["複数の読み取りで一貫した内容が必要ですか？"]
	A -->|No|RW["更新内容が大規模ですか？"]
	
 B -->|Yes|J["読み取り内容が最新の情報であることを必要としますか？"]
 J -->|Yes|D["強い整合性読み取りトランザクション"]
 J -->|No|E["ステイル読み取りトランザクション"]
 
 B --> |No|I["単一読み取りメソッドが利用可能です。
 読み取り内容は最新である必要があります？"]
 I --> |Yes|SingleStrong["強い整合性読み取り"]
 I --> |No|SingleStale["ステイル読み取り"]
 
 RW -->|Yes|PDML["パーティション化 DML で適用できないか検討"]
 RW -->|No|RWTX["読み取り/書き込みトランザクション"]
```
