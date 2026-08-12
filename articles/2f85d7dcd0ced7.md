---
title: "フツーのデータベースとしてSpannerを使うには"
emoji: "⚒️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["spanner","gcp"]
publication_name: "google_cloud_jp"
published: true
---
## この記事の目的

Spannerはスケーラビリティに優れたデータベースであると説明されることの多い製品です。スケーラビリティの面が強調された結果、「Spannerは何か特殊なデータベースではないか」「名前は聞いたことあるけど、普通のアプリケーションでは使えないんでしょ」というイメージを持たれていると感じています。スケーラビリティに特長があるのは事実ですが、データベースとしてみるとテーブル定義とデータ型があり、トランザクションが実行可能で、SQLでクエリーや更新が行える、いわば「普通のリレーショナルデータベース」としての側面もあります。

実際にSpannerを普通のリレーショナルデータベース（以下、RDB）として使うと、MySQLやPostgreSQLとどこがどのように違うのか、どこを意識すればアプリケーションの移植が可能になるかを解説するのがこの記事の目的となります。後半では普通のRDBと比較してSpannerのスケーラビリティ以外のメリットについて解説します。

## Spanner はどれぐらい普通のRDBか

[Spanner](https://en.wikipedia.org/wiki/Spanner_(database))はGoogleが開発した分散データベースです。[2012年に論文](https://research.google/pubs/spanner-googles-globally-distributed-database-2/)として発表され、当初はGoogle社内のサービスで使われていましたが、その後2017年にはクラウドサービスとしても一般に利用可能となりました。当初は専用のAPIでのアクセスのみが提供されていましたが、[2017年の論文](https://research.google/pubs/spanner-becoming-a-sql-system/)でSQLによるアクセス方法も実装したことが発表されました。

Spanner以前にもGoogle社内にはBigtableというキーバリューストア（KVS）の一種であるワイドカラム型のデータベースが存在しました。KVSはスケーラビリティやレイテンシーに優れていますが、アプリケーションを開発するという観点ではそのままでは使いにくい場面もあります。複雑なアプリケーションを実装するにはRDBと併用するか、トランザクションの仕組みを独自に実装する必要が出てきます。Spannerでは開発者にとってより使いやすいデータベースを目指してリレーショナルモデルを採用しました。そのため、RDBで慣れ親しんだテーブル定義があり、カラムにはデータ型が存在し、トランザクションによりアトミックな操作が可能です。

クエリーは`SELECT`で行うことができ、更新は `INSERT`, `DELETE`, `UPDATE` 文が使えます。経緯のところでもご紹介した通り、Spannerの開発当初はAPIによる更新のみが提供されている時期がありました。現在でも[API（ミューテーション）による更新](https://cloud.google.com/spanner/docs/modify-mutation-api?hl=ja)も利用可能です。バルクロードしたいときなどSQLを書くほどではない場面では、今でもミューテーションの方が便利なときもあります。

Spannerがサポートしている現代的なRDBに求められる機能を列挙してみました。

- [SQLによるCRUD](https://cloud.google.com/spanner/docs/reference/standard-sql/overview)
- [サブクエリー](https://cloud.google.com/spanner/docs/reference/standard-sql/subqueries)
- [複数のテーブルへの更新がアトミックに行えるトランザクション](https://cloud.google.com/spanner/docs/transactions?hl=ja#read-write_transactions)
- [データ型](https://cloud.google.com/spanner/docs/reference/standard-sql/data-types)
- [セカンダリインデックス](https://cloud.google.com/spanner/docs/secondary-indexes?hl=ja)
- [外部キー](https://cloud.google.com/spanner/docs/foreign-keys/overview?hl=ja)
- [CHECK制約](https://cloud.google.com/spanner/docs/check-constraint/how-to?hl=ja)
- [生成列](https://cloud.google.com/spanner/docs/generated-column/how-to?hl=ja)
- [シーケンス](https://cloud.google.com/spanner/docs/sequence-tasks?hl=ja)
- [JSON型](https://cloud.google.com/spanner/docs/working-with-json?hl=ja)
- [全文検索](https://cloud.google.com/spanner/docs/full-text-search?hl=ja)

一覧をみて、一般的なアプリケーションから使う機能は一通り揃っているなと実感いただけるのではないかと思います。さらにローカル環境での開発・テスト向けに公式の[Spanner エミュレータ](https://cloud.google.com/spanner/docs/emulator)（Dockerコンテナ）も提供されているため、従来のRDBと同様にローカルで手軽に動作確認やCI/CDを構築することが可能です。

## 接続ライブラリ

接続されないことには話が始まらないので、まずは接続ライブラリから始めます。
Spannerはクライアントアプリケーションとの接続にgRPCを使います。これは既存のRDBとは異なるため、クライアントライブラリは専用のものが開発されています。対応プラットフォームは以下の通りです。

- C++
- C#
- Go
- Java
- Node.js
- PHP
- Python
- Ruby

[クライアントライブラリ](https://cloud.google.com/spanner/docs/reference/libraries)はいわばSpannerへのネイティブの接続方法です。
ライブラリの使い方はさほど特殊ではありませんが、Spanner独自の機能や概念を扱うために一般的なRDBのドライバーと少し違う部分もあります（たとえば、SQL以外での更新方法もサポートしている、トランザクションのリトライを考慮しているなど）。
新規のアプリケーションでSpannerのフル機能を活用したい場合はこちらの使用もおすすめですが、一般的なRDBのドライバーに慣れている場合や既存アプリケーションの移植では後述のドライバーを使う方が良い場合もあります。

クライアントライブラリとは別に[ドライバー](https://cloud.google.com/spanner/docs/drivers-overview)というものも提供されています。
クライアントライブラリがSpannerに接続するための専用のライブラリであるのに対して、ドライバーは各言語向けのデータベースを扱うための汎用的な仕組みに沿うようにクライアントライブラリをラップしたものです。具体的にはたとえばJavaでのJDBCやGoの[database/sql](https://pkg.go.dev/database/sql)がそれにあたります。この仕組みに乗って操作していれば、データベースがMySQLやPostgreSQLでも多くの操作を共通した記述で行うことができます。

ORマッパーもドライバーの一環として提供されています。たとえばJavaの[Hibernate](https://cloud.google.com/spanner/docs/use-hibernate)やRubyの[Active Record](https://cloud.google.com/spanner/docs/use-active-record)、Pythonの[SQLAlchemy](https://cloud.google.com/spanner/docs/use-sqlalchemy)などです。これらを使えば既存のアプリケーションの移植に際して、SQLの書き換えも不要です。

ドライバーは既存アプリケーションを移植するときには便利ですが、Spannerの独自機能などには対応していない場合もあります。各ドライバーとSpannerの機能の対応状況については[対応表](https://cloud.google.com/spanner/docs/drivers-overview#googlesql_drivers_and_orms)に記載されています。

まとめると、まったく新規のアプリケーションの場合はクライアントライブラリを使うと良いですが、既存のアプリケーションを移植したいときや開発者にノウハウがあるフレームワークやドライバーからの移行の場合はSpannerのドライバーが便利です。

### PostgreSQL互換ドライバー

Spannerはアプリケーションとの間の通信にgRPCベースの独自プロトコルを使っていますが、これをPostgreSQLのワイヤープロトコルに変換する[PGAdapter](https://cloud.google.com/spanner/docs/pgadapter)も提供されています。これを使えば、アプリケーションもPostgreSQLへの接続ドライバーをそのまま利用できますので、移植の手間がひとつ省略できます。

PGAdapterはライブラリや独立したソフトウェアのいずれの方法でも利用できます。スタンドアロンでも実行できますし、アプリケーションがコンテナで実行されている場合はPGAdapterをサイドカーとして実行すると良いでしょう。

## 認証

RDBへの接続にはIDとパスワードの組み合わせを利用されることが多いと思います。Spannerの場合は[IAMで認証と認可](https://cloud.google.com/spanner/docs/iam?hl=ja)を行います。

Spannerに限定される話ではありませんが、IAM認証では固定の鍵ファイル（サービスアカウントキー）をコードや環境に配置する方法は漏洩リスクやローテーション問題があるため避けるのが好ましいです。Google Cloud上であればCompute EngineやGKEに[サービスアカウント](https://cloud.google.com/compute/docs/access/create-enable-service-accounts-for-instances?hl=ja)を直接紐付ける形式、実行環境がAWSやAzure等の場合は[Workload Identity連携](https://cloud.google.com/iam/docs/workload-identity-federation?hl=ja)を使うことで、鍵を使わないキーレス認証（Application Default Credentials）を実現できます。

クラウドでの認証方式に慣れていないとIDとパスワードに比べて煩雑に感じられるかもしれませんが、IAMでの認証はクラウドサービスを使う上では一般的な仕組みですし、IAMとデータベースで二重管理にならずローテーションや漏洩のリスクを避ける手段が確立されているというメリットは大きいと思います。

## プライマリキー

ここがRDBとSpannerの一番の違いかもしれません。Spannerには一般的な `AUTO INCREMENT` はありません。代替としてシーケンスと[BIT_REVERSE関数](https://cloud.google.com/spanner/docs/reference/standard-sql/bit_functions#bit_reverse)があります。

じゃあ、`AUTO INCREMENT`を使っているところは機械的にこれに書き換えればいいかと言うと、それはちょっと待ってください。`AUTO INCREMENT`をプライマリキーとして使う場面の多くで必要なのは以下のような条件ではないでしょうか。

- データベース側で自動採番される
- 一意性がある
- 追記順（昇順）

このとき単調増加する、つまり追記順が保証されている性質が必要ないのであれば、UUIDなど衝突の恐れがほぼない一意な文字列を使うという方法もあります。[UUIDの自動生成をデフォルトとする](https://zenn.dev/google_cloud_jp/articles/8bc8338a07f7b5#uuid-%E3%81%AE%E8%87%AA%E5%8B%95%E7%94%9F%E6%88%90)（`DEFAULT (GENERATE_UUID())`）ことにより、データベースで自動的に生成され重複がないという目的は達成することが可能です。

一方、アプリケーション側で追記順（昇順）の値を扱いたい場合にも、`BIT_REVERSE` 関数が有効な解決策となります。`BIT_REVERSE` 関数は数値をビット反転させる関数です。書き込み時にビット反転した値を主キーとして格納することで、データベース上では値をランダムに分散させてホットスポットの発生を防ぎます。そして `BIT_REVERSE` は可逆な操作（再度 `BIT_REVERSE` を実行すると元の数値に戻る）であるため、データを取得したあとにアプリケーション側で再度 `BIT_REVERSE` を行うことで、元の昇順の値を復元して扱うことができます。つまり、「DBのストレージ上では分散させつつ、アプリ上では追記順として扱う」という使い方が可能です。

プライマリキーとして単調増加する数値そのままを使うべきでない理由は、負荷の集中する箇所（ホットスポット）が発生し、それがインスタンス全体の性能のボトルネックになるためです。他の回避策としては、単調増加してほしい範囲が限定できるか検討することです。たとえば、マルチテナントで各テナントの範囲内でのみ昇順・降順の並び替えが欲しいのであれば、`tenant_id` を第1キーとする複合主キーに設計したり、[インターリーブテーブル](https://cloud.google.com/spanner/docs/schema-and-data-model#parent-child_tables)（親テーブルのデータと子テーブルのデータを物理的に近くに配置する構造）を活用する方法が考えられます。また、更新頻度が高くないマスタテーブルなどでは、昇順のキーを使っていても極端に性能のボトルネックとはならないケースもあります。

## SQL

Spannerは2種類のSQL構文をサポートしています。[ANSI準拠のGoogleSQL](https://cloud.google.com/spanner/docs/reference/standard-sql/overview)と[PostgreSQL互換](https://cloud.google.com/spanner/docs/reference/postgresql/overview)です。SQLにはANSI標準があるものの、RDBの実装ごとに拡張が多い言語です。そのため、標準への準拠によって相互運用性が保証されるわけではありませんが、基本的な構文については概ね互換性があります。

MySQLを使っている小さいアプリケーションを数本移植した感触としては、SQLの構文レベルでの互換性があるため多くの場合そのままで動作しましたが、関数名や引数の取り方など表記違いの箇所は書き換えが必要でした。

### INSERT 〜 ON DUPLICATE KEY UPDATE

MySQLではINSERT時にすでに同じプライマリキーのレコードが存在した場合には更新を、なかったときには追記をするという [INSERT 〜 ON DUPLICATE KEY UPDATE 構文があります](https://dev.mysql.com/doc/refman/8.0/ja/insert-on-duplicate.html)。PostgreSQLではINSERTの [ON CONFLICT句](https://www.postgresql.jp/document/15/html/sql-insert.html#SQL-ON-CONFLICT)がそれにあたります。この構文は便利なのでアプリケーションで使われることも多いと思います。

この機能はSpanner（GoogleSQL構文）の[INSERT OR UPDATEという構文](https://zenn.dev/google_cloud_jp/articles/cfffd24b356f71)で実現が可能です（PostgreSQL互換構文の場合は `ON CONFLICT DO UPDATE` も利用できます）。この構文がサポートされる前はトランザクション中で`SELECT`による存在確認と、アプリケーションロジックでの分岐によって`INSERT`と`UPDATE`を出し分ける必要がありました。この部分の書き換えはロジックの変更を伴うため移植のハードルでしたが、現在はこの構文を使うことにより書き換え箇所を大幅に省略できます。

## 対話的操作（REPL）

DBを使うアプリを開発しているとMySQLにおける`mysql`コマンド、PostgreSQLにおける`psql`のようにCLIでSQLを実行し、サッと結果を確認したい時があると思います。Spannerはクラウドサービスとして提供されているため強力なウェブインターフェイスが提供されていますが、[CLIで操作するためのコマンド spanner-cli](https://github.com/cloudspannerecosystem/spanner-cli)も開発されています。

`SELECT`文のクエリーを実行する、DMLで更新する、トランザクションを発行する、ヒストリとライン編集機能など基本的な操作が揃っています。その他には実行計画を表形式で表示する、バッチモードで結果をテキストに書き出すなども可能です。Spanner特有の機能としては優先度の指定、トランザクション・リクエストタグをつけて実行なども可能です。（なお、PGAdapter経由やPostgreSQL互換構文を使用している場合は、おなじみの `psql` コマンドで直接接続することもできます）。

個人的にはロックの挙動を確認するときに、複数のターミナルから横に並べつつ実行してトランザクションを開始してDMLを実行するなどの操作をspanner-cliで行ったりしています。

ちなみに、gcloudコマンドの[spannerサブコマンドからSQLを実行する](https://cloud.google.com/spanner/docs/getting-started/gcloud?hl=ja#query-data)ことも可能です。

## スケーラビリティ以外でのSpannerのメリット

スケーラビリティについては他の記事などでも散々触れられているので、それ以外の話題を中心にSpannerのメリットを解説します。

### 可用性が高い

Spannerはリージョナル構成で99.99%、マルチリージョンとデュアルリージョン構成で99.999%の可用性のSLAが提供されています。後述の通りメンテナンスウィンドウもないため、この可用性には「メンテナンスウィンドウを除く」といった注釈もありません。

### メンテナンスの手間がない

Spannerには内部にコンピュートノードという概念がありますが、ノードへのハードウェアメンテナンスがある場合にもダウンタイムはありません。コンピュートとストレージが分離されており、コンピュートは物理的実体の上で動作していますがそれぞれのコンポーネントは冗長構成となっており、各コンポーネントをダウンタイムなく交換できるためコンピュートのメンテナンスが必要な場合もダウンタイムなしで自動的に実行されます。

これはソフトウェアの更新に関しても同様です。ソフトウェアの更新は順次適用され、更新中にダウンタイムなども生じません。そのため、ソフトウェアアップデートのためのメンテナンスウィンドウの概念も存在しませんし、パッチ適用のためのサービス中断の調整なども必要ありません。

2022年にはSpannerで[ストレージのレイアウトを変更したという記事](https://cloud.google.com/blog/ja/products/databases/spanner-modern-columnar-storage-engine?hl=ja)を公開しました。ストレージのレイアウト変更はRDBではメジャーバージョンアップで行われるような大きな変更ですが、こういった内部的な大規模アップデートもユーザーへのダウンタイムなしで計画・実施されました。

### オンラインでのスキーマ変更（DDL）

運用面で特に大きなメリットとなるのが**完全オンラインでのスキーマ変更（DDL）**です。一般的なRDBでは大規模テーブルに対する `ALTER TABLE`（カラム追加やインデックス生成）でテーブルロックが発生し、クエリ処理や書き込みの遅延・タイムアウトのリスクが生じますが、Spannerではダウンタイムやサービスへの影響なしにバックグラウンドで安全に実行されます。

### 高速で一貫性のある読み取り（Read-Only Transaction）

データの読み取り処理において、普通のRDBではロック競合や読み取り一貫性の制御（分離レベルの考慮）が必要になります。SpannerではTrueTime技術を活用した**Read-Only Transaction（読み取り専用トランザクション）**が提供されており、ロックを一切取得せずに過去の特定のタイムスタンプや最新の一貫したデータを高速に読み取ることが可能です。画面表示やレポート出力などの読み取り処理にRead-Only Transactionを明示的に使うことで、書き込み処理をブロックせず高いスループットを維持できます。

### グラフデータベースとしても使える

2024年8月にSpannerは[グラフデータベース](https://cloud.google.com/spanner/docs/graph/overview)への対応を発表しました。Spannerのグラフデータベース対応はリレーショナルデータベースを拡張する形で提供されます。通常通りテーブルに書き込んだデータに対して、グラフとしてアクセスするときの定義を追加することで、SQLに加えてグラフ用のクエリー言語であるGQL（Graph Query Language/GraphQLとは異なります）でもアクセスできるようになるというものです。

リレーショナルデータベースで複数のテーブルをJOINして結果を得るSQLをよく使われると思います。このような処理はグラフ操作でのノード間の接続をたどる操作に相当します。GQLはグラフをたどる処理がシンプルに記述できます。グラフデータベースでなければ絶対に解けない問題は多くありませんが、SQLで記述すると非常に複雑になる処理もGQLで記述するとシンプルに書ける場合があります。GQLとSQLは混ぜて使うこともできるため、グラフ処理で書きやすい部分はGQLでそれ以外の部分はSQLといった混在も可能です。

データ自体はテーブルにあるため、グラフ操作のために既存のテーブルからデータを移動して二重に持ったり、データを同期する仕組みを作る必要がありません。

### 全文検索としての機能

全文検索の機能自体は多くのRDBで実装されています。Spannerの全文検索機能は、GoogleのWeb検索で培われた自然言語処理の技術を活かして多言語対応をしているのが特長です。

また、全文検索は更新頻度が高いと全文検索用のインデックスの更新がボトルネックとなる場合がありますが、Spannerでは書き込みのスケーラビリティを向上させる仕組みがあるためノード数を増やすことで書き込み性能もスケールアウトすることが可能です。

### Google Cloudの他サービスとの親和性が高い

SpannerはBigQueryを始め、Google Cloudのサービスと連携できます。
SpannerのデータをBigQueryで分析したいときには、[DataBoost](https://cloud.google.com/spanner/docs/databoost/databoost-overview?hl=ja)という仕組みを使うことでSpannerのインスタンスに負荷をかけることなくデータが連携できます。逆にBigQueryで分析した結果を書き戻したい場合には、[リバースETL](https://cloud.google.com/bigquery/docs/export-to-spanner?hl=ja)を使えばBigQuery上ではクエリーを書くだけで連携できます。

その他、汎用的なCDCの仕組みである[Datastreamでデータを書き込む](https://cloud.google.com/dataflow/docs/guides/templates/provided/datastream-to-cloud-spanner?hl=ja)、[Vertex AIを使ってMLのモデルで推論](https://cloud.google.com/dataproc-serverless/docs/templates/storage-to-spanner?hl=ja)を行うなども可能です。

## まとめ

この記事では、Spannerを従来のRDBとして活用する方法について解説しました。主なポイントは以下のとおりです。

- Spannerはスケーラビリティだけでなく、従来のリレーショナルデータベースと同様の機能（SQL、トランザクション、データ型など）を備えています。
- Spannerへの接続には、クライアントライブラリとドライバー（JDBC、ORMなど）の2種類の方法があり、既存アプリケーションの移植にはドライバーの使用が有効です。ローカル開発にはエミュレータも利用できます。
- プライマリキーの設計においては、`AUTO INCREMENT` の代替として `DEFAULT (GENERATE_UUID())` や `BIT_REVERSE` 関数（分散用）などを適切に選択する必要があります。
- Spannerはスケーラビリティに加えて、高可用性、完全オンラインのメンテナンス/スキーマ変更、Read-Only Transaction、グラフデータベース機能、全文検索機能など、一般的なRDBと比較しても数多くの実用的なメリットを提供します。

現在は最小100 Processing Units（0.1ノード相当）や無料トライアルから利用を開始できます。これらの特徴を理解することで、Spannerを「普通のRDB」として効果的に活用し、アプリケーション開発や移植に役立てることができるでしょう。
