---
title: "Spannerが負荷に応じてスプリットを自動分割・再配置する仕組み"
emoji: "🗄️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["spanner", "googlecloud", "database"]
publication_name: "google_cloud_jp"
published: false
---

## tl;dr

- Spannerはテーブルのデータをプライマリキー順の連続した範囲である「スプリット（Split）」に区切り、複数のサーバーへ分散配置することでスケールアウトを実現しています。
- データ量が増えたときだけでなく、特定のキー範囲にアクセスが集中したときにも、Spannerは自動的にスプリットの境界（Split Boundary）を追加して細分化し、別のサーバーへ移動させて負荷を分散します。
- プライマリキーの先頭に連番やタイムスタンプを避けるべき理由や、親子テーブルを同居させるインターリーブの設計指針も、このスプリット自動分割の仕組みから論理的に説明できます。

## はじめに

Spannerは、リレーショナルデータベースの構造やトランザクション整合性を保ったまま水平方向にスケールアウトできるデータベースとして紹介されることが多い製品です。

一般的なリレーショナルデータベース（以下、RDB）を使ったことはあっても、Spannerのような分散データベースを触ったことがない方からすると、「1つのテーブルとしてSQLでクエリーや更新が行えるのに、裏側ではどうやって複数のサーバーへ処理を分散しているのか」「なぜRDBでは当たり前に使っていた1, 2, 3...と単調増加する連番（RDBの`AUTO_INCREMENT`など）のプライマリキーがSpannerでは推奨されないのか」と疑問に思われるのではないでしょうか。

本稿では、一般的なRDBのデータ管理方式と対比しながら、Spannerがデータ量やアクセス負荷に応じて自動的にデータを分割・再配置する「スプリット（Split）」の仕組みを視覚的に解説します。あわせて、この内部動作から導かれるプライマリキー設計の定石や、運用中にホットスポットが発生した際の見分け方について整理します。

## 一般的なRDBとSpannerのデータ配置の違い

MySQLやPostgreSQLなどの一般的なRDBでは、原則として1台のプライマリサーバーがデータベース全体のデータを保持し、すべての書き込みリクエストを処理します。負荷が増えた場合はサーバーのCPUやメモリを増強するスケールアップで対応しますが、1台のサーバー性能の限界を超えると、アプリケーション側でユーザーIDなどをもとに接続先データベースを振り分ける「シャーディング（水平分割）」を実装・運用しなければならなくなります。

一方で、Spannerではアプリケーションから見える論理的なテーブルは1つのまま、裏側の複数のサーバー（コンピュートリソース）がデータを分担して受け持ちます。このとき、データを分割してサーバーへ割り当てる単位となるのが「スプリット（Split）」です[^1]。

マニュアルにはスプリットについて、以下のような説明がされています[^1]。

> Spanner is a distributed database, which means that as your database grows, Spanner divides your data into chunks called "splits." Individual splits can move independently from each other and get assigned to different servers, which can be in different physical locations. A split holds a range of contiguous rows. The start and end keys of this range are called "split boundaries". Spanner automatically adds and removes split boundaries based on size and load, which changes the number of splits in the database.

テーブルの全行をプライマリキーの辞書順に一列に並べた上で、それを適当な区切り位置（スプリット境界 / Split Boundary）で複数の束（スプリット）に分け、それぞれの束を別々のサーバーに担当させるイメージです。なお、各スプリットのキー範囲は開始キーを含み終了キーを含まない半開区間として定義されるため、以降の図では`[1, 3500)`（1以上3500未満）のように表記します[^4]。

```html-embed
<div class="diagram">
  <div class="compare-grid">
    <div class="compare-card">
      <div class="card-head">
        <span class="badge badge-muted">一般的なRDB</span>
        <span class="card-title">単一サーバー管理と手動シャーディング</span>
      </div>
      <div class="card-desc">1台のプライマリが全行を処理します。限界を超えるとアプリ側で接続先DBを分ける必要があります。</div>
      <div class="flow-box">
        <div class="app-row">アプリケーション（振り分けロジックが必要）</div>
        <div class="arrow-down" aria-hidden="true">↓</div>
        <div class="server-grid-2">
          <div class="srv-box srv-neutral">
            <div class="srv-name">DB Server 1</div>
            <div class="range-pill pill-neutral">Users_A (1 .. 5000)</div>
          </div>
          <div class="srv-box srv-neutral">
            <div class="srv-name">DB Server 2</div>
            <div class="range-pill pill-neutral">Users_B (5001 .. 9999)</div>
          </div>
        </div>
      </div>
    </div>

    <div class="compare-card card-spanner">
      <div class="card-head">
        <span class="badge badge-primary">Spanner</span>
        <span class="card-title">プライマリキー範囲（スプリット）による自動分散</span>
      </div>
      <div class="card-desc">アプリからは単一テーブルに見え、裏側でキー範囲（Split）ごとに別サーバーへ自動配置されます。</div>
      <div class="flow-box">
        <div class="app-row app-spanner">アプリケーション（単一の <code>Users</code> テーブルとしてアクセス）</div>
        <div class="arrow-down arrow-primary" aria-hidden="true">↓ Spanner API ルーティング</div>
        <div class="server-grid-3">
          <div class="srv-box srv-active">
            <div class="srv-name">Server A</div>
            <div class="range-pill pill-primary">Split 1<br><code>[1, 3500)</code></div>
          </div>
          <div class="srv-box srv-active">
            <div class="srv-name">Server B</div>
            <div class="range-pill pill-primary">Split 2<br><code>[3500, 7200)</code></div>
          </div>
          <div class="srv-box srv-active">
            <div class="srv-name">Server C</div>
            <div class="range-pill pill-primary">Split 3<br><code>[7200, ∞)</code></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --primary: #0284c7; --primary-bg: #e0f2fe;
    box-sizing: border-box; padding: 14px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 13px; line-height: 1.4;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --primary: #38bdf8; --primary-bg: #0c4a6e;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .compare-grid { display: grid; grid-template-columns: 1fr 1.25fr; gap: 12px; }
  .compare-card {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 12px;
    display: flex; flex-direction: column; justify-content: space-between;
  }
  .card-spanner { border: 1.5px solid var(--primary); }
  .card-head { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
  .card-title { font-weight: 700; font-size: 13px; }
  .card-desc { font-size: 12px; color: var(--text-muted); margin-bottom: 10px; }
  .badge {
    display: inline-block; padding: 2px 6px; border-radius: 4px; font-size: 12px; font-weight: 700;
  }
  .badge-muted { background: var(--surface-sub); color: var(--text-muted); border: 1px solid var(--border); }
  .badge-primary { background: var(--primary-bg); color: var(--primary); }
  .flow-box { display: flex; flex-direction: column; gap: 6px; }
  .app-row {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px;
    padding: 6px 8px; text-align: center; font-size: 12px; font-weight: 600;
  }
  .app-spanner { border-color: var(--primary); background: var(--primary-bg); color: var(--primary); }
  .arrow-down { text-align: center; font-size: 12px; color: var(--text-muted); font-weight: 700; line-height: 1.1; }
  .arrow-primary { color: var(--primary); }
  .server-grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .server-grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 6px; }
  .srv-box {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px;
    padding: 6px; text-align: center; display: flex; flex-direction: column; gap: 4px;
  }
  .srv-active { background: var(--surface); border-color: var(--primary); }
  .srv-name { font-size: 12px; font-weight: 700; }
  .range-pill {
    font-size: 12px; padding: 3px 4px; border-radius: 4px; line-height: 1.25;
  }
  .pill-neutral { background: var(--surface); border: 1px solid var(--border); color: var(--text-muted); }
  .pill-primary { background: var(--primary-bg); color: var(--primary); font-weight: 600; }
  @media (max-width: 640px) {
    .diagram { padding: 10px; }
    .compare-grid { grid-template-columns: 1fr; gap: 8px; }
    .compare-card { padding: 8px 10px; }
    .card-desc { margin-bottom: 6px; }
    .range-pill br { display: none; }
  }
</style>
```

なお、実際のSpannerでは高い可用性を確保するため、各スプリットは複数のゾーン（障害ドメイン）に複製され、Paxos合意アルゴリズムによってデータの整合性が保たれています[^2]。本稿ではスプリットがどのように分割・移動するかに焦点を当てるため、以降の図ではゾーン間のレプリケーションを省略し、各スプリットを担当するサーバー単位で表現します。

## スプリットが自動的に分割・再配置される仕組み

テーブルを作成したばかりの初期状態やデータ量が少ない状態では、テーブル全体が1つのスプリットに収まり、1台のサーバーで処理されます[^1]。そこからデータベースの規模やトラフィックが成長するにつれて、Spannerは「データ量（サイズ）」と「アクセス負荷（ロード）」の2つの観点で自動的にスプリットの分割と統合を行います[^1][^3]。

### データ量が増えたときの分割（サイズベース）

テーブルに行が追加され続けて1つのスプリットのデータサイズが大きくなると、Spannerはそのスプリットの中間付近のプライマリキーに新しいスプリット境界を追加し、2つのスプリットに分割します。分割されたスプリットはそれぞれ独立して別のサーバーへ移動できるため、1台のサーバーのディスクやメモリ容量に縛られずにデータを増やし続けることができます。

### アクセスが集中したときの分割（負荷ベース）

もう1つの重要な仕組みが、データサイズにかかわらずアクセス負荷に応じて分割を行う「負荷ベースのスプリット分割（Load-based splitting）」です[^1][^4]。

マニュアルには負荷ベースのスプリット分割について、以下のような例が挙げられています[^1]。

> As an example of how Spanner performs load-based splitting to mitigate read hotspots, suppose your database contains a table with 10 rows that are read more frequently than all of the other rows in the table. Spanner can add split boundaries between each of those 10 rows so that they're each handled by a different server, rather than allowing all the reads of those rows to consume the resources of a single server.

たとえテーブル全体のデータサイズが小さくても、特定の10行だけにアクセスが集中していれば、Spannerはその10行の1行ずつにスプリット境界を差し込み、それぞれを別々のサーバーに担当させて負荷を逃がすという動作を行います。

負荷の集中を検知してからスプリットを分割し、別サーバーへ再配置するまでの流れを3つのステップに分けて見てみます。

```html-embed
<div class="diagram">
  <div class="step-list">
    <div class="step-box">
      <div class="step-header">
        <span class="step-num">Step 1</span>
        <span class="step-title">特定のキー範囲への負荷集中（ホットスポットの発生）</span>
      </div>
      <div class="step-body">
        <div class="srv-row">
          <div class="srv-card srv-hot">
            <div class="srv-top">
              <span class="srv-label">Server A（Split 1: <code>[1, 1000)</code>）</span>
              <span class="status-tag tag-hot">CPU 高負荷 (Hot)</span>
            </div>
            <div class="key-bar">
              <div class="key-seg seg-cold">Key: <code>[1, 400)</code></div>
              <div class="key-seg seg-hot">Key: <code>[400, 410)</code> (アクセス集中)</div>
              <div class="key-seg seg-cold">Key: <code>[410, 1000)</code></div>
            </div>
          </div>
          <div class="srv-card srv-idle">
            <div class="srv-top">
              <span class="srv-label">Server B / C</span>
              <span class="status-tag tag-idle">余裕あり</span>
            </div>
            <div class="idle-note">担当スプリットなし（または低負荷）</div>
          </div>
        </div>
      </div>
    </div>

    <div class="step-box">
      <div class="step-header">
        <span class="step-num">Step 2</span>
        <span class="step-title">スプリット境界（Split Boundary）の自動追加</span>
      </div>
      <div class="step-body">
        <div class="srv-card srv-warn">
          <div class="srv-top">
            <span class="srv-label">Server A</span>
            <span class="status-tag tag-warn">バックグラウンドで分割実行</span>
          </div>
          <div class="split-3-grid">
            <div class="split-piece piece-cold">Split 1a: <code>[1, 400)</code></div>
            <div class="split-piece piece-hot">Split 1b: <code>[400, 405)</code></div>
            <div class="split-piece piece-hot">Split 1c: <code>[405, 410)</code></div>
            <div class="split-piece piece-cold">Split 1d: <code>[410, 1000)</code></div>
          </div>
        </div>
      </div>
    </div>

    <div class="step-box">
      <div class="step-header">
        <span class="step-num">Step 3</span>
        <span class="step-title">別サーバーへのスプリット移動と負荷分散の完了</span>
      </div>
      <div class="step-body">
        <div class="srv-3-grid">
          <div class="srv-card srv-ok">
            <div class="srv-top">
              <span class="srv-label">Server A</span>
              <span class="status-tag tag-ok">負荷解消</span>
            </div>
            <div class="moved-pill">Split 1a <code>[1, 400)</code> + 1d</div>
          </div>
          <div class="srv-card srv-ok">
            <div class="srv-top">
              <span class="srv-label">Server B</span>
              <span class="status-tag tag-ok">負荷分散</span>
            </div>
            <div class="moved-pill pill-active">Split 1b <code>[400, 405)</code></div>
          </div>
          <div class="srv-card srv-ok">
            <div class="srv-top">
              <span class="srv-label">Server C</span>
              <span class="status-tag tag-ok">負荷分散</span>
            </div>
            <div class="moved-pill pill-active">Split 1c <code>[405, 410)</code></div>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --primary: #0284c7; --primary-bg: #e0f2fe;
    --green: #15803d; --green-bg: #dcfce7;
    --amber: #b45309; --amber-bg: #fef3c7;
    --red: #b91c1c; --red-bg: #fee2e2;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.35;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --primary: #38bdf8; --primary-bg: #0c4a6e;
    --green: #4ade80; --green-bg: #064e3b;
    --amber: #fbbf24; --amber-bg: #451a03;
    --red: #f87171; --red-bg: #450a0a;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .step-list { display: flex; flex-direction: column; gap: 8px; }
  .step-box { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 8px 10px; }
  .step-header { display: flex; align-items: center; gap: 8px; margin-bottom: 6px; }
  .step-num {
    background: var(--primary-bg); color: var(--primary); font-weight: 700;
    padding: 1px 6px; border-radius: 4px; font-size: 12px; flex-shrink: 0;
  }
  .step-title { font-weight: 700; font-size: 13px; }
  .srv-row { display: grid; grid-template-columns: 1.7fr 1fr; gap: 8px; }
  .srv-3-grid { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 8px; }
  .srv-card {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px; padding: 6px 8px;
  }
  .srv-hot { border: 1.5px solid var(--red); }
  .srv-warn { border: 1.5px dashed var(--amber); }
  .srv-ok { border: 1.5px solid var(--green); }
  .srv-idle { border-style: dashed; }
  .srv-top { display: flex; justify-content: space-between; align-items: center; gap: 4px; margin-bottom: 5px; flex-wrap: wrap; }
  .srv-label { font-weight: 700; font-size: 12px; }
  .status-tag { padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; }
  .tag-hot { background: var(--red-bg); color: var(--red); }
  .tag-warn { background: var(--amber-bg); color: var(--amber); }
  .tag-ok { background: var(--green-bg); color: var(--green); }
  .tag-idle { background: var(--surface); color: var(--text-muted); border: 1px solid var(--border); }
  .key-bar { display: grid; grid-template-columns: 1fr 1.4fr 1fr; gap: 3px; }
  .key-seg { padding: 3px 4px; border-radius: 4px; text-align: center; font-size: 12px; }
  .seg-cold { background: var(--surface); border: 1px solid var(--border); color: var(--text-muted); }
  .seg-hot { background: var(--red-bg); border: 1px solid var(--red); color: var(--red); font-weight: 700; }
  .idle-note { color: var(--text-muted); font-size: 12px; text-align: center; padding: 3px 0; }
  .split-3-grid { display: grid; grid-template-columns: repeat(4, 1fr); gap: 4px; }
  .split-piece { padding: 3px 4px; border-radius: 4px; text-align: center; font-size: 12px; }
  .piece-cold { background: var(--surface); border: 1px solid var(--border); color: var(--text-muted); }
  .piece-hot { background: var(--amber-bg); border: 1.5px dashed var(--amber); color: var(--amber); font-weight: 700; }
  .moved-pill {
    background: var(--surface); border: 1px solid var(--border); border-radius: 4px;
    padding: 3px 6px; text-align: center; font-size: 12px;
  }
  .pill-active { background: var(--primary-bg); border-color: var(--primary); color: var(--primary); font-weight: 700; }
  @media (max-width: 640px) {
    .diagram { padding: 8px; }
    .step-box { padding: 6px 8px; }
    .srv-row, .srv-3-grid { grid-template-columns: 1fr; gap: 4px; }
    .key-bar { grid-template-columns: 1fr; gap: 2px; }
    .split-3-grid { grid-template-columns: 1fr 1fr; gap: 3px; }
    .idle-note { display: none; }
  }
</style>
```

まずStep 1では、`Key: [1, 1000)`を受け持つ1つのスプリット（Split 1）がServer Aに配置されています。このうち`[400, 410)`の範囲にアクセスが集中すると、Server AのCPU使用率が上昇します。Spanner内部では、単一サーバーのリソース制約に対してスプリットの負荷が50%を超えた状態を「Warm」、100%に達してリクエストのレイテンシに影響が出うる状態を「Hot（ホットスポット）」として検知します[^4]。

続いてStep 2では、Spannerがバックグラウンドで負荷の高いキー範囲の前後や内部に新しいスプリット境界を追加し、`Split 1a`〜`1d`へと細かく分割します[^1][^4]。このとき、最小で1行単位までスプリットの境界を狭めることができます[^1]。

最後にStep 3では、分割された`Split 1b`と`Split 1c`をインスタンス内の余裕があるServer BやServer Cへ移動させます。これにより、当初はServer Aの1台だけで受けていた集中アクセスが複数のサーバーで並列に処理されるようになり、数分おきに段階的にスループットが改善していきます[^1]。反対に、アクセスが落ち着いて負荷の低い状態が続いたスプリット同士は、再び自動的に統合（マージ）されます[^3]。

### 自動分割にかかる時間と事前分割（Pre-splitting）

このようにSpannerはトラフィックの変化に合わせて自律的にスプリットを再構成しますが、負荷の検知から分割・移動が完了するまでには数分程度の時間がかかります[^1][^3]。緩やかにトラフィックが増える通常の運用では問題になりませんが、「空の新規テーブルに対して最初から大量のデータを一括投入する場面」や「大規模なキャンペーン開始で秒間アクセス数が一気に跳ね上がる場面」では、自動分割が追いつく前に最初の1スプリット（1台のサーバー）が飽和し、レイテンシ悪化やタイムアウトを招くことがあります[^3]。

そうした予測可能な急激な負荷増加に対しては、あらかじめキー空間にスプリット境界を作成しておく「事前分割（Pre-splitting / Split Points）」という機能も用意されています[^3]。

## なぜプライマリキーに連番やタイムスタンプを使うとスケールしないのか

ここまでのスプリット自動分割の仕組みを踏まえると、Spannerのスキーマ設計で必ず言及される「プライマリキーの先頭に単調増加する値（連番やタイムスタンプ）を使ってはならない」というルールの理由がはっきりと見えてきます[^5]。

一般的なRDBでは、`AUTO_INCREMENT`による`1, 2, 3...`と単調増加する連番や、現在時刻のタイムスタンプをプライマリキーにすることがよくあります。B+Treeの末尾に順番に追記されるため、単一サーバーのディスクI/Oやページ分割の観点ではむしろ効率が良いケースが多いためです。

しかし、キー範囲ごとにサーバーを分担するSpannerで同じ設計を行うと、先ほどの負荷ベースのスプリット分割がうまく機能しなくなります。

```html-embed
<div class="diagram">
  <div class="pk-grid">
    <div class="pk-card card-bad">
      <div class="pk-head">
        <span class="pk-badge badge-red">アンチパターン</span>
        <span class="pk-title">単調増加キー（連番 / 先頭タイムスタンプ）</span>
      </div>
      <div class="pk-desc">新しい書き込みが常にキー空間の末尾（最大値側）に集中します。スプリットを分割しても、次の新規行はすべて分割後の「新しい末尾スプリット」へ入るため、常に1台のサーバーに負荷が集中し続けます。</div>
      <div class="split-list">
        <div class="sp-row sp-cold">
          <span class="sp-name">Split 1 (Server A)</span>
          <span class="sp-range">Key: <code>[1, 1000)</code></span>
          <span class="sp-load load-zero">新規書込 0%</span>
        </div>
        <div class="sp-row sp-cold">
          <span class="sp-name">Split 2 (Server B)</span>
          <span class="sp-range">Key: <code>[1000, 2000)</code></span>
          <span class="sp-load load-zero">新規書込 0%</span>
        </div>
        <div class="sp-row sp-hot">
          <span class="sp-name">Split 3 (Server C)</span>
          <span class="sp-range">Key: <code>[2000, ∞)</code> ⇐ ID: 2001, 2002...</span>
          <span class="sp-load load-full">書込 100% 集中</span>
        </div>
      </div>
    </div>

    <div class="pk-card card-good">
      <div class="pk-head">
        <span class="pk-badge badge-green">推奨パターン</span>
        <span class="pk-title">分散キー（UUID v4 / ビット反転 / ハッシュ先頭）</span>
      </div>
      <div class="pk-desc">新しいキー値がキー空間全体にランダム（一様）に散らばります。分割されたすべてのスプリットへ同時に書き込みが振り分けられるため、サーバー台数に応じて書き込み性能がスケールします。</div>
      <div class="split-list">
        <div class="sp-row sp-ok">
          <span class="sp-name">Split 1 (Server A)</span>
          <span class="sp-range">Key: <code>[0000.., 5555..)</code> ⇐ <code>2a9f..</code></span>
          <span class="sp-load load-even">書込 約33%</span>
        </div>
        <div class="sp-row sp-ok">
          <span class="sp-name">Split 2 (Server B)</span>
          <span class="sp-range">Key: <code>[5555.., aaaa..)</code> ⇐ <code>7c1b..</code></span>
          <span class="sp-load load-even">書込 約33%</span>
        </div>
        <div class="sp-row sp-ok">
          <span class="sp-name">Split 3 (Server C)</span>
          <span class="sp-range">Key: <code>[aaaa.., ∞)</code> ⇐ <code>e402..</code></span>
          <span class="sp-load load-even">書込 約33%</span>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --green: #15803d; --green-bg: #dcfce7;
    --red: #b91c1c; --red-bg: #fee2e2;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.4;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --green: #4ade80; --green-bg: #064e3b;
    --red: #f87171; --red-bg: #450a0a;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .pk-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 10px; }
  .pk-card {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 10px 12px;
    display: flex; flex-direction: column; justify-content: space-between;
  }
  .card-bad { border: 1.5px solid var(--red); }
  .card-good { border: 1.5px solid var(--green); }
  .pk-head { display: flex; align-items: center; gap: 6px; margin-bottom: 6px; flex-wrap: wrap; }
  .pk-badge { padding: 1px 6px; border-radius: 4px; font-weight: 700; font-size: 12px; }
  .badge-red { background: var(--red-bg); color: var(--red); }
  .badge-green { background: var(--green-bg); color: var(--green); }
  .pk-title { font-weight: 700; font-size: 13px; }
  .pk-desc { font-size: 12px; color: var(--text-muted); margin-bottom: 8px; }
  .split-list { display: flex; flex-direction: column; gap: 5px; }
  .sp-row {
    display: flex; justify-content: space-between; align-items: center; gap: 6px; flex-wrap: wrap;
    padding: 6px 8px; border-radius: 6px; border: 1px solid var(--border); background: var(--surface-sub);
  }
  .sp-cold { opacity: 0.75; }
  .sp-hot { background: var(--red-bg); border-color: var(--red); }
  .sp-ok { background: var(--surface); border-color: var(--green); }
  .sp-name { font-weight: 700; font-size: 12px; }
  .sp-range { font-size: 12px; color: var(--text-muted); }
  .sp-hot .sp-range { color: var(--red); font-weight: 600; }
  .sp-load { padding: 1px 6px; border-radius: 4px; font-size: 12px; font-weight: 700; }
  .load-zero { background: var(--surface); color: var(--text-muted); }
  .load-full { background: var(--surface); color: var(--red); border: 1px solid var(--red); }
  .load-even { background: var(--green-bg); color: var(--green); }
  @media (max-width: 640px) {
    .diagram { padding: 8px; }
    .pk-grid { grid-template-columns: 1fr; gap: 8px; }
    .pk-card { padding: 8px 10px; }
    .pk-desc { margin-bottom: 6px; }
    .sp-row { padding: 4px 6px; }
  }
</style>
```

たとえば、`1, 2, 3...`と増え続ける連番や現在時刻をプライマリキーの先頭に置いたとします。すると、新しく`INSERT`される行は常にキー空間の一番末尾にあるスプリット（図のSplit 3）に書き込まれます。

Spannerが末尾のスプリットの負荷上昇を検知し、`Key: 2000`の位置でスプリットを分割したとしても、その直後にやってくる`2001, 2002...`という書き込みはすべて分割後の最大値側（`[2000, ∞)`）のスプリットへ入ります。つまり、せっかくスプリットを分割しても過去のキー範囲（`[1000, 2000)`）には二度と新規の書き込みが来ず、常に「その時点で一番末尾を担当している1台のサーバー」だけがすべての書き込みを受け止め続けることになります[^4][^5]。これはセカンダリインデックスでも同様であり、更新頻度の高いタイムスタンプ列を先頭にした通常のインデックスを作成すると、インデックス側のスプリット末尾で同じホットスポットが発生します[^5]。

これを防ぐため、Spannerでは新しい行のプライマリキーがキー空間全体に散らばるように設計します[^5]。図の右側（推奨パターン）のように、生成されるキーの先頭部分が`2a9f..`、`7c1b..`、`e402..`とばらければ、既存のすべてのスプリット（Server A〜C）へ同時に書き込みが振り分けられ、サーバー台数を増やした分だけ書き込み性能を引き出すことができます。

キーを分散させる代表的な方法には、ランダムなUUID v4（`NEW_UUID()`関数など）の採用、連番のビット順序を反転させて大きな整数空間へ値を散らすビット反転シーケンス（`bit_reversed_positive`）、あるいは複合プライマリキーの先頭に`UserId`やハッシュ値などのばらつきのある列を置いて2列目にタイムスタンプを置く（`PRIMARY KEY (UserId, LastAccess)`）といった手法があります[^5]。なお、RDBの単調増加する連番とは異なり、Spannerの`IDENTITY`列やそのDDLエイリアスである`AUTO_INCREMENT`（PostgreSQL方言では`SERIAL`）は、内部でこのビット反転シーケンスを使って自動的に値を分散させる仕組みになっています[^8]。

## 関連する複数テーブルを同じスプリットに同居させる（インターリーブ）

スプリットによってデータが複数のサーバーに分散されると、1つのサーバーの限界を超えてスケールできる一方で、別の考慮点も生まれます。それは、複数のスプリットをまたぐ読み書きにはサーバー間の通信コストがかかるという点です[^2][^6]。

たとえば、歌手テーブル（`Singers`）とアルバムテーブル（`Albums`）を通常の独立したテーブル（トップレベルテーブル）として作成した場合、Spannerはそれぞれのテーブルを別々にスプリット分割し、異なるサーバーへ配置します[^1]。すると、「ある1人の歌手と、その歌手に紐づくアルバム一覧をまとめて取得・更新する」というよくある処理のたびに、複数のサーバーをまたいだ通信や、サーバー間で整合性を取るための合意処理（2フェーズコミット / 2PC）が発生しやすくなります[^1][^2]。

そこでSpannerには、親テーブルの行と同じスプリットの中に子テーブルの行を物理的に並べて格納する「テーブルのインターリーブ（`INTERLEAVE IN PARENT`）」という機能があります[^1][^6]。

```sql:schema.sql
CREATE TABLE Singers (
  SingerId   INT64 NOT NULL,
  FirstName  STRING(1024),
  LastName   STRING(1024),
) PRIMARY KEY (SingerId);

CREATE TABLE Albums (
  SingerId     INT64 NOT NULL,
  AlbumId      INT64 NOT NULL,
  AlbumTitle   STRING(MAX),
) PRIMARY KEY (SingerId, AlbumId),
  INTERLEAVE IN PARENT Singers ON DELETE CASCADE;
```

このように子テーブル（`Albums`）のプライマリキーの先頭に親テーブル（`Singers`）のプライマリキーを含め、`INTERLEAVE IN PARENT`を宣言すると、ストレージ上のスプリット内での並び順は以下のようになります[^1]。

```html-embed
<div class="diagram">
  <div class="storage-grid">
    <div class="split-box">
      <div class="split-head">
        <span class="split-label">Split 1 (Server A)</span>
        <span class="split-range">Key Range (SingerId): <code>[1, 5000)</code></span>
      </div>
      <div class="parent-row">
        <div class="row-line">
          <span class="row-tag tag-parent">親: Singers</span>
          <span class="row-data">SingerId = <code>1024</code> ("Alice")</span>
        </div>
        <div class="child-group">
          <div class="child-row">
            <span class="row-tag tag-child">子: Albums</span>
            <span>SingerId = <code>1024</code>, AlbumId = <code>1</code></span>
          </div>
          <div class="child-row">
            <span class="row-tag tag-child">子: Albums</span>
            <span>SingerId = <code>1024</code>, AlbumId = <code>2</code></span>
          </div>
        </div>
      </div>
    </div>

    <div class="split-divider">
      <span>スプリット境界（原則として親テーブルの行の間で分割される）</span>
    </div>

    <div class="split-box">
      <div class="split-head">
        <span class="split-label">Split 2 (Server B)</span>
        <span class="split-range">Key Range (SingerId): <code>[5000, ∞)</code></span>
      </div>
      <div class="parent-row">
        <div class="row-line">
          <span class="row-tag tag-parent">親: Singers</span>
          <span class="row-data">SingerId = <code>6400</code> ("Bob")</span>
        </div>
        <div class="child-group">
          <div class="child-row">
            <span class="row-tag tag-child">子: Albums</span>
            <span>SingerId = <code>6400</code>, AlbumId = <code>1</code></span>
          </div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --primary: #0284c7; --primary-bg: #e0f2fe;
    --green: #15803d; --green-bg: #dcfce7; --amber: #b45309;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.4;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --primary: #38bdf8; --primary-bg: #0c4a6e;
    --green: #4ade80; --green-bg: #064e3b; --amber: #fbbf24;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .storage-grid { display: flex; flex-direction: column; gap: 8px; }
  .split-box {
    background: var(--surface); border: 1.5px solid var(--primary); border-radius: 8px; padding: 8px 10px;
  }
  .split-head {
    display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 6px;
    padding-bottom: 5px; margin-bottom: 6px; border-bottom: 1px solid var(--border); font-size: 12px;
  }
  .split-label { font-weight: 700; color: var(--primary); }
  .split-range { color: var(--text-muted); }
  .parent-row {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px; padding: 6px 8px;
  }
  .row-line { display: flex; align-items: center; gap: 8px; flex-wrap: wrap; margin-bottom: 6px; }
  .row-data { font-size: 12px; font-weight: 600; }
  .child-group { display: flex; flex-direction: column; gap: 4px; padding-left: 10px; border-left: 2px solid var(--green); }
  .child-row {
    background: var(--surface); border: 1px solid var(--border); border-radius: 4px;
    padding: 4px 8px; font-size: 12px; display: flex; align-items: center; flex-wrap: wrap; gap: 8px;
  }
  .row-tag {
    display: inline-block; padding: 1px 6px; border-radius: 4px; font-size: 12px; font-weight: 700;
  }
  .tag-parent { background: var(--primary-bg); color: var(--primary); }
  .tag-child { background: var(--green-bg); color: var(--green); }
  .split-divider {
    display: flex; align-items: center; gap: 8px; color: var(--amber); font-size: 12px; font-weight: 600;
  }
  .split-divider::before, .split-divider::after {
    content: ""; flex: 1; border-top: 1.5px dashed var(--amber);
  }
  @media (max-width: 640px) {
    .diagram { padding: 8px; }
    .split-box { padding: 6px 8px; }
    .parent-row { padding: 5px 6px; }
    .child-row { padding: 3px 6px; gap: 4px; }
  }
</style>
```

インターリーブされたテーブルでは、データの局所性（ローカリティ）を保つため、Spannerは原則として親テーブルの行の間にスプリット境界を追加します[^6]。これにより、特定の`SingerId`に紐づく親行と子行のツリー全体が同じスプリット（同じサーバー）に保たれ、`SingerId`を指定した結合（JOIN）やトランザクションを別サーバーと通信することなくローカルで完結させることができます[^1][^6]。

ただし、特定の親1行に紐づく子行のデータ量がスプリットのサイズ上限を超えるほど肥大化した場合や、特定の子行に激しいアクセス集中（ホットスポット）が発生した場合には、Spannerは例外的に子テーブルの行間にもスプリット境界を追加し、該当する子行とその配下の子孫行を別スプリットへ切り離して負荷を分散します[^1][^6]。

## 運用中に自動分割で解消できないホットスポットを見分けるには

ここまでに見た通り、スプリットの分割や移動はSpannerが裏側で自動的に行うため、普段の運用で開発者やDB管理者がスプリットの境界位置を意識する必要はありません。一方で、負荷テストや本番運用において「特定のスプリットが過負荷になっているが、Spannerの自動分割だけでは解消できていない状態」を見分けられると、トラブルシューティングが格段に進めやすくなります。

Spannerには、スプリットごとの負荷状況を可視化するGoogle Cloudコンソールの「Hotspot insights」ダッシュボードや、SQLで直接クエリーできるシステム統計ビュー`SPANNER_SYS.SPLIT_STATS_TOP_MINUTE`（および`10MINUTE`、`HOUR`）が用意されています[^4][^7]。

```sql:check_hot_splits.sql
SELECT
  INTERVAL_END,
  SPLIT_START,
  SPLIT_LIMIT,
  CPU_USAGE_SCORE,
  AFFECTED_TABLES,
  UNSPLITTABLE_REASONS
FROM SPANNER_SYS.SPLIT_STATS_TOP_MINUTE
ORDER BY INTERVAL_END DESC, CPU_USAGE_SCORE DESC
LIMIT 10;
```

このシステム統計ビューでは、各スプリットの開始キーである`SPLIT_START`（範囲に含む）と終了キーである`SPLIT_LIMIT`（範囲に含まない）から、どのキー範囲で負荷が高まっているかを特定できます[^4]。あわせて、単一サーバーのリソース制約に達している割合を示す`CPU_USAGE_SCORE`（50%超でWarm、100%でHot）や、負荷ベースのスプリット分割で解消できない原因を示す`UNSPLITTABLE_REASONS`列を確認できます[^4]。

| `UNSPLITTABLE_REASONS`の値 | 内部で起きている状態 | 主な原因と対処の方向性 |
| :--- | :--- | :--- |
| （空配列 `[]`） | 分割による解消が可能、または一時的な負荷 | 新たに発生した負荷に対してSpannerがバックグラウンドで分割を進めている途中か、短時間で収まった負荷です。10分以内に解消すれば対応は不要です[^4][^7]。 |
| `MOVING_HOT_SPOT` | 高負荷なキー範囲が時間とともに移動している | 連番やタイムスタンプ先頭など単調増加・減少するキーへの`INSERT`や、キー順の連続的な読み取りにより、Spannerが分割する前にホットスポットの位置が移動し続けています。UUIDやハッシュなどの分散キーへの見直しが必要です[^4]。 |
| `HOT_ROW` | 単一の1行に負荷が集中している | スプリットは最小でも1行より細かくは分割できないため、1つのキーに過剰な読み書きが集中すると別サーバーへ負荷を分けられません。カウンタ行のシャード分割やキャッシュを検討します[^4]。 |
| `UNISOLATABLE_HOT_ROW` | 狭いキー範囲に負荷が集中し境界を入れられない | 単一行または共通プレフィックスを持つ隣接行に激しい負荷が集中しており、適切な分割点を作れない状態です。`HOT_ROW`と同様に特定キーへのアクセス集中を緩和します[^4]。 |
| `LARGE_SCAN_HOT_SPOT` | 広範囲のスキャン操作により負荷が高い | 広いキー範囲を読むクエリーやDMLが多発しています。スプリットを細かくしすぎるとかえって範囲スキャンの性能が落ちるため、Spannerは過剰な分割を控えます。適切なインデックスの追加やスキャン行数の削減を行います[^4]。 |

運用上の判断基準として、`CPU_USAGE_SCORE`が100%に達していても、それが数分で収まり`UNSPLITTABLE_REASONS`が空であれば、Spannerの自動分割と再配置が正常に機能して負荷が分散されたと判断できます[^4][^7]。逆に、`CPU_USAGE_SCORE`が100%の状態が10分以上継続し、かつ上記の`UNSPLITTABLE_REASONS`が記録されている場合は、データベース側の自動分割だけでは解消できないアクセスパターンになっているため、スキーマやクエリーの見直しに進むのが効果的です[^4][^7]。

## まとめ

Spannerが負荷やデータ量に応じてスプリットを分割・再配置する仕組みについて整理しました。

- Spannerはテーブルやインデックスのデータをプライマリキー順の連続した範囲（スプリット）に分割し、複数のサーバーへ割り当てて処理します。
- データサイズが大きくなったときだけでなく、特定のキー範囲へのアクセス負荷が高まったときにも、自動的にスプリット境界を追加して別のサーバーへ移動させることでホットスポットを緩和します。
- プライマリキーの先頭に連番やタイムスタンプを使うと、スプリットを分割しても常に一番末尾のスプリット（1台のサーバー）に新しい書き込みが集中し続けるため、UUIDやハッシュなどキー空間全体に散らばるキー設計が重要になります。
- 関連の強い親子テーブルは`INTERLEAVE IN PARENT`で同じスプリットに同居させることで、複数サーバーをまたぐ通信を抑えて効率よく処理できます。

これらの内部動作のイメージを持っておくことで、RDBとの設計上の違いに戸惑うことなく、Spannerのスケーラビリティを活かしたテーブル設計や運用判断が行えるようになります。

[^1]: [Schemas overview - Database splits | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/schema-and-data-model#database-splits)
[^2]: [Life of Spanner Reads & Writes | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/whitepapers/life-of-reads-and-writes)
[^3]: [Pre-splitting overview | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/pre-splitting-overview)
[^4]: [Split hotspot statistics | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/introspection/hot-split-statistics)
[^5]: [Schema design best practices | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/schema-design)
[^6]: [Optimizing Schema Design for Spanner | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/whitepapers/optimizing-schema-design)
[^7]: [Find hotspots in your database (Hotspot insights) | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/find-hotspots-in-database)
[^8]: [Default primary key values (IDENTITY / AUTO_INCREMENT) | Cloud Spanner Documentation](https://docs.cloud.google.com/spanner/docs/primary-key-default-value)
