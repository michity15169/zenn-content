---
title: "Cloud Spanner の運用について"
emoji: "🔧"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["cloudspanner", "gcp"]
publication_name: "google_cloud_jp"
published: false
---
## はじめに
## バックアップ
Cloud Spanner のストレージとして [Colossus](https://cloud.google.com/blog/ja/products/storage-data-transfer/a-peek-behind-colossus-googles-file-system) という分散ストレージシステムを利用を利用しているため、物理障害に対して高い耐久性があります。一方で、論理障害などに備えるためのバックアップも可能です。

バックアップはバックアップ元のインスタンスと同じインスタンスに取られ、新しい Spanner インスタンスへ復元できます。バックアップはインスタンスのサーバーリソースを使わずに取得するため、更新やクエリーのパフォーマンスへの影響なく取得が可能です。そのため、一般的な RDBMS の物理バックアップに近い考え方です。


注意点として、バックアップは最大1年の有効期間が有るという点です。これを超えて保存したい場合には次の項目のインポート・エクスポートを使う事をご検討ください。

## インポート・エクスポート



## ポイントインタイムリカバリ(Point In Time Recovery)
[PITR](https://cloud.google.com/spanner/docs/pitr?hl=ja) とはデータベースの内容を特定の時刻に戻すという機能です。
アプリケーションの誤動作やオペレーションミスによるデータ破損など、論理障害への対応で利用されます。

一般的な RDBMS ではバックアップとトランザクションログ(binlog や WAL、アーカイブログ)と組み合わせて、稼働中のデータベースとは**別に**リストア先のインスタンスを作られることが多いかと思います。これに対して Cloud Spanner の PITR の考え方は少し違います。

Cloud Spanner では、過去特定の時点でのデータをクエリーを行う事ができる機能(ステイル読み取り)を使って過去の時刻を指定した読み取りを行いその内容をテーブルに書き戻すことで特定の時刻の内容を復元します。そのため、特定のテーブルやレコードのみ復元するといったことも可能です。

ステイル読み取りで参照が可能な過去の時刻は、デフォルトでは現在時刻よりも1時間前です。設定により最大7日間まで延長することが可能です。
過去のデータの参照可能期間を延長すると更新量によってストレージの使用量が増加いたしますので、その点についてご注意ください。

## 断片化(fragment)対策
多くの RDBMS ではセカンダリインデックスについて作成後にレコードに対して追記・削除・更新を繰り返すとデータの物理レイアウトが断片化を起こしパフォーマンス低下を招きます。この対策として、定期的なインデックスのメンテナンスを行う必要があります。

Cloud Spanner ではこのような明示的な操作は不要です。

Cloud Spanner では [LSM tree](https://en.wikipedia.org/wiki/Log-structured_merge-tree) と呼ばれるデータ構造を使っています。このデータ構造では追記はもちろん、削除・更新もその場(in-place)での書き換えではなく追記として実施されます。そのためデータファイルが歯抜けになり断片化するという動作が原理的に発生しません。

断片化に近い事象としては、大量のデータを一気にロードしたり削除したときが考えられます。先程の説明の通りこれらの操作は追記によって実現されます。もちろん削除操作を追記しただけではデータの占有サイズが大きくなる一方であるため、バックグラウンドタスクとしてコンパクション(Compaction)が実施されます。これによりデータが再編成されます。コンパクションの操作はシステムタスクとして実装されており、[中間の優先度で実行されています](https://cloud.google.com/spanner/docs/cpu-utilization?hl=ja#task-priority)。そのため、大量のデータ操作をしてからある程度の時間をおいてシステムタスクが CPU 時間を使用する様子が観測されることがあります。ある意味定常的に詰め直しが発生しているとも言えるでしょう。この仕組みによってデータベース管理の一環として管理者による定期的なインデックスの再編成は必要ありません。

## バルクロード

## 実行計画
## サイジング
Cloud Spanner ではノード数の増減によって性能をスケールアウト・インすることができます。
1ノードは1000 PU(Processing Units/処理ユニット)とも呼ばれ、[1ノードの性能の目安が提供されています](https://cloud.google.com/spanner/docs/performance?hl=ja#typical-workloads)。

2024年6月時点ではリージョナル構成で、1ノードあたり 22,500 QPS の処理能力があります。これはスキーマなどがベストプラクティスに沿った設計の場合での数値であるため、実際のアプリケーションではこれよりも小さい値で CPU 使用率などが増設を推奨する基準に到達したりレイテンシが悪化する可能性もあります。実際のワークロードを実行したときの性能がノード数に対して適切であるかの比較対象として参照されると良いです。

ノード数の追加はダウンタイムなく実行できますので、負荷に応じて増減されるのが良いでしょう。削除についてもダウンタイムなく可能ですが、大幅にノード数が変わる場合、例えば50ノードから10ノードへ削減されるときはその時点のノード数の10%ずつ10分以上間を置いて削減することをお勧めします。これによりアプリケーションへの性能インパクトを最小化することができます。

# モニタリング
## CPU 使用率

Cloud Spanner の CPU 使用率はノード全体の平均 [CPU 使用率](https://cloud.google.com/spanner/docs/cpu-utilization?hl=ja)がメトリクスとして提供されています。
CPU使用率にはユーザー・システムの区分と優先度について高・中・低の3つの区分で集計されます。そのため全体としては6種類あります。通常はユーザーの優先度高がアプリケーションからのクエリや更新処理の実行用として使用されるため、負荷の大半はこちらになるでしょう。

[推奨される最大の CPU 使用率](https://cloud.google.com/spanner/docs/cpu-utilization?hl=ja#recommended-max)は一般的な RDBMS よりも比較的低いことにご注意ください。例えば、リージョナル構成では高い優先度の合計が 65% 以下を維持することを推奨しています。これを超える場合にはノードを追加する事を検討ください。

## レイテンシ
Cloud Spanner ではリクエスト(クエリやDML)の処理にかかったレイテンシをメトリクスとして提供します。

レイテンシはウェブコンソール上で50パーセンタイル(p50)と99パーセンタイル(p99)が確認可能です。p50 レイテンシはアプリケーションの全般的なレスポンス傾向を判断する指標として重要です。

p99 レイテンシについては、特にテールレイテンシ(tail latency)と呼ぶ場合もあります。テールレイテンシの原因調査については[一本の記事があるほど](https://cloud.google.com/blog/ja/topics/developers-practitioners/how-investigate-high-tail-latency-when-using-cloud-spanner)様々な原因が考えられます。Cloud Spanner は分散システムであるという性質から、構成コンポーネントの一部が交換された場合にそのコンポーネントで処理してた内容を引き継ぐなどを行うことがあり、テールレイテンシの上昇が完全には避けられない場合があります。テールレイテンシの上昇が継続時間で数分程度の過渡的な事象なのか、継続的に発生している事象であるかは原因と対策を判断する上で重要です。継続的に発生している場合にはクエリの処理時間が長くなっている、あるいは CPU などのリソース不足になっている可能性があります。

メトリクスとして得られるレイテンシは Cloud Spanner 自身が報告する指標です。一方でアプリケーション側の観点ではリクエストの送信側での測定であるため、2つの指標が一致しない場合、原因はその間にある可能性があります。クライアントライブラリは Cloud Spanner とのコネクションを使い回す仕組みがありますが、初回の接続時などは比較的大きなレイテンシとなる場合があります。

## show processlist
## Slow query log
## キャッシュヒット率
## ログ
## コネクション数


