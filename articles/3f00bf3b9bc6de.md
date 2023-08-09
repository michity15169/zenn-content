---
title: "Cloud Spanner の運用について"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudspanner","spanner","gcp"]
published: false
---
# はじめに
# バックアップ
# ポイントインタイムリカバリ(Point In Time Recovery)
[PITR](https://cloud.google.com/spanner/docs/pitr?hl=ja) とはデータベースの内容を特定の時刻に戻すという機能です。
アプリケーションの誤動作やオペレーションミスによるデータ破損など、論理障害への対応で利用されます。

一般的な RDBMS ではバックアップとトランザクションログ(binlog や WAL、アーカイブログ)と組み合わせて、稼働中のデータベースとは**別に**リストア先のインスタンスを作られることが多いかと思います。これに対して Cloud Spanner の PITR の考え方は少し違います。

Cloud Spanner では、過去特定の時点でのデータをクエリーを行う事ができる機能(ステイル読み取り)を使って過去の時刻を指定した読み取りを行いその内容をテーブルに書き戻すことで特定の時刻の内容を復元します。そのため、特定のテーブルやレコードのみ復元するといったことも可能です。

ステイル読み取りで参照が可能な過去の時刻は、デフォルトでは現在時刻よりも1時間前です。設定により最大7日間まで延長することが可能です。

# 断片化(fragment)対策
多くの RDBMS ではセカンダリインデックスについて作成後にレコードに対して追記・削除・更新を繰り返すとデータの物理レイアウトが断片化を起こしパフォーマンス低下を招きます。この対策として、定期的なインデックスのメンテナンスを行う必要があります。

Cloud Spanner ではこのような明示的な操作は不要です。

Cloud Spanner では [LSM-Tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) と呼ばれるデータ構造を使っています。このデータ構造では追記はもちろん、削除・更新もその場(in-place)での書き換えではなく追記として実施されます。そのためデータファイルが歯抜けになり断片化するという動作が原理的に発生しません。

断片化に近い事象としては、大量のデータを一気にロードしたり削除したときが考えられます。先程の説明の通りこれらの操作は追記によって実現されます。もちろん削除操作を追記しただけではデータの占有サイズが大きくなる一方であるため、バックグラウンドタスクとしてコンパクション(Compaction)が実施されます。これによりデータが再編成されます。コンパクションの操作はシステムタスクとして実装されており、[中間の優先度で実行されています](https://cloud.google.com/spanner/docs/cpu-utilization?hl=ja#task-priority)。そのため、大量のデータ操作をしてからある程度の時間をおいてシステムタスクが CPU 時間を使用する様子が観測されることがあります。ある意味定常的に詰め直しが発生しているとも言えるでしょう。この仕組みによってデータベース管理の一環として管理者による定期的なインデックスの再編成は必要ありません。

# バルクロード

# 実行計画

# サイジング
# モニタリング
## CPU 使用率
## レイテンシ
## show processlist
## Slow query log
## キャッシュヒット率
## ログ
## コネクション数

