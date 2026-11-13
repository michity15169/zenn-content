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
        <span class="card-title">アプリ側での手動シャーディング</span>
      </div>
      <div class="card-desc">1台の限界を超えると、アプリ側でキー範囲ごとに接続先DBをパチパチと手動で切り替える必要があります。</div>
      <div class="flow-box">
        <div class="app-row app-rdb">
          <div class="app-title">アプリケーション（振り分け実装・保守が必要）</div>
          <div class="rdb-switch">
            <span class="sw-opt sw-1"><code>id &lt;= 5000</code> → DB 1 へ接続</span>
            <span class="sw-opt sw-2"><code>id &gt; 5000</code> → DB 2 へ接続</span>
          </div>
        </div>
        <div class="rdb-wires" aria-hidden="true">
          <div class="wire wire-1">↓ 接続切替</div>
          <div class="wire wire-2">↓ 接続切替</div>
        </div>
        <div class="server-grid-2">
          <div class="srv-box srv-rdb-1">
            <div class="srv-name">DB Server 1</div>
            <div class="range-pill pill-neutral">Users_A (1..5000)</div>
          </div>
          <div class="srv-box srv-rdb-2">
            <div class="srv-name">DB Server 2</div>
            <div class="range-pill pill-neutral">Users_B (5001..9999)</div>
          </div>
        </div>
      </div>
    </div>

    <div class="compare-card card-spanner">
      <div class="card-head">
        <span class="badge badge-primary">Spanner</span>
        <span class="card-title">スプリット単位での自動ルーティング</span>
      </div>
      <div class="card-desc">アプリからは常に単一テーブルに見え、裏側でキー範囲（Split）ごとの担当ノードへ自動振り分けされます。</div>
      <div class="flow-box">
        <div class="app-row app-spanner">
          <div class="app-title">アプリケーション（単一の <code>Users</code> テーブルへ接続）</div>
          <div class="sp-note">振り分けロジック不要・接続先は常に1つ</div>
        </div>
        <div class="arrow-down arrow-primary" aria-hidden="true">
          <span class="flow-track"><span class="flow-dot dot-primary"></span></span>
          <span>↓ Spanner API 自動ルーティング</span>
        </div>
        <div class="server-grid-3">
          <div class="srv-box srv-active srv-anim-1">
            <div class="srv-name">Node 1</div>
            <div class="range-pill pill-primary">Split 1<br><code>[1, 3500)</code></div>
          </div>
          <div class="srv-box srv-active srv-anim-2">
            <div class="srv-name">Node 2</div>
            <div class="range-pill pill-primary">Split 2<br><code>[3500, 7200)</code></div>
          </div>
          <div class="srv-box srv-active srv-anim-3">
            <div class="srv-name">Node 3</div>
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
    --amber: #b45309; --amber-bg: #fef3c7;
    box-sizing: border-box; padding: 14px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 13px; line-height: 1.4;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --primary: #38bdf8; --primary-bg: #0c4a6e;
    --amber: #fbbf24; --amber-bg: #451a03;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .compare-grid { display: grid; grid-template-columns: 1fr 1.2fr; gap: 12px; }
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
    display: flex; flex-direction: column; gap: 4px;
  }
  .app-title { font-size: 12px; font-weight: 700; }
  .rdb-switch { display: grid; grid-template-columns: 1fr 1fr; gap: 4px; }
  .sw-opt {
    padding: 2px 4px; border-radius: 4px; font-size: 12px; font-weight: 600;
    background: var(--surface); border: 1px solid var(--border); color: var(--text-muted);
  }
  .sw-1 { animation: rdb-sw-1 2.4s infinite steps(1, end); }
  .sw-2 { animation: rdb-sw-2 2.4s infinite steps(1, end); }
  .rdb-wires { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; text-align: center; font-size: 12px; font-weight: 700; }
  .wire-1 { animation: rdb-wire-1 2.4s infinite steps(1, end); }
  .wire-2 { animation: rdb-wire-2 2.4s infinite steps(1, end); }
  .srv-rdb-1 { animation: rdb-srv-1 2.4s infinite steps(1, end); }
  .srv-rdb-2 { animation: rdb-srv-2 2.4s infinite steps(1, end); }

  .app-spanner { border-color: var(--primary); background: var(--primary-bg); color: var(--primary); }
  .sp-note { font-size: 12px; font-weight: 600; opacity: 0.9; }
  .arrow-down {
    display: flex; align-items: center; justify-content: center; gap: 6px;
    font-size: 12px; color: var(--text-muted); font-weight: 700; line-height: 1.1;
  }
  .arrow-primary { color: var(--primary); }
  .flow-track {
    position: relative; display: inline-block; width: 6px; height: 18px;
    background: var(--border); border-radius: 3px; overflow: hidden;
  }
  .flow-dot {
    position: absolute; left: 0; top: -6px; width: 6px; height: 6px; border-radius: 50%;
    animation: flow-drop 1.8s infinite ease-in-out;
  }
  .dot-primary { background: var(--primary); }
  .server-grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 6px; }
  .server-grid-3 { display: grid; grid-template-columns: 1fr 1fr 1fr; gap: 6px; }
  .srv-box {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px;
    padding: 6px; text-align: center; display: flex; flex-direction: column; gap: 4px;
  }
  .srv-active {
    background: var(--surface); border-color: var(--primary);
    animation: srv-receive 2.4s infinite ease-in-out;
  }
  .srv-anim-1 { animation-delay: 0s; }
  .srv-anim-2 { animation-delay: 0.8s; }
  .srv-anim-3 { animation-delay: 1.6s; }
  .srv-name { font-size: 12px; font-weight: 700; }
  .range-pill {
    font-size: 12px; padding: 3px 4px; border-radius: 4px; line-height: 1.25;
  }
  .pill-neutral { background: var(--surface); border: 1px solid var(--border); color: var(--text-muted); }
  .pill-primary { background: var(--primary-bg); color: var(--primary); font-weight: 600; }

  @keyframes rdb-sw-1 {
    0%, 49% { background: var(--amber-bg); color: var(--amber); border-color: var(--amber); font-weight: 700; }
    50%, 100% { background: var(--surface); color: var(--text-muted); border-color: var(--border); font-weight: 600; opacity: 0.6; }
  }
  @keyframes rdb-sw-2 {
    0%, 49% { background: var(--surface); color: var(--text-muted); border-color: var(--border); font-weight: 600; opacity: 0.6; }
    50%, 100% { background: var(--amber-bg); color: var(--amber); border-color: var(--amber); font-weight: 700; }
  }
  @keyframes rdb-wire-1 {
    0%, 49% { color: var(--amber); opacity: 1; }
    50%, 100% { color: var(--text-muted); opacity: 0.25; }
  }
  @keyframes rdb-wire-2 {
    0%, 49% { color: var(--text-muted); opacity: 0.25; }
    50%, 100% { color: var(--amber); opacity: 1; }
  }
  @keyframes rdb-srv-1 {
    0%, 49% { border-color: var(--amber); background: var(--amber-bg); }
    50%, 100% { border-color: var(--border); background: var(--surface-sub); opacity: 0.7; }
  }
  @keyframes rdb-srv-2 {
    0%, 49% { border-color: var(--border); background: var(--surface-sub); opacity: 0.7; }
    50%, 100% { border-color: var(--amber); background: var(--amber-bg); }
  }
  @keyframes flow-drop {
    0% { top: -6px; opacity: 0; }
    30% { opacity: 1; }
    80% { opacity: 1; }
    100% { top: 18px; opacity: 0; }
  }
  @keyframes srv-receive {
    0%, 100% { box-shadow: 0 0 0 0 transparent; }
    50% { box-shadow: 0 0 0 2px var(--primary-bg); }
  }
  @media (max-width: 640px) {
    .diagram { padding: 10px; }
    .compare-grid { grid-template-columns: 1fr; gap: 8px; }
    .compare-card { padding: 8px 10px; }
    .card-desc { margin-bottom: 6px; }
    .rdb-switch { grid-template-columns: 1fr; gap: 2px; }
    .range-pill br { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after {
      animation: none !important; transition: none !important;
    }
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

負荷の集中を検知してからスプリット境界を設定して分割し、既存ノード間での負荷分散、さらにノード追加（スケールアウト）による負荷分散へと推移する一連の流れを1つの図解で見てみます（デフォルトで自動再生され、上部のボタンで各ステップに固定することもできます）。

```html-embed
<div class="diagram">
  <input type="radio" name="sp-sim" id="sim-auto" class="sim-radio" checked>
  <input type="radio" name="sp-sim" id="sim-p1" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p2" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p3" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p4" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p5" class="sim-radio">

  <div class="sim-toolbar">
    <div class="sim-tabs">
      <label for="sim-auto" class="sim-tab tab-auto">自動再生 (25秒)</label>
      <label for="sim-p1" class="sim-tab tab-p1">1. 初期</label>
      <label for="sim-p2" class="sim-tab tab-p2">2. 負荷集中</label>
      <label for="sim-p3" class="sim-tab tab-p3">3. 2分割 (500)</label>
      <label for="sim-p4" class="sim-tab tab-p4">4. Node 2へ移動</label>
      <label for="sim-p5" class="sim-tab tab-p5">5. 再分割・Node 3追加</label>
    </div>
    <div class="tl-bar"><div class="tl-progress"></div></div>
    <div class="phase-banner">
      <div class="ph-text ph-1"><strong>Step 1 (初期状態):</strong> 1つのスプリット（<code>[1, 1000)</code>）に全データがあり、<code>Node 1</code> のみで処理しています。</div>
      <div class="ph-text ph-2"><strong>Step 2 (アクセス集中):</strong> まだ1つのスプリットのまま負荷が高まり、<code>Split 1 [1, 1000)</code> 全体が高負荷（Hot）として検知されます。</div>
      <div class="ph-text ph-3"><strong>Step 3 (Split point追加で2分割):</strong> Spannerが境界（<code>500</code>）を1つ追加し、<code>[1, 500)</code> と <code>[500, 1000)</code> の2つへ分割します。</div>
      <div class="ph-text ph-4"><strong>Step 4 (Node 2へ移動):</strong> 片方の <code>[500, 1000)</code> が <code>Node 2</code> へ移動します（今回は後半にアクセスが偏っていたため <code>Node 2</code> 側が高負荷となります）。</div>
      <div class="ph-text ph-5"><strong>Step 5 (ノード追加と再度の2分割):</strong> <code>Node 3</code> が追加され、<code>Node 2</code> 上でさらに境界（<code>750</code>）を追加して2分割し、<code>[750, 1000)</code> を <code>Node 3</code> へ分散します。</div>
    </div>
  </div>

  <!-- 2次元マトリクス: 横軸=キー空間 / 縦軸=ノードレーン -->
  <div class="matrix-wrap">
    <div class="matrix-head">
      <div class="mh-node">ノード / CPU使用率</div>
      <div class="mh-keys">
        <span><code>[1, 500)</code></span>
        <span><code>[500, 750)</code></span>
        <span><code>[750, 1000)</code></span>
      </div>
    </div>

    <div class="lanes">
      <!-- Node 1 行 -->
      <div class="lane-row lane-1">
        <div class="lane-info">
          <div class="li-top">
            <span class="li-name">Node 1</span>
            <span class="state-layer">
              <span class="st-item ph-1 badge-n1">CPU 20%</span>
              <span class="st-item ph-2 badge-hot">CPU 95%</span>
              <span class="st-item ph-3 badge-hot">CPU 92%</span>
              <span class="st-item ph-4 badge-ok">CPU 20%</span>
              <span class="st-item ph-5 badge-ok">CPU 20%</span>
            </span>
          </div>
          <div class="cpu-meter" aria-label="Node 1 CPU使用率">
            <div class="cpu-fill fill-n1"></div>
          </div>
        </div>

        <div class="lane-track">
          <!-- Step 1〜2: まだ分割されていない単一スプリット [1, 1000) 全体 -->
          <div class="unified-bar">
            <div class="ub-single state-layer">
              <span class="st-item ph-1">Split 1: <code>[1, 1000)</code>（境界なし・単一スプリット）</span>
              <span class="st-item ph-2 ub-hot-lbl">Split 1: <code>[1, 1000)</code>（高負荷 / Hot 検知）</span>
            </div>
          </div>

          <!-- Step 3〜5: まず 500 で2分割 → 後半 [500,1000) が Node 2 へ移動し、Step 5 でさらに 750 で2分割 -->
          <div class="split-grid">
            <!-- 左半分 [1, 500): ずっと Node 1 に留まる -->
            <div class="sp-block sp-left">
              <span class="sp-id"><code>[1, 500)</code></span>
              <span class="sp-sub">Node 1 担当</span>
            </div>

            <!-- 右半分の前半 [500, 750): Step 3 では [500,1000) の一部として一体表示、Step 4 で Node 2 へ、Step 5 で独立 -->
            <div class="sp-block sp-mid">
              <span class="sp-id state-layer">
                <span class="st-item ph-3"><code>[500, 1000)</code></span>
                <span class="st-item ph-4"><code>[500, 1000)</code></span>
                <span class="st-item ph-5"><code>[500, 750)</code></span>
              </span>
              <span class="sp-sub state-layer">
                <span class="st-item ph-3">分割された高負荷範囲</span>
                <span class="st-item ph-4">Node 2 へ移動</span>
                <span class="st-item ph-5">Node 2 担当</span>
              </span>
            </div>

            <!-- 右半分の後半 [750, 1000): Step 3〜4 は sp-mid と一体化、Step 5 で境界750が入り Node 3 へ移動 -->
            <div class="sp-block sp-right">
              <span class="sp-id"><code>[750, 1000)</code></span>
              <span class="sp-sub">Node 3 へ移動</span>
            </div>
          </div>
        </div>
      </div>

      <!-- Node 2 行 -->
      <div class="lane-row lane-2">
        <div class="lane-info">
          <div class="li-top">
            <span class="li-name">Node 2</span>
            <span class="state-layer">
              <span class="st-item ph-1 badge-muted">CPU 10%</span>
              <span class="st-item ph-2 badge-muted">CPU 10%</span>
              <span class="st-item ph-3 badge-muted">CPU 10%</span>
              <span class="st-item ph-4 badge-hot">CPU 85%</span>
              <span class="st-item ph-5 badge-n2">CPU 42%</span>
            </span>
          </div>
          <div class="cpu-meter" aria-label="Node 2 CPU使用率">
            <div class="cpu-fill fill-n2"></div>
          </div>
        </div>
        <div class="lane-track track-drop">
          <div class="drop-idle">
            <span class="state-layer">
              <span class="st-item ph-1 hint-txt">担当スプリットなし（余裕あり）</span>
              <span class="st-item ph-2 hint-txt">担当スプリットなし（余裕あり）</span>
              <span class="st-item ph-3 hint-txt">担当スプリットなし（余裕あり）</span>
            </span>
          </div>
        </div>
      </div>

      <!-- Node 3 行（Step 5 で初めて出現） -->
      <div class="lane-row lane-3">
        <div class="lane-info">
          <div class="li-top">
            <span class="li-name">Node 3 (追加)</span>
            <span class="badge-ok">CPU 43%</span>
          </div>
          <div class="cpu-meter" aria-label="Node 3 CPU使用率">
            <div class="cpu-fill fill-n3"></div>
          </div>
        </div>
        <div class="lane-track track-drop-3">
          <div class="drop-spacer"></div>
          <div class="drop-spacer"></div>
          <div class="drop-slot slot-n3"></div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --meter-bg: #e2e8f0;
    --n1: #0284c7; --n1-bg: #e0f2fe;
    --n2: #7c3aed; --n2-bg: #ede9fe;
    --n3: #15803d; --n3-bg: #dcfce7;
    --amber: #b45309; --amber-bg: #fef3c7;
    --red: #b91c1c; --red-bg: #fee2e2;
    --cycle: 25s;
    --row-h: 64px;
    --row-gap: 8px;
    --step-y: 72px;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.35;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --meter-bg: #0f172a;
    --n1: #38bdf8; --n1-bg: #0c4a6e;
    --n2: #a78bfa; --n2-bg: #3b0764;
    --n3: #4ade80; --n3-bg: #064e3b;
    --amber: #fbbf24; --amber-bg: #451a03;
    --red: #f87171; --red-bg: #450a0a;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .sim-radio { display: none; }

  .sim-toolbar {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px;
    padding: 8px 10px; margin-bottom: 8px;
  }
  .sim-tabs { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 6px; }
  .sim-tab {
    padding: 2px 7px; border-radius: 4px; font-size: 12px; font-weight: 600;
    background: var(--surface-sub); color: var(--text-muted); border: 1px solid var(--border);
    cursor: pointer; user-select: none;
  }
  #sim-auto:checked ~ .sim-toolbar .tab-auto,
  #sim-p1:checked ~ .sim-toolbar .tab-p1,
  #sim-p2:checked ~ .sim-toolbar .tab-p2,
  #sim-p3:checked ~ .sim-toolbar .tab-p3,
  #sim-p4:checked ~ .sim-toolbar .tab-p4,
  #sim-p5:checked ~ .sim-toolbar .tab-p5 {
    background: var(--n1-bg); color: var(--n1); border-color: var(--n1); font-weight: 700;
  }
  .tl-bar { width: 100%; height: 4px; background: var(--surface-sub); border-radius: 2px; overflow: hidden; margin-bottom: 6px; }
  .tl-progress { height: 100%; background: var(--n1); width: 100%; }
  #sim-auto:checked ~ .sim-toolbar .tl-progress { animation: tl-run var(--cycle) infinite linear; }
  #sim-p1:checked ~ .sim-toolbar .tl-progress { width: 20%; }
  #sim-p2:checked ~ .sim-toolbar .tl-progress { width: 40%; background: var(--red); }
  #sim-p3:checked ~ .sim-toolbar .tl-progress { width: 60%; background: var(--amber); }
  #sim-p4:checked ~ .sim-toolbar .tl-progress { width: 80%; background: var(--n2); }
  #sim-p5:checked ~ .sim-toolbar .tl-progress { width: 100%; background: var(--n3); }

  .phase-banner, .state-layer { display: grid; }
  .ph-text, .st-item { grid-area: 1 / 1; opacity: 0; pointer-events: none; transition: opacity 0.25s ease; }

  #sim-auto:checked ~ * .ph-1 { animation: vis-p1 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-2 { animation: vis-p2 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-3 { animation: vis-p3 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-4 { animation: vis-p4 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-5 { animation: vis-p5 var(--cycle) infinite; }
  #sim-p1:checked ~ * .ph-1,
  #sim-p2:checked ~ * .ph-2,
  #sim-p3:checked ~ * .ph-3,
  #sim-p4:checked ~ * .ph-4,
  #sim-p5:checked ~ * .ph-5 { opacity: 1; animation: none !important; }

  /* マトリクス本体 */
  .matrix-wrap {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px;
    padding: 8px 10px;
  }
  .matrix-head {
    display: grid; grid-template-columns: 155px 1fr; gap: 8px;
    padding-bottom: 6px; margin-bottom: 6px; border-bottom: 1px solid var(--border);
    font-size: 12px; font-weight: 700; color: var(--text-muted);
  }
  .mh-keys { display: grid; grid-template-columns: 2fr 1fr 1fr; gap: 6px; text-align: center; }

  .lanes { display: flex; flex-direction: column; gap: var(--row-gap); }
  .lane-row {
    display: grid; grid-template-columns: 155px 1fr; gap: 8px; align-items: center;
    height: var(--row-h); padding: 6px 8px; border-radius: 8px;
    background: var(--surface-sub); border: 1.5px solid var(--border);
    position: relative;
  }
  .lane-1 { z-index: 3; border-color: var(--n1); }
  .lane-2 { z-index: 2; }
  .lane-3 {
    z-index: 1; height: 0; opacity: 0; padding-top: 0; padding-bottom: 0;
    border-width: 0; margin-top: calc(-1 * var(--row-gap)); overflow: hidden;
    border-color: var(--n3);
  }

  .lane-info { display: flex; flex-direction: column; gap: 5px; }
  .li-top { display: flex; justify-content: space-between; align-items: center; gap: 4px; }
  .li-name { font-weight: 700; font-size: 12px; white-space: nowrap; }
  .cpu-meter {
    width: 100%; height: 10px; background: var(--meter-bg); border-radius: 5px;
    overflow: hidden; border: 1px solid var(--border);
  }
  .cpu-fill { height: 100%; border-radius: 4px; width: 20%; background: var(--n1); transition: width 0.5s ease, background 0.5s ease; }
  .fill-n2 { width: 10%; background: var(--text-muted); }
  .fill-n3 { width: 43%; background: var(--n3); }

  .badge-n1, .badge-n2, .badge-hot, .badge-ok, .badge-muted {
    display: inline-block; padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; text-align: right; white-space: nowrap;
  }
  .badge-n1 { background: var(--n1-bg); color: var(--n1); }
  .badge-n2 { background: var(--n2-bg); color: var(--n2); }
  .badge-ok { background: var(--n3-bg); color: var(--n3); }
  .badge-hot { background: var(--red-bg); color: var(--red); }
  .badge-muted { background: var(--surface); color: var(--text-muted); border: 1px solid var(--border); }

  .lane-track { position: relative; height: 48px; display: grid; }
  .unified-bar {
    grid-area: 1 / 1; z-index: 3;
    background: var(--n1-bg); border: 2px solid var(--n1); border-radius: 6px;
    display: grid; overflow: hidden;
    font-weight: 700; font-size: 12px; color: var(--n1);
  }
  .ub-single { grid-area: 1 / 1; display: grid; align-items: center; text-align: center; padding: 0 6px; }
  .ub-hot-split { grid-area: 1 / 1; display: grid; grid-template-columns: 1fr 1fr; align-items: stretch; }
  .ub-left { display: grid; align-items: center; text-align: center; opacity: 0.75; }
  .ub-right { display: grid; align-items: center; text-align: center; padding: 0 4px; }
  .ub-hot-lbl { color: var(--red); }

  /* スプリットトラック: [1,500) が 2fr、[500,750) が 1fr、[750,1000) が 1fr */
  .split-grid {
    grid-area: 1 / 1; display: grid; grid-template-columns: 2fr 1fr 1fr; gap: 6px; height: 100%;
  }
  .track-drop {
    display: grid; grid-template-columns: 1fr; height: 48px;
  }
  .track-drop-3 {
    display: grid; grid-template-columns: 2fr 1fr 1fr; gap: 6px; height: 48px;
  }
  .drop-spacer { background: transparent; border: none; }
  .drop-idle {
    display: flex; align-items: center; justify-content: center; height: 100%;
  }
  .sp-block {
    border-radius: 6px; display: flex; flex-direction: column; align-items: center; justify-content: center;
    background: var(--n1-bg); border: 1.5px solid var(--n1); color: var(--n1);
    font-weight: 700; position: relative; z-index: 5;
    transition: transform 0.9s cubic-bezier(0.22, 1, 0.36, 1), background 0.4s ease, border-color 0.4s ease, color 0.4s ease;
  }
  /* Step 3〜4では sp-mid が右2列分(2 / 4)を統合して [500, 1000) として振る舞う */
  .sp-left { grid-column: 1 / 2; grid-row: 1; }
  .sp-mid { grid-column: 2 / 4; grid-row: 1; }
  .sp-right { grid-column: 3 / 4; grid-row: 1; opacity: 0; pointer-events: none; z-index: 6; }

  .sp-id { font-size: 12px; font-weight: 700; line-height: 1.15; text-align: center; justify-items: center; }
  .sp-sub { font-size: 12px; font-weight: 600; line-height: 1.15; text-align: center; justify-items: center; }

  .drop-slot {
    border-radius: 6px; border: 1px dashed var(--border); background: var(--surface);
    display: flex; align-items: center; justify-content: center;
  }
  .hint-txt { font-size: 12px; color: var(--text-muted); }

  /* 自動再生 (25秒サイクル: P1 0-18%, P2 20-38%, P3 40-58%, P4 60-78%, P5 80-98%) */
  #sim-auto:checked ~ .matrix-wrap .unified-bar { animation: unibar-vis var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .split-grid  { animation: slots-vis var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .sp-mid      { animation: move-mid var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .sp-right    { animation: move-right var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-1      { animation: l1-box var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-2      { animation: l2-box var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .drop-idle   { animation: drop-vis var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-3      { animation: l3-add var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .fill-n1     { animation: cpu-n1 var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .fill-n2     { animation: cpu-n2 var(--cycle) infinite; }

  /* 手動フェーズ固定 */
  #sim-p1:checked ~ .matrix-wrap .split-grid,
  #sim-p2:checked ~ .matrix-wrap .split-grid { opacity: 0; pointer-events: none; }
  #sim-p2:checked ~ .matrix-wrap .unified-bar { border-color: var(--red); background: var(--red-bg); }
  #sim-p2:checked ~ .matrix-wrap .fill-n1 { width: 95%; background: var(--red); }
  #sim-p2:checked ~ .matrix-wrap .lane-1 { border-color: var(--red); }

  #sim-p3:checked ~ .matrix-wrap .unified-bar,
  #sim-p4:checked ~ .matrix-wrap .unified-bar,
  #sim-p5:checked ~ .matrix-wrap .unified-bar { opacity: 0; pointer-events: none; }

  #sim-p3:checked ~ .matrix-wrap .sp-mid { background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
  #sim-p3:checked ~ .matrix-wrap .fill-n1 { width: 92%; background: var(--red); }
  #sim-p3:checked ~ .matrix-wrap .lane-1 { border-color: var(--amber); }

  #sim-p4:checked ~ .matrix-wrap .sp-mid {
    transform: translateY(var(--step-y));
    background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red);
  }
  #sim-p4:checked ~ .matrix-wrap .fill-n1 { width: 20%; background: var(--n3); }
  #sim-p4:checked ~ .matrix-wrap .fill-n2 { width: 85%; background: var(--red); }
  #sim-p4:checked ~ .matrix-wrap .lane-1 { border-color: var(--n3); }
  #sim-p4:checked ~ .matrix-wrap .lane-2 { border-color: var(--red); }

  #sim-p5:checked ~ .matrix-wrap .sp-mid {
    grid-column: 2 / 3;
    transform: translateY(var(--step-y));
    background: var(--n2-bg); border-color: var(--n2); color: var(--n2);
  }
  #sim-p5:checked ~ .matrix-wrap .sp-right {
    opacity: 1;
    transform: translateY(calc(2 * var(--step-y)));
    background: var(--n3-bg); border-color: var(--n3); color: var(--n3);
  }
  #sim-p5:checked ~ .matrix-wrap .fill-n1 { width: 20%; background: var(--n3); }
  #sim-p5:checked ~ .matrix-wrap .fill-n2 { width: 42%; background: var(--n2); }
  #sim-p5:checked ~ .matrix-wrap .lane-1 { border-color: var(--n3); }
  #sim-p5:checked ~ .matrix-wrap .lane-2 { border-color: var(--n2); }
  #sim-p4:checked ~ .matrix-wrap .drop-idle,
  #sim-p5:checked ~ .matrix-wrap .drop-idle { opacity: 0; }
  #sim-p5:checked ~ .matrix-wrap .lane-3 {
    height: var(--row-h); opacity: 1; padding: 6px 8px; border-width: 1.5px; margin-top: 0;
  }

  @keyframes tl-run {
    0% { width: 0%; background: var(--n1); }
    19% { width: 20%; background: var(--n1); }
    20% { width: 20%; background: var(--red); }
    39% { width: 40%; background: var(--red); }
    40% { width: 40%; background: var(--amber); }
    59% { width: 60%; background: var(--amber); }
    60% { width: 60%; background: var(--n2); }
    79% { width: 80%; background: var(--n2); }
    80% { width: 80%; background: var(--n3); }
    100% { width: 100%; background: var(--n3); }
  }
  @keyframes vis-p1 { 0%, 18% { opacity: 1; } 20%, 100% { opacity: 0; } }
  @keyframes vis-p2 { 0%, 19% { opacity: 0; } 20%, 38% { opacity: 1; } 40%, 100% { opacity: 0; } }
  @keyframes vis-p3 { 0%, 39% { opacity: 0; } 40%, 58% { opacity: 1; } 60%, 100% { opacity: 0; } }
  @keyframes vis-p4 { 0%, 59% { opacity: 0; } 60%, 78% { opacity: 1; } 80%, 100% { opacity: 0; } }
  @keyframes vis-p5 { 0%, 79% { opacity: 0; } 80%, 98% { opacity: 1; } 100% { opacity: 0; } }

  @keyframes unibar-vis {
    0%, 18% { opacity: 1; background: var(--n1-bg); border-color: var(--n1); }
    20%, 38% { opacity: 1; background: var(--red-bg); border-color: var(--red); }
    40%, 98% { opacity: 0; }
    100% { opacity: 1; background: var(--n1-bg); border-color: var(--n1); }
  }
  @keyframes slots-vis { 0%, 38% { opacity: 0; gap: 0px; } 42%, 98% { opacity: 1; gap: 6px; } 100% { opacity: 0; } }

  @keyframes move-mid {
    0%, 38% { grid-column: 2 / 4; transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    40%, 59% { grid-column: 2 / 4; transform: translateY(0); background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
    67%, 78% { grid-column: 2 / 4; transform: translateY(var(--step-y)); background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
    80%, 98% { grid-column: 2 / 3; transform: translateY(var(--step-y)); background: var(--n2-bg); border: 1.5px solid var(--n2); color: var(--n2); }
    100% { grid-column: 2 / 4; transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
  }
  @keyframes move-right {
    0%, 78% { opacity: 0; transform: translateY(var(--step-y)); background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
    80% { opacity: 1; transform: translateY(var(--step-y)); background: var(--amber-bg); border: 1.5px dashed var(--amber); color: var(--amber); }
    87%, 98% { opacity: 1; transform: translateY(calc(2 * var(--step-y))); background: var(--n3-bg); border: 1.5px solid var(--n3); color: var(--n3); }
    100% { opacity: 0; transform: translateY(var(--step-y)); }
  }
  @keyframes l1-box {
    0%, 18% { border-color: var(--n1); }
    20%, 58% { border-color: var(--red); }
    60%, 98% { border-color: var(--n3); }
    100% { border-color: var(--n1); }
  }
  @keyframes l2-box {
    0%, 60% { border-color: var(--border); }
    66%, 78% { border-color: var(--red); }
    82%, 98% { border-color: var(--n2); }
    100% { border-color: var(--border); }
  }
  @keyframes drop-vis {
    0%, 58% { opacity: 1; }
    62%, 98% { opacity: 0; }
    100% { opacity: 1; }
  }
  @keyframes l3-add {
    0%, 78% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--row-gap)); }
    83%, 98% { height: var(--row-h); opacity: 1; padding-top: 6px; padding-bottom: 6px; border-width: 1.5px; margin-top: 0; }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--row-gap)); }
  }
  @keyframes cpu-n1 {
    0%, 18% { width: 20%; background: var(--n1); }
    20%, 58% { width: 95%; background: var(--red); }
    66%, 98% { width: 20%; background: var(--n3); }
    100% { width: 20%; background: var(--n1); }
  }
  @keyframes cpu-n2 {
    0%, 60% { width: 10%; background: var(--text-muted); }
    66%, 78% { width: 85%; background: var(--red); }
    85%, 98% { width: 42%; background: var(--n2); }
    100% { width: 10%; background: var(--text-muted); }
  }

  @media (max-width: 640px) {
    .diagram {
      padding: 8px;
      --row-h: 82px;
      --row-gap: 6px;
      --step-y: 88px;
    }
    .sim-toolbar, .matrix-wrap { padding: 6px 8px; }
    .matrix-head { grid-template-columns: 1fr; gap: 3px; }
    .mh-node { display: none; }
    .mh-keys { gap: 3px; font-size: 12px; }
    .lane-row { grid-template-columns: 1fr; gap: 4px; padding: 5px 6px; }
    .lane-info { flex-direction: row; align-items: center; justify-content: space-between; gap: 8px; }
    .li-top { gap: 6px; }
    .cpu-meter { width: 76px; flex-shrink: 0; }
    .lane-track { height: 44px; }
    .ub-left { display: none; }
    .ub-hot-split { grid-template-columns: 1fr; }
    .split-grid, .track-drop, .track-drop-3 { gap: 3px; }
    .sp-sub { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after { animation: none !important; transition: none !important; }
    #sim-auto:checked ~ * .ph-5 { opacity: 1; }
    #sim-auto:checked ~ .matrix-wrap .unified-bar { opacity: 0; }
    #sim-auto:checked ~ .matrix-wrap .split-grid { opacity: 1; }
    #sim-auto:checked ~ .matrix-wrap .lane-3 { height: var(--row-h); opacity: 1; padding: 6px 8px; border-width: 1.5px; margin-top: 0; }
    #sim-auto:checked ~ .matrix-wrap .sp-mid { grid-column: 2 / 3; transform: translateY(var(--step-y)); background: var(--n2-bg); border-color: var(--n2); color: var(--n2); }
    #sim-auto:checked ~ .matrix-wrap .sp-right { opacity: 1; transform: translateY(calc(2 * var(--step-y))); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
  }
</style>
```

まずStep 1（初期状態）では、`Key: [1, 1000)`を受け持つ1つのスプリットに全データが格納され、Node 1の1台で処理されています。続くStep 2（アクセス集中）では、まだスプリットは1つのままアクセス負荷が高まり、`Split 1 [1, 1000)`全体が高負荷状態として検知され、Node 1のCPU使用率が急上昇します。Spanner内部では、単一サーバーのリソース制約に対してスプリットの負荷が50%を超えた状態を「Warm」、100%に達してリクエストのレイテンシに影響が出うる状態を「Hot（ホットスポット）」として検知します[^4]。

続いてStep 3では、Spannerがバックグラウンドでスプリット境界（`500`）を追加し、`[1, 500)`と`[500, 1000)`の2つへ分割します[^1][^4]。Step 4では、高負荷な`[500, 1000)`をインスタンス内の余裕があるNode 2へ移動させることで、Node 1の負荷が一気に解消されます。

さらにトラフィックが増加してNode 3が追加（スケールアウト）されたStep 5では、まだ高負荷な`[500, 1000)`に対してNode 2上で再び境界（`750`）を追加して2分割し、片方の`[750, 1000)`を新しく追加されたNode 3へ再配置します。このように、Spannerは負荷に応じて「境界を追加して分割し、別ノードへ逃がす」という処理を数分おきに段階的に繰り返すことで、各ノードの負荷を平準化してスループットを向上させていきます[^1]。なお、アクセスが集中しているキーが少数行に限られる場合は、最小で1行単位までスプリットの境界を狭めることも可能です[^1]。反対に、アクセスが落ち着いて負荷の低い状態が続いたスプリット同士は、再び自動的に統合（マージ）されます[^3]。

### 自動分割にかかる時間と事前分割（Pre-splitting）

このようにSpannerはトラフィックの変化に合わせて自律的にスプリットを再構成しますが、負荷の検知から分割・移動が完了するまでには数分程度の時間がかかります[^1][^3]。緩やかにトラフィックが増える通常の運用では問題になりませんが、「空の新規テーブルに対して最初から大量のデータを一括投入する場面」や「大規模なキャンペーン開始で秒間アクセス数が一気に跳ね上がる場面」では、自動分割が追いつく前に最初の1スプリット（1台のサーバー）が飽和し、レイテンシ悪化やタイムアウトを招くことがあります[^3]。

そうした予測可能な急激な負荷増加に対しては、あらかじめキー空間にスプリット境界（Split points）を作成しておく「事前分割（Pre-splitting）」という機能が用意されています[^3]。実際にトラフィックが急増する前に事前分割とノードの追加を済ませておくと、以下の流れで高負荷を迎え撃つことができます。

```html-embed
<div class="diagram">
  <input type="radio" name="ps-sim" id="ps-auto" class="ps-radio" checked>
  <input type="radio" name="ps-sim" id="ps-p1" class="ps-radio">
  <input type="radio" name="ps-sim" id="ps-p2" class="ps-radio">
  <input type="radio" name="ps-sim" id="ps-p3" class="ps-radio">
  <input type="radio" name="ps-sim" id="ps-p4" class="ps-radio">
  <input type="radio" name="ps-sim" id="ps-p5" class="ps-radio">

  <div class="ps-toolbar">
    <div class="ps-tabs">
      <label for="ps-auto" class="ps-tab pst-auto">自動再生 (25秒)</label>
      <label for="ps-p1" class="ps-tab pst-p1">1. 初期 (1ノード)</label>
      <label for="ps-p2" class="ps-tab pst-p2">2. 分割点指定</label>
      <label for="ps-p3" class="ps-tab pst-p3">3. 事前分割</label>
      <label for="ps-p4" class="ps-tab pst-p4">4. ノード追加・分散</label>
      <label for="ps-p5" class="ps-tab pst-p5">5. 負荷到来</label>
    </div>
    <div class="ps-tl"><div class="ps-tl-bar"></div></div>
    <div class="ps-banner">
      <div class="ps-ph psp-1"><strong>Step 1 (初期状態):</strong> 高負荷が来る前の段階では、<code>Node 1</code> 上の1つのスプリット（<code>[1, 1000)</code>）にデータが格納されています。</div>
      <div class="ps-ph psp-2"><strong>Step 2 (事前準備・分割点指定):</strong> 大量アクセスの開始前（推奨12時間〜7日前）に、pre-splitting で分割点（<code>333</code>, <code>666</code>）を指定します。</div>
      <div class="ps-ph psp-3"><strong>Step 3 (スプリット分割):</strong> 負荷が来る前に、指定した分割点に沿ってスプリットが3つへ事前に分割されます。</div>
      <div class="ps-ph psp-4"><strong>Step 4 (ノード追加・徐々に分散):</strong> <code>Node 2</code>・<code>Node 3</code> を追加すると、分割済みのスプリットが追加ノードへ順次分散配置されます。</div>
      <div class="ps-ph psp-5"><strong>Step 5 (負荷到来・受け止め完了):</strong> 大量リクエストが一気に到来しても、すでに全ノードへ分散されているため CPU 45% 程度で安定して処理できます。</div>
    </div>
  </div>

  <div class="ps-matrix">
    <div class="ps-mhead">
      <div class="ps-mh-node">ノード / CPU使用率</div>
      <div class="ps-mh-keys">
        <span><code>[1, 333)</code></span>
        <span><code>[333, 666)</code></span>
        <span><code>[666, 1000)</code></span>
      </div>
    </div>

    <div class="ps-lanes">
      <!-- Node 1 行 -->
      <div class="ps-row ps-lane-1">
        <div class="ps-info">
          <div class="ps-itop">
            <span class="ps-iname">Node 1</span>
            <span class="ps-state">
              <span class="ps-st psp-1 ps-b-idle">CPU 15%</span>
              <span class="ps-st psp-2 ps-b-prep">CPU 15%</span>
              <span class="ps-st psp-3 ps-b-n1">CPU 15%</span>
              <span class="ps-st psp-4 ps-b-n1">CPU 15%</span>
              <span class="ps-st psp-5 ps-b-ok">CPU 45%</span>
            </span>
          </div>
          <div class="ps-meter"><div class="ps-fill ps-f1"></div></div>
        </div>

        <div class="ps-track">
          <!-- Step 1〜2: 単一スプリット（Step 2 で分割点マーカー表示） -->
          <div class="ps-unified">
            <div class="ps-u-seg">
              <span class="ps-state">
                <span class="ps-st psp-1"></span>
                <span class="ps-st psp-2 ps-pt-txt"><code>[1, 333)</code></span>
              </span>
            </div>
            <div class="ps-u-mid">
              <span class="ps-state">
                <span class="ps-st psp-1">Split 1: <code>[1, 1000)</code>（単一スプリット）</span>
                <span class="ps-st psp-2 ps-pt-txt">分割点指定: <code>333</code> | <code>666</code></span>
              </span>
            </div>
            <div class="ps-u-seg">
              <span class="ps-state">
                <span class="ps-st psp-1"></span>
                <span class="ps-st psp-2 ps-pt-txt"><code>[666, 1000)</code></span>
              </span>
            </div>
          </div>

          <!-- Step 3〜5: 3つに事前分割されたスプリット（Step 4 で Node 2, 3 へ順次移動） -->
          <div class="ps-split3">
            <div class="ps-blk ps-b1">
              <span class="ps-bid"><code>[1, 333)</code></span>
              <span class="ps-state">
                <span class="ps-st psp-3 ps-bsub">分割完了</span>
                <span class="ps-st psp-4 ps-bsub">Node 1</span>
                <span class="ps-st psp-5 ps-bsub-ok">負荷受信(安定)</span>
              </span>
            </div>
            <div class="ps-blk ps-b2">
              <span class="ps-bid"><code>[333, 666)</code></span>
              <span class="ps-state">
                <span class="ps-st psp-3 ps-bsub">分割完了</span>
                <span class="ps-st psp-4 ps-bsub">Node 2へ分散</span>
                <span class="ps-st psp-5 ps-bsub-ok">負荷受信(安定)</span>
              </span>
            </div>
            <div class="ps-blk ps-b3">
              <span class="ps-bid"><code>[666, 1000)</code></span>
              <span class="ps-state">
                <span class="ps-st psp-3 ps-bsub">分割完了</span>
                <span class="ps-st psp-4 ps-bsub">Node 3へ分散</span>
                <span class="ps-st psp-5 ps-bsub-ok">負荷受信(安定)</span>
              </span>
            </div>
          </div>
        </div>
      </div>

      <!-- Node 2 行（Step 4 で追加） -->
      <div class="ps-row ps-lane-2">
        <div class="ps-info">
          <div class="ps-itop">
            <span class="ps-iname">Node 2 (追加)</span>
            <span class="ps-state">
              <span class="ps-st psp-4 ps-b-n2">CPU 15%</span>
              <span class="ps-st psp-5 ps-b-ok">CPU 45%</span>
            </span>
          </div>
          <div class="ps-meter"><div class="ps-fill ps-f2"></div></div>
        </div>
        <div class="ps-track ps-drop-track">
          <div class="ps-spacer"></div>
          <div class="ps-drop"></div>
          <div class="ps-spacer"></div>
        </div>
      </div>

      <!-- Node 3 行（Step 4 で追加） -->
      <div class="ps-row ps-lane-3">
        <div class="ps-info">
          <div class="ps-itop">
            <span class="ps-iname">Node 3 (追加)</span>
            <span class="ps-state">
              <span class="ps-st psp-4 ps-b-n3">CPU 15%</span>
              <span class="ps-st psp-5 ps-b-ok">CPU 45%</span>
            </span>
          </div>
          <div class="ps-meter"><div class="ps-fill ps-f3"></div></div>
        </div>
        <div class="ps-track ps-drop-track">
          <div class="ps-spacer"></div>
          <div class="ps-spacer"></div>
          <div class="ps-drop"></div>
        </div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --meter-bg: #e2e8f0;
    --n1: #0284c7; --n1-bg: #e0f2fe;
    --n2: #7c3aed; --n2-bg: #ede9fe;
    --n3: #15803d; --n3-bg: #dcfce7;
    --amber: #b45309; --amber-bg: #fef3c7;
    --pscycle: 25s;
    --ps-rh: 58px;
    --ps-gap: 7px;
    --ps-sy: 65px;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.35;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --meter-bg: #0f172a;
    --n1: #38bdf8; --n1-bg: #0c4a6e;
    --n2: #a78bfa; --n2-bg: #3b0764;
    --n3: #4ade80; --n3-bg: #064e3b;
    --amber: #fbbf24; --amber-bg: #451a03;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .ps-radio { display: none; }

  .ps-toolbar {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px;
    padding: 8px 10px; margin-bottom: 8px;
  }
  .ps-tabs { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 6px; }
  .ps-tab {
    padding: 2px 7px; border-radius: 4px; font-size: 12px; font-weight: 600;
    background: var(--surface-sub); color: var(--text-muted); border: 1px solid var(--border);
    cursor: pointer; user-select: none;
  }
  #ps-auto:checked ~ .ps-toolbar .pst-auto,
  #ps-p1:checked ~ .ps-toolbar .pst-p1,
  #ps-p2:checked ~ .ps-toolbar .pst-p2,
  #ps-p3:checked ~ .ps-toolbar .pst-p3,
  #ps-p4:checked ~ .ps-toolbar .pst-p4,
  #ps-p5:checked ~ .ps-toolbar .pst-p5 {
    background: var(--n1-bg); color: var(--n1); border-color: var(--n1); font-weight: 700;
  }
  .ps-tl { width: 100%; height: 4px; background: var(--surface-sub); border-radius: 2px; overflow: hidden; margin-bottom: 6px; }
  .ps-tl-bar { height: 100%; background: var(--n1); width: 100%; }
  #ps-auto:checked ~ .ps-toolbar .ps-tl-bar { animation: pstl var(--pscycle) infinite linear; }
  #ps-p1:checked ~ .ps-toolbar .ps-tl-bar { width: 20%; }
  #ps-p2:checked ~ .ps-toolbar .ps-tl-bar { width: 40%; background: var(--amber); }
  #ps-p3:checked ~ .ps-toolbar .ps-tl-bar { width: 60%; background: var(--n1); }
  #ps-p4:checked ~ .ps-toolbar .ps-tl-bar { width: 80%; background: var(--n2); }
  #ps-p5:checked ~ .ps-toolbar .ps-tl-bar { width: 100%; background: var(--n3); }

  .ps-banner, .ps-state { display: grid; }
  .ps-ph, .ps-st { grid-area: 1 / 1; opacity: 0; pointer-events: none; transition: opacity 0.25s ease; text-align: center; }
  .ps-ph { text-align: left; }

  #ps-auto:checked ~ * .psp-1 { animation: psvis-1 var(--pscycle) infinite; }
  #ps-auto:checked ~ * .psp-2 { animation: psvis-2 var(--pscycle) infinite; }
  #ps-auto:checked ~ * .psp-3 { animation: psvis-3 var(--pscycle) infinite; }
  #ps-auto:checked ~ * .psp-4 { animation: psvis-4 var(--pscycle) infinite; }
  #ps-auto:checked ~ * .psp-5 { animation: psvis-5 var(--pscycle) infinite; }
  #ps-p1:checked ~ * .psp-1,
  #ps-p2:checked ~ * .psp-2,
  #ps-p3:checked ~ * .psp-3,
  #ps-p4:checked ~ * .psp-4,
  #ps-p5:checked ~ * .psp-5 { opacity: 1; animation: none !important; }

  .ps-matrix { background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 8px 10px; }
  .ps-mhead {
    display: grid; grid-template-columns: 155px 1fr; gap: 8px;
    padding-bottom: 5px; margin-bottom: 6px; border-bottom: 1px solid var(--border);
    font-size: 12px; font-weight: 700; color: var(--text-muted);
  }
  .ps-mh-keys { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; text-align: center; }
  .ps-lanes { display: flex; flex-direction: column; gap: var(--ps-gap); }
  .ps-row {
    display: grid; grid-template-columns: 155px 1fr; gap: 8px; align-items: center;
    height: var(--ps-rh); padding: 5px 8px; border-radius: 8px;
    background: var(--surface-sub); border: 1.5px solid var(--border); position: relative;
  }
  .ps-lane-1 { z-index: 3; border-color: var(--n1); }
  .ps-lane-2, .ps-lane-3 {
    z-index: 2; height: 0; opacity: 0; padding-top: 0; padding-bottom: 0;
    border-width: 0; margin-top: calc(-1 * var(--ps-gap)); overflow: hidden;
  }
  .ps-lane-2 { border-color: var(--n2); }
  .ps-lane-3 { z-index: 1; border-color: var(--n3); }

  .ps-info { display: flex; flex-direction: column; gap: 4px; }
  .ps-itop { display: flex; justify-content: space-between; align-items: center; gap: 4px; }
  .ps-iname { font-weight: 700; font-size: 12px; white-space: nowrap; }
  .ps-meter { width: 100%; height: 9px; background: var(--meter-bg); border-radius: 5px; overflow: hidden; border: 1px solid var(--border); }
  .ps-fill { height: 100%; border-radius: 4px; width: 15%; background: var(--n1); transition: width 0.4s ease, background 0.4s ease; }

  .ps-b-idle, .ps-b-prep, .ps-b-n1, .ps-b-n2, .ps-b-n3, .ps-b-ok {
    display: inline-block; padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; white-space: nowrap;
  }
  .ps-b-idle { background: var(--surface); color: var(--text-muted); border: 1px solid var(--border); }
  .ps-b-prep { background: var(--amber-bg); color: var(--amber); }
  .ps-b-n1 { background: var(--n1-bg); color: var(--n1); }
  .ps-b-n2 { background: var(--n2-bg); color: var(--n2); }
  .ps-b-n3, .ps-b-ok { background: var(--n3-bg); color: var(--n3); }

  .ps-track { position: relative; height: 42px; display: grid; }
  .ps-unified {
    grid-area: 1 / 1; z-index: 3;
    background: var(--n1-bg); border: 2px solid var(--n1); border-radius: 6px;
    display: grid; grid-template-columns: 1fr 1fr 1fr; align-items: stretch; overflow: hidden;
    font-weight: 700; font-size: 12px; color: var(--n1);
  }
  .ps-u-seg, .ps-u-mid { display: grid; align-items: center; text-align: center; padding: 0 3px; }
  .ps-pt-txt { color: var(--amber); font-weight: 700; }

  .ps-split3 { grid-area: 1 / 1; display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; height: 100%; }
  .ps-drop-track { display: grid; grid-template-columns: repeat(3, 1fr); gap: 6px; height: 42px; }
  .ps-spacer { background: transparent; border: none; }
  .ps-drop { border-radius: 6px; border: 1px dashed var(--border); background: var(--surface); }

  .ps-blk {
    border-radius: 6px; display: flex; flex-direction: column; align-items: center; justify-content: center;
    background: var(--n1-bg); border: 1.5px solid var(--n1); color: var(--n1);
    font-weight: 700; position: relative; z-index: 5; text-align: center;
  }
  .ps-bid { font-size: 12px; line-height: 1.1; }
  .ps-bsub, .ps-bsub-ok { font-size: 12px; font-weight: 600; line-height: 1.1; }

  /* 自動再生 (25秒サイクル: P1 0-18%, P2 20-38%, P3 40-58%, P4 60-78%, P5 80-98%) */
  #ps-auto:checked ~ .ps-matrix .ps-unified { animation: ps-unibar var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-u-mid   { animation: ps-umid var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-split3  { animation: ps-splits var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-b1      { animation: ps-blk-1 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-b2      { animation: ps-blk-2 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-b3      { animation: ps-blk-3 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-lane-1  { animation: ps-l1 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-lane-2  { animation: ps-add-2 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-lane-3  { animation: ps-add-3 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-f1      { animation: ps-cpu-all var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-f2      { animation: ps-cpu-23 var(--pscycle) infinite; }
  #ps-auto:checked ~ .ps-matrix .ps-f3      { animation: ps-cpu-23 var(--pscycle) infinite; }

  /* 手動固定 */
  #ps-p1:checked ~ .ps-matrix .ps-split3,
  #ps-p2:checked ~ .ps-matrix .ps-split3 { opacity: 0; pointer-events: none; }
  #ps-p2:checked ~ .ps-matrix .ps-unified { border-color: var(--amber); background: var(--amber-bg); }
  #ps-p2:checked ~ .ps-matrix .ps-u-mid { border-left: 2px dashed var(--amber); border-right: 2px dashed var(--amber); }
  #ps-p2:checked ~ .ps-matrix .ps-lane-1 { border-color: var(--amber); }

  #ps-p3:checked ~ .ps-matrix .ps-unified,
  #ps-p4:checked ~ .ps-matrix .ps-unified,
  #ps-p5:checked ~ .ps-matrix .ps-unified { opacity: 0; pointer-events: none; }

  #ps-p4:checked ~ .ps-matrix .ps-lane-2,
  #ps-p4:checked ~ .ps-matrix .ps-lane-3,
  #ps-p5:checked ~ .ps-matrix .ps-lane-2,
  #ps-p5:checked ~ .ps-matrix .ps-lane-3 { height: var(--ps-rh); opacity: 1; padding: 5px 8px; border-width: 1.5px; margin-top: 0; }

  #ps-p4:checked ~ .ps-matrix .ps-b2,
  #ps-p5:checked ~ .ps-matrix .ps-b2 { transform: translateY(var(--ps-sy)); background: var(--n2-bg); border-color: var(--n2); color: var(--n2); }
  #ps-p4:checked ~ .ps-matrix .ps-b3,
  #ps-p5:checked ~ .ps-matrix .ps-b3 { transform: translateY(calc(2 * var(--ps-sy))); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }

  #ps-p4:checked ~ .ps-matrix .ps-f2 { width: 15%; background: var(--n2); }
  #ps-p4:checked ~ .ps-matrix .ps-f3 { width: 15%; background: var(--n3); }

  #ps-p5:checked ~ .ps-matrix .ps-f1,
  #ps-p5:checked ~ .ps-matrix .ps-f2,
  #ps-p5:checked ~ .ps-matrix .ps-f3 { width: 45%; background: var(--n3); }
  #ps-p5:checked ~ .ps-matrix .ps-b1,
  #ps-p5:checked ~ .ps-matrix .ps-b2,
  #ps-p5:checked ~ .ps-matrix .ps-b3 { background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
  #ps-p5:checked ~ .ps-matrix .ps-lane-1,
  #ps-p5:checked ~ .ps-matrix .ps-lane-2,
  #ps-p5:checked ~ .ps-matrix .ps-lane-3 { border-color: var(--n3); }

  @keyframes pstl {
    0% { width: 0%; background: var(--n1); } 19% { width: 20%; background: var(--n1); }
    20% { width: 20%; background: var(--amber); } 39% { width: 40%; background: var(--amber); }
    40% { width: 40%; background: var(--n1); } 59% { width: 60%; background: var(--n1); }
    60% { width: 60%; background: var(--n2); } 79% { width: 80%; background: var(--n2); }
    80% { width: 80%; background: var(--n3); } 100% { width: 100%; background: var(--n3); }
  }
  @keyframes psvis-1 { 0%, 18% { opacity: 1; } 20%, 100% { opacity: 0; } }
  @keyframes psvis-2 { 0%, 19% { opacity: 0; } 20%, 38% { opacity: 1; } 40%, 100% { opacity: 0; } }
  @keyframes psvis-3 { 0%, 39% { opacity: 0; } 40%, 58% { opacity: 1; } 60%, 100% { opacity: 0; } }
  @keyframes psvis-4 { 0%, 59% { opacity: 0; } 60%, 78% { opacity: 1; } 80%, 100% { opacity: 0; } }
  @keyframes psvis-5 { 0%, 79% { opacity: 0; } 80%, 98% { opacity: 1; } 100% { opacity: 0; } }

  @keyframes ps-unibar {
    0%, 18% { opacity: 1; background: var(--n1-bg); border-color: var(--n1); }
    20%, 38% { opacity: 1; background: var(--amber-bg); border-color: var(--amber); }
    40%, 98% { opacity: 0; } 100% { opacity: 1; background: var(--n1-bg); border-color: var(--n1); }
  }
  @keyframes ps-umid {
    0%, 18% { border-color: transparent; }
    20%, 38% { border-left: 2px dashed var(--amber); border-right: 2px dashed var(--amber); }
    40%, 100% { border-color: transparent; }
  }
  @keyframes ps-splits { 0%, 38% { opacity: 0; gap: 0px; } 42%, 98% { opacity: 1; gap: 6px; } 100% { opacity: 0; } }
  @keyframes ps-blk-1 {
    0%, 78% { background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    82%, 98% { background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
    100% { background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
  }
  @keyframes ps-blk-2 {
    0%, 59% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    65%, 78% { transform: translateY(var(--ps-sy)); background: var(--n2-bg); border-color: var(--n2); color: var(--n2); }
    82%, 98% { transform: translateY(var(--ps-sy)); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
    100% { transform: translateY(0); }
  }
  @keyframes ps-blk-3 {
    0%, 66% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    74%, 98% { transform: translateY(calc(2 * var(--ps-sy))); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
    100% { transform: translateY(0); }
  }
  @keyframes ps-l1 {
    0%, 18% { border-color: var(--n1); } 20%, 38% { border-color: var(--amber); }
    40%, 78% { border-color: var(--n1); } 82%, 98% { border-color: var(--n3); } 100% { border-color: var(--n1); }
  }
  @keyframes ps-add-2 {
    0%, 58% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--ps-gap)); }
    62%, 98% { height: var(--ps-rh); opacity: 1; padding-top: 5px; padding-bottom: 5px; border-width: 1.5px; margin-top: 0; border-color: var(--n2); }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--ps-gap)); }
  }
  @keyframes ps-add-3 {
    0%, 63% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--ps-gap)); }
    67%, 98% { height: var(--ps-rh); opacity: 1; padding-top: 5px; padding-bottom: 5px; border-width: 1.5px; margin-top: 0; border-color: var(--n3); }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--ps-gap)); }
  }
  @keyframes ps-cpu-all {
    0%, 78% { width: 15%; background: var(--n1); }
    84%, 98% { width: 45%; background: var(--n3); }
    100% { width: 15%; background: var(--n1); }
  }
  @keyframes ps-cpu-23 {
    0%, 78% { width: 15%; background: var(--n2); }
    84%, 98% { width: 45%; background: var(--n3); }
    100% { width: 15%; background: var(--n2); }
  }

  @media (max-width: 640px) {
    .diagram { padding: 8px; --ps-rh: 74px; --ps-gap: 5px; --ps-sy: 79px; }
    .ps-toolbar, .ps-matrix { padding: 6px 8px; }
    .ps-mhead { grid-template-columns: 1fr; gap: 3px; }
    .ps-mh-node { display: none; }
    .ps-mh-keys { gap: 3px; font-size: 12px; }
    .ps-row { grid-template-columns: 1fr; gap: 4px; padding: 4px 6px; }
    .ps-info { flex-direction: row; align-items: center; justify-content: space-between; gap: 8px; }
    .ps-meter { width: 76px; flex-shrink: 0; }
    .ps-track, .ps-drop-track { height: 38px; }
    .ps-u-seg { display: none; }
    .ps-unified { grid-template-columns: 1fr; }
    .ps-split3, .ps-drop-track { gap: 3px; }
    .ps-bsub, .ps-bsub-ok { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after { animation: none !important; transition: none !important; }
    #ps-auto:checked ~ * .psp-5 { opacity: 1; }
    #ps-auto:checked ~ .ps-matrix .ps-unified { opacity: 0; }
    #ps-auto:checked ~ .ps-matrix .ps-split3 { opacity: 1; }
    #ps-auto:checked ~ .ps-matrix .ps-lane-2,
    #ps-auto:checked ~ .ps-matrix .ps-lane-3 { height: var(--ps-rh); opacity: 1; padding: 5px 8px; border-width: 1.5px; margin-top: 0; }
    #ps-auto:checked ~ .ps-matrix .ps-b2 { transform: translateY(var(--ps-sy)); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
    #ps-auto:checked ~ .ps-matrix .ps-b3 { transform: translateY(calc(2 * var(--ps-sy))); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
    #ps-auto:checked ~ .ps-matrix .ps-f1,
    #ps-auto:checked ~ .ps-matrix .ps-f2,
    #ps-auto:checked ~ .ps-matrix .ps-f3 { width: 45%; background: var(--n3); }
  }
</style>
```

まずStep 1では1つのスプリットに全データが収まっており、Step 2の事前準備としてトラフィック増加の12時間〜7日前を目安にSplit points（分割点）を指定します[^3]。するとStep 3でトラフィック到来前にスプリットが事前分割され、Step 4でノードを追加するにつれて分割されたスプリットが追加ノードへ徐々に分散配置されます。これにより、Step 5で一気に高い負荷が来ても最初から複数のノードへ分散した状態で受け止めることができます[^3]。なお、公式ガイドでは目安として1ノードあたり10個のSplit pointsを作成することが推奨されています[^3]。

## なぜプライマリキーに連番やタイムスタンプを使うとスケールしないのか

ここまでのスプリット自動分割の仕組みを踏まえると、Spannerのスキーマ設計で必ず言及される「プライマリキーの先頭に単調増加する値（連番やタイムスタンプ）を使ってはならない」というルールの理由がはっきりと見えてきます[^5]。

一般的なRDBでは、`AUTO_INCREMENT`による`1, 2, 3...`と単調増加する連番や、現在時刻のタイムスタンプをプライマリキーにすることがよくあります。B+Treeの末尾に順番に追記されるため、単一サーバーのディスクI/Oやページ分割の観点ではむしろ効率が良いケースが多いためです。

しかし、キー範囲ごとにサーバーを分担するSpannerで同じ設計を行うと、先ほどの負荷ベースのスプリット分割がうまく機能しなくなります。

```html-embed
<div class="diagram">
  <input type="radio" name="pk-sim" id="pk-auto" class="pk-radio" checked>
  <input type="radio" name="pk-sim" id="pk-p1" class="pk-radio">
  <input type="radio" name="pk-sim" id="pk-p2" class="pk-radio">
  <input type="radio" name="pk-sim" id="pk-p3" class="pk-radio">
  <input type="radio" name="pk-sim" id="pk-p4" class="pk-radio">

  <div class="pk-toolbar">
    <div class="pk-tabs">
      <label for="pk-auto" class="pk-tab pkt-auto">自動再生 (20秒)</label>
      <label for="pk-p1" class="pk-tab pkt-p1">1. 初期 (1台)</label>
      <label for="pk-p2" class="pk-tab pkt-p2">2. 分割 (2台)</label>
      <label for="pk-p3" class="pk-tab pkt-p3">3. 分割 (3台)</label>
      <label for="pk-p4" class="pk-tab pkt-p4">4. 分割 (4台)</label>
    </div>
    <div class="pk-tl"><div class="pk-tl-bar"></div></div>
    <div class="pk-banner">
      <div class="pk-ph pkp-1"><strong>Step 1 (初期状態):</strong> どちらのパターンも最初は <code>Node 1</code> 上の1スプリットへ新規書き込みが集中します。</div>
      <div class="pk-ph pkp-2"><strong>Step 2 (2分割・Node 2追加):</strong> 単調増加キーは新しい末尾（<code>Node 2</code>）へ負荷が丸ごと移るだけですが、分散キーは2台へ均等分散します。</div>
      <div class="pk-ph pkp-3"><strong>Step 3 (3分割・Node 3追加):</strong> さらに分割してノードを増やしても、単調増加キーは末尾の <code>Node 3</code> だけが95%（Moving Hotspot）になります。</div>
      <div class="pk-ph pkp-4"><strong>Step 4 (4分割・Node 4追加):</strong> 何度分割しノードを4台に増やしても、単調増加キーは最後の1台に負荷が集中し続けます。一方、分散キーは全4台へ24%ずつ分散します。</div>
    </div>
  </div>

  <div class="pk-grid">
    <!-- 左パネル: 単調増加キー (Moving Hotspot) -->
    <div class="pk-panel panel-bad">
      <div class="pk-phead">
        <span class="pk-badge b-bad">アンチパターン</span>
        <span class="pk-ptitle">単調増加キー（連番 / タイムスタンプ先頭）</span>
      </div>
      <div class="pk-khead">
        <span>ノード / CPU</span>
        <div class="pk-kcols">
          <span><code>[1,1k)</code></span>
          <span><code>[1k,2k)</code></span>
          <span><code>[2k,3k)</code></span>
          <span><code>[3k,∞)</code></span>
        </div>
      </div>

      <div class="pk-lanes">
        <div class="pk-row row-bad-1">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 1</span>
              <span class="pk-state">
                <span class="pk-st pkp-1 b-hot">CPU 95%</span>
                <span class="pk-st pkp-2 b-idle">CPU 5%</span>
                <span class="pk-st pkp-3 b-idle">CPU 5%</span>
                <span class="pk-st pkp-4 b-idle">CPU 5%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-bad-1"></div></div>
          </div>
          <div class="pk-track">
            <div class="pk-blk blk-bad-1">
              <span class="pk-state">
                <span class="pk-st pkp-1 blk-lbl">100%</span>
                <span class="pk-st pkp-2 blk-lbl">0%</span>
                <span class="pk-st pkp-3 blk-lbl">0%</span>
                <span class="pk-st pkp-4 blk-lbl">0%</span>
              </span>
            </div>
            <div></div><div></div><div></div>
          </div>
        </div>

        <div class="pk-row row-bad-2">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 2</span>
              <span class="pk-state">
                <span class="pk-st pkp-2 b-hot">CPU 95%</span>
                <span class="pk-st pkp-3 b-idle">CPU 5%</span>
                <span class="pk-st pkp-4 b-idle">CPU 5%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-bad-2"></div></div>
          </div>
          <div class="pk-track">
            <div></div>
            <div class="pk-blk blk-bad-2">
              <span class="pk-state">
                <span class="pk-st pkp-2 blk-lbl">100%</span>
                <span class="pk-st pkp-3 blk-lbl">0%</span>
                <span class="pk-st pkp-4 blk-lbl">0%</span>
              </span>
            </div>
            <div></div><div></div>
          </div>
        </div>

        <div class="pk-row row-bad-3">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 3</span>
              <span class="pk-state">
                <span class="pk-st pkp-3 b-hot">CPU 95%</span>
                <span class="pk-st pkp-4 b-idle">CPU 5%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-bad-3"></div></div>
          </div>
          <div class="pk-track">
            <div></div><div></div>
            <div class="pk-blk blk-bad-3">
              <span class="pk-state">
                <span class="pk-st pkp-3 blk-lbl">100%</span>
                <span class="pk-st pkp-4 blk-lbl">0%</span>
              </span>
            </div>
            <div></div>
          </div>
        </div>

        <div class="pk-row row-bad-4">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 4</span>
              <span class="b-hot">CPU 95%</span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-bad-4"></div></div>
          </div>
          <div class="pk-track">
            <div></div><div></div><div></div>
            <div class="pk-blk blk-bad-4">
              <span class="blk-lbl">100%</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- 右パネル: 分散キー (均等分散) -->
    <div class="pk-panel panel-good">
      <div class="pk-phead">
        <span class="pk-badge b-good">推奨パターン</span>
        <span class="pk-ptitle">分散キー（UUID v4 / ビット反転 / ハッシュ）</span>
      </div>
      <div class="pk-khead">
        <span>ノード / CPU</span>
        <div class="pk-kcols">
          <span><code>00..</code></span>
          <span><code>40..</code></span>
          <span><code>80..</code></span>
          <span><code>c0..</code></span>
        </div>
      </div>

      <div class="pk-lanes">
        <div class="pk-row row-good-1">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 1</span>
              <span class="pk-state">
                <span class="pk-st pkp-1 b-hot">CPU 95%</span>
                <span class="pk-st pkp-2 b-ok">CPU 48%</span>
                <span class="pk-st pkp-3 b-ok">CPU 32%</span>
                <span class="pk-st pkp-4 b-ok">CPU 24%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-good-1"></div></div>
          </div>
          <div class="pk-track">
            <div class="pk-blk blk-good-1">
              <span class="pk-state">
                <span class="pk-st pkp-1 blk-lbl">100%</span>
                <span class="pk-st pkp-2 blk-lbl">50%</span>
                <span class="pk-st pkp-3 blk-lbl">33%</span>
                <span class="pk-st pkp-4 blk-lbl">25%</span>
              </span>
            </div>
            <div></div><div></div><div></div>
          </div>
        </div>

        <div class="pk-row row-good-2">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 2</span>
              <span class="pk-state">
                <span class="pk-st pkp-2 b-ok">CPU 48%</span>
                <span class="pk-st pkp-3 b-ok">CPU 32%</span>
                <span class="pk-st pkp-4 b-ok">CPU 24%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-good-2"></div></div>
          </div>
          <div class="pk-track">
            <div></div>
            <div class="pk-blk blk-good-2">
              <span class="pk-state">
                <span class="pk-st pkp-2 blk-lbl">50%</span>
                <span class="pk-st pkp-3 blk-lbl">33%</span>
                <span class="pk-st pkp-4 blk-lbl">25%</span>
              </span>
            </div>
            <div></div><div></div>
          </div>
        </div>

        <div class="pk-row row-good-3">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 3</span>
              <span class="pk-state">
                <span class="pk-st pkp-3 b-ok">CPU 32%</span>
                <span class="pk-st pkp-4 b-ok">CPU 24%</span>
              </span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-good-3"></div></div>
          </div>
          <div class="pk-track">
            <div></div><div></div>
            <div class="pk-blk blk-good-3">
              <span class="pk-state">
                <span class="pk-st pkp-3 blk-lbl">33%</span>
                <span class="pk-st pkp-4 blk-lbl">25%</span>
              </span>
            </div>
            <div></div>
          </div>
        </div>

        <div class="pk-row row-good-4">
          <div class="pk-ninfo">
            <div class="pk-ntop">
              <span class="pk-nname">Node 4</span>
              <span class="b-ok">CPU 24%</span>
            </div>
            <div class="pk-meter"><div class="pk-fill f-good-4"></div></div>
          </div>
          <div class="pk-track">
            <div></div><div></div><div></div>
            <div class="pk-blk blk-good-4">
              <span class="blk-lbl">25%</span>
            </div>
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
    --meter-bg: #e2e8f0;
    --primary: #0284c7; --primary-bg: #e0f2fe;
    --green: #15803d; --green-bg: #dcfce7;
    --red: #b91c1c; --red-bg: #fee2e2;
    --pcycle: 20s;
    --pk-rh: 44px;
    --pk-gap: 4px;
    box-sizing: border-box; padding: 10px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.3;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --meter-bg: #0f172a;
    --primary: #38bdf8; --primary-bg: #0c4a6e;
    --green: #4ade80; --green-bg: #064e3b;
    --red: #f87171; --red-bg: #450a0a;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }
  .pk-radio { display: none; }

  .pk-toolbar {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px;
    padding: 7px 10px; margin-bottom: 8px;
  }
  .pk-tabs { display: flex; flex-wrap: wrap; gap: 4px; margin-bottom: 5px; }
  .pk-tab {
    padding: 2px 7px; border-radius: 4px; font-size: 12px; font-weight: 600;
    background: var(--surface-sub); color: var(--text-muted); border: 1px solid var(--border);
    cursor: pointer; user-select: none;
  }
  #pk-auto:checked ~ .pk-toolbar .pkt-auto,
  #pk-p1:checked ~ .pk-toolbar .pkt-p1,
  #pk-p2:checked ~ .pk-toolbar .pkt-p2,
  #pk-p3:checked ~ .pk-toolbar .pkt-p3,
  #pk-p4:checked ~ .pk-toolbar .pkt-p4 {
    background: var(--primary-bg); color: var(--primary); border-color: var(--primary); font-weight: 700;
  }
  .pk-tl { width: 100%; height: 4px; background: var(--surface-sub); border-radius: 2px; overflow: hidden; margin-bottom: 5px; }
  .pk-tl-bar { height: 100%; background: var(--primary); width: 100%; }
  #pk-auto:checked ~ .pk-toolbar .pk-tl-bar { animation: pktl var(--pcycle) infinite linear; }
  #pk-p1:checked ~ .pk-toolbar .pk-tl-bar { width: 25%; }
  #pk-p2:checked ~ .pk-toolbar .pk-tl-bar { width: 50%; }
  #pk-p3:checked ~ .pk-toolbar .pk-tl-bar { width: 75%; }
  #pk-p4:checked ~ .pk-toolbar .pk-tl-bar { width: 100%; }

  .pk-banner, .pk-state { display: grid; }
  .pk-ph, .pk-st { grid-area: 1 / 1; opacity: 0; pointer-events: none; transition: opacity 0.25s ease; }

  #pk-auto:checked ~ * .pkp-1 { animation: pkvis-1 var(--pcycle) infinite; }
  #pk-auto:checked ~ * .pkp-2 { animation: pkvis-2 var(--pcycle) infinite; }
  #pk-auto:checked ~ * .pkp-3 { animation: pkvis-3 var(--pcycle) infinite; }
  #pk-auto:checked ~ * .pkp-4 { animation: pkvis-4 var(--pcycle) infinite; }
  #pk-p1:checked ~ * .pkp-1,
  #pk-p2:checked ~ * .pkp-2,
  #pk-p3:checked ~ * .pkp-3,
  #pk-p4:checked ~ * .pkp-4 { opacity: 1; animation: none !important; }

  .pk-grid { display: grid; grid-template-columns: 1fr 1fr; gap: 8px; }
  .pk-panel {
    background: var(--surface); border: 1.5px solid var(--border); border-radius: 8px; padding: 8px;
  }
  .panel-bad { border-color: var(--red); }
  .panel-good { border-color: var(--green); }
  .pk-phead { display: flex; align-items: center; gap: 5px; margin-bottom: 5px; flex-wrap: wrap; }
  .pk-badge { padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; }
  .b-bad { background: var(--red-bg); color: var(--red); }
  .b-good { background: var(--green-bg); color: var(--green); }
  .pk-ptitle { font-weight: 700; font-size: 12px; }

  .pk-khead {
    display: grid; grid-template-columns: 104px 1fr; gap: 6px;
    font-size: 12px; font-weight: 700; color: var(--text-muted);
    padding-bottom: 4px; margin-bottom: 4px; border-bottom: 1px solid var(--border);
  }
  .pk-kcols { display: grid; grid-template-columns: repeat(4, 1fr); gap: 3px; text-align: center; }

  .pk-lanes { display: flex; flex-direction: column; gap: var(--pk-gap); }
  .pk-row {
    display: grid; grid-template-columns: 104px 1fr; gap: 6px; align-items: center;
    height: var(--pk-rh); padding: 3px 6px; border-radius: 6px;
    background: var(--surface-sub); border: 1px solid var(--border); overflow: hidden;
  }
  .pk-ninfo { display: flex; flex-direction: column; gap: 3px; }
  .pk-ntop { display: flex; justify-content: space-between; align-items: center; gap: 3px; }
  .pk-nname { font-weight: 700; font-size: 12px; white-space: nowrap; }
  .b-hot, .b-ok, .b-idle {
    display: inline-block; padding: 1px 4px; border-radius: 3px; font-size: 12px; font-weight: 700; white-space: nowrap;
  }
  .b-hot { background: var(--red-bg); color: var(--red); }
  .b-ok { background: var(--green-bg); color: var(--green); }
  .b-idle { background: var(--surface); color: var(--text-muted); }

  .pk-meter { width: 100%; height: 7px; background: var(--meter-bg); border-radius: 4px; overflow: hidden; border: 1px solid var(--border); }
  .pk-fill { height: 100%; border-radius: 3px; transition: width 0.4s ease, background 0.4s ease; }

  .pk-track { display: grid; grid-template-columns: repeat(4, 1fr); gap: 3px; height: 32px; align-items: stretch; }
  .pk-blk {
    border-radius: 4px; display: flex; flex-direction: column; align-items: center; justify-content: center;
    font-size: 12px; font-weight: 700; padding: 2px; text-align: center; white-space: nowrap;
    border: 1px solid var(--border); background: var(--surface);
  }
  .blk-lbl { font-size: 12px; font-weight: 700; line-height: 1.05; text-align: center; }

  /* 初期は Node 2〜4 を非表示にし、分割ごとに展開 */
  .row-bad-2, .row-good-2, .row-bad-3, .row-good-3, .row-bad-4, .row-good-4 {
    height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap));
  }
  .f-bad-4 { width: 95%; background: var(--red); }
  .blk-bad-4 { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  .row-bad-4 { border-color: var(--red); }
  .f-good-4 { width: 24%; background: var(--green); }
  .blk-good-4 { background: var(--green-bg); border-color: var(--green); color: var(--green); }
  .row-good-4 { border-color: var(--green); }

  /* 自動再生 (20秒サイクル: P1 0-23%, P2 25-48%, P3 50-73%, P4 75-98%) */
  #pk-auto:checked ~ .pk-grid .row-bad-2,
  #pk-auto:checked ~ .pk-grid .row-good-2 { animation: pk-add-2 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .row-bad-3,
  #pk-auto:checked ~ .pk-grid .row-good-3 { animation: pk-add-3 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .row-bad-4,
  #pk-auto:checked ~ .pk-grid .row-good-4 { animation: pk-add-4 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-bad-1 { animation: f-bad-1 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-bad-2 { animation: f-bad-2 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-bad-3 { animation: f-bad-3 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .blk-bad-1 { animation: b-bad-1 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .blk-bad-2 { animation: b-bad-2 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .blk-bad-3 { animation: b-bad-3 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-good-1 { animation: f-good-1 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-good-2 { animation: f-good-2 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .f-good-3 { animation: f-good-3 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .blk-good-1 { animation: b-good-1 var(--pcycle) infinite; }
  #pk-auto:checked ~ .pk-grid .blk-good-2,
  #pk-auto:checked ~ .pk-grid .blk-good-3 { animation: b-good-ok var(--pcycle) infinite; }

  /* 手動ステップ固定 */
  #pk-p1:checked ~ .pk-grid .f-bad-1,
  #pk-p1:checked ~ .pk-grid .f-good-1 { width: 95%; background: var(--red); }
  #pk-p1:checked ~ .pk-grid .blk-bad-1,
  #pk-p1:checked ~ .pk-grid .blk-good-1 { background: var(--red-bg); border-color: var(--red); color: var(--red); }

  #pk-p2:checked ~ .pk-grid .row-bad-2,
  #pk-p2:checked ~ .pk-grid .row-good-2 { height: var(--pk-rh); opacity: 1; padding: 3px 6px; border-width: 1px; margin-top: 0; }
  #pk-p2:checked ~ .pk-grid .f-bad-1 { width: 5%; background: var(--text-muted); }
  #pk-p2:checked ~ .pk-grid .f-bad-2 { width: 95%; background: var(--red); }
  #pk-p2:checked ~ .pk-grid .blk-bad-2 { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  #pk-p2:checked ~ .pk-grid .f-good-1,
  #pk-p2:checked ~ .pk-grid .f-good-2 { width: 48%; background: var(--green); }
  #pk-p2:checked ~ .pk-grid .blk-good-1,
  #pk-p2:checked ~ .pk-grid .blk-good-2 { background: var(--green-bg); border-color: var(--green); color: var(--green); }

  #pk-p3:checked ~ .pk-grid .row-bad-2,
  #pk-p3:checked ~ .pk-grid .row-good-2,
  #pk-p3:checked ~ .pk-grid .row-bad-3,
  #pk-p3:checked ~ .pk-grid .row-good-3 { height: var(--pk-rh); opacity: 1; padding: 3px 6px; border-width: 1px; margin-top: 0; }
  #pk-p3:checked ~ .pk-grid .f-bad-1,
  #pk-p3:checked ~ .pk-grid .f-bad-2 { width: 5%; background: var(--text-muted); }
  #pk-p3:checked ~ .pk-grid .f-bad-3 { width: 95%; background: var(--red); }
  #pk-p3:checked ~ .pk-grid .blk-bad-3 { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  #pk-p3:checked ~ .pk-grid .f-good-1,
  #pk-p3:checked ~ .pk-grid .f-good-2,
  #pk-p3:checked ~ .pk-grid .f-good-3 { width: 32%; background: var(--green); }
  #pk-p3:checked ~ .pk-grid .blk-good-1,
  #pk-p3:checked ~ .pk-grid .blk-good-2,
  #pk-p3:checked ~ .pk-grid .blk-good-3 { background: var(--green-bg); border-color: var(--green); color: var(--green); }

  #pk-p4:checked ~ .pk-grid .row-bad-2,
  #pk-p4:checked ~ .pk-grid .row-good-2,
  #pk-p4:checked ~ .pk-grid .row-bad-3,
  #pk-p4:checked ~ .pk-grid .row-good-3,
  #pk-p4:checked ~ .pk-grid .row-bad-4,
  #pk-p4:checked ~ .pk-grid .row-good-4 { height: var(--pk-rh); opacity: 1; padding: 3px 6px; border-width: 1px; margin-top: 0; }
  #pk-p4:checked ~ .pk-grid .f-bad-1,
  #pk-p4:checked ~ .pk-grid .f-bad-2,
  #pk-p4:checked ~ .pk-grid .f-bad-3 { width: 5%; background: var(--text-muted); }
  #pk-p4:checked ~ .pk-grid .f-good-1,
  #pk-p4:checked ~ .pk-grid .f-good-2,
  #pk-p4:checked ~ .pk-grid .f-good-3 { width: 24%; background: var(--green); }
  #pk-p4:checked ~ .pk-grid .blk-good-1,
  #pk-p4:checked ~ .pk-grid .blk-good-2,
  #pk-p4:checked ~ .pk-grid .blk-good-3 { background: var(--green-bg); border-color: var(--green); color: var(--green); }

  @keyframes pktl { 0% { width: 0%; } 25% { width: 25%; } 50% { width: 50%; } 75% { width: 75%; } 100% { width: 100%; } }
  @keyframes pkvis-1 { 0%, 23% { opacity: 1; } 25%, 100% { opacity: 0; } }
  @keyframes pkvis-2 { 0%, 24% { opacity: 0; } 25%, 48% { opacity: 1; } 50%, 100% { opacity: 0; } }
  @keyframes pkvis-3 { 0%, 49% { opacity: 0; } 50%, 73% { opacity: 1; } 75%, 100% { opacity: 0; } }
  @keyframes pkvis-4 { 0%, 74% { opacity: 0; } 75%, 98% { opacity: 1; } 100% { opacity: 0; } }

  @keyframes pk-add-2 {
    0%, 23% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
    27%, 98% { height: var(--pk-rh); opacity: 1; padding-top: 3px; padding-bottom: 3px; border-width: 1px; margin-top: 0; }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
  }
  @keyframes pk-add-3 {
    0%, 48% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
    52%, 98% { height: var(--pk-rh); opacity: 1; padding-top: 3px; padding-bottom: 3px; border-width: 1px; margin-top: 0; }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
  }
  @keyframes pk-add-4 {
    0%, 73% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
    77%, 98% { height: var(--pk-rh); opacity: 1; padding-top: 3px; padding-bottom: 3px; border-width: 1px; margin-top: 0; }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--pk-gap)); }
  }

  @keyframes f-bad-1 { 0%, 23% { width: 95%; background: var(--red); } 27%, 98% { width: 5%; background: var(--text-muted); } 100% { width: 95%; background: var(--red); } }
  @keyframes f-bad-2 { 0%, 48% { width: 95%; background: var(--red); } 52%, 98% { width: 5%; background: var(--text-muted); } 100% { width: 95%; background: var(--red); } }
  @keyframes f-bad-3 { 0%, 73% { width: 95%; background: var(--red); } 77%, 98% { width: 5%; background: var(--text-muted); } 100% { width: 95%; background: var(--red); } }

  @keyframes b-bad-1 {
    0%, 23% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
    27%, 98% { background: var(--surface); border-color: var(--border); color: var(--text-muted); }
    100% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  }
  @keyframes b-bad-2 {
    0%, 48% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
    52%, 98% { background: var(--surface); border-color: var(--border); color: var(--text-muted); }
    100% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  }
  @keyframes b-bad-3 {
    0%, 73% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
    77%, 98% { background: var(--surface); border-color: var(--border); color: var(--text-muted); }
    100% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  }

  @keyframes f-good-1 {
    0%, 23% { width: 95%; background: var(--red); }
    27%, 48% { width: 48%; background: var(--green); }
    52%, 73% { width: 32%; background: var(--green); }
    77%, 98% { width: 24%; background: var(--green); }
    100% { width: 95%; background: var(--red); }
  }
  @keyframes f-good-2 {
    0%, 48% { width: 48%; background: var(--green); }
    52%, 73% { width: 32%; background: var(--green); }
    77%, 98% { width: 24%; background: var(--green); }
    100% { width: 48%; background: var(--green); }
  }
  @keyframes f-good-3 {
    0%, 73% { width: 32%; background: var(--green); }
    77%, 98% { width: 24%; background: var(--green); }
    100% { width: 32%; background: var(--green); }
  }
  @keyframes b-good-1 {
    0%, 23% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
    27%, 98% { background: var(--green-bg); border-color: var(--green); color: var(--green); }
    100% { background: var(--red-bg); border-color: var(--red); color: var(--red); }
  }
  @keyframes b-good-ok {
    0%, 98% { background: var(--green-bg); border-color: var(--green); color: var(--green); }
  }

  @media (max-width: 640px) {
    .diagram { padding: 8px; --pk-rh: 32px; --pk-gap: 3px; }
    .pk-toolbar { padding: 6px 8px; margin-bottom: 6px; }
    .pk-grid { grid-template-columns: 1fr; gap: 6px; }
    .pk-panel { padding: 6px 8px; }
    .pk-khead { display: none; }
    .pk-row { grid-template-columns: 112px 1fr; padding: 2px 5px; }
    .pk-track { height: 24px; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after { animation: none !important; transition: none !important; }
    #pk-auto:checked ~ * .pkp-4 { opacity: 1; }
    #pk-auto:checked ~ .pk-grid .row-bad-2,
    #pk-auto:checked ~ .pk-grid .row-good-2,
    #pk-auto:checked ~ .pk-grid .row-bad-3,
    #pk-auto:checked ~ .pk-grid .row-good-3,
    #pk-auto:checked ~ .pk-grid .row-bad-4,
    #pk-auto:checked ~ .pk-grid .row-good-4 { height: var(--pk-rh); opacity: 1; padding: 3px 6px; border-width: 1px; margin-top: 0; }
    #pk-auto:checked ~ .pk-grid .f-bad-1,
    #pk-auto:checked ~ .pk-grid .f-bad-2,
    #pk-auto:checked ~ .pk-grid .f-bad-3 { width: 5%; background: var(--text-muted); }
    #pk-auto:checked ~ .pk-grid .f-good-1,
    #pk-auto:checked ~ .pk-grid .f-good-2,
    #pk-auto:checked ~ .pk-grid .f-good-3 { width: 24%; background: var(--green); }
    #pk-auto:checked ~ .pk-grid .blk-good-1,
    #pk-auto:checked ~ .pk-grid .blk-good-2,
    #pk-auto:checked ~ .pk-grid .blk-good-3 { background: var(--green-bg); border-color: var(--green); color: var(--green); }
  }
</style>
```

たとえば、`1, 2, 3...`と増え続ける連番や現在時刻をプライマリキーの先頭に置いたとします。すると、新しく`INSERT`される行は常にキー空間の一番末尾にあるスプリットに書き込まれます。

Spannerが末尾のスプリットの負荷上昇を検知し、`Key: 1000`、`2000`、`3000`とスプリットを分割したとしても、その直後にやってくる新規の書き込みはすべて分割後の最大値側（図の`[3k, ∞)`）のスプリットへ入ります。つまり、せっかくスプリットを分割しても過去のキー範囲（`[1, 1000)`や`[1000, 2000)`）には二度と新規の書き込みが来ず、常に「その時点で一番末尾を担当している1台のノード」だけがすべての書き込みを受け止め続けることになります[^4][^5]。これはセカンダリインデックスでも同様であり、更新頻度の高いタイムスタンプ列を先頭にした通常のインデックスを作成すると、インデックス側のスプリット末尾で同じホットスポットが発生します[^5]。

これを防ぐため、Spannerでは新しい行のプライマリキーがキー空間全体に散らばるように設計します[^5]。図の右側（推奨パターン）のように、生成されるキーの先頭部分がばらければ、分割されたすべてのスプリット（Node 1〜4）へ同時に書き込みが振り分けられ、ノード台数を増やした分だけ書き込み性能を引き出すことができます。

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
  <div class="il-layout">
    <!-- Split 1 (Node 1): 複数の Singers とその Albums が同居 -->
    <div class="split-box box-n1">
      <div class="split-head">
        <span class="split-label lbl-n1">Split 1 (Node 1)</span>
        <span class="split-range"><code>SingerId: [1, 5000)</code></span>
      </div>
      <div class="parent-list">
        <div class="parent-row">
          <div class="row-line">
            <span class="row-tag tag-parent">親: Singers</span>
            <span class="row-data"><code>1024</code> ("Alice")</span>
          </div>
          <div class="child-group">
            <div class="child-row"><span class="row-tag tag-child">子: Albums</span><code>1024, 1</code> / <code>1024, 2</code></div>
          </div>
        </div>

        <div class="parent-row">
          <div class="row-line">
            <span class="row-tag tag-parent">親: Singers</span>
            <span class="row-data"><code>2500</code> ("Carol")</span>
          </div>
          <div class="child-group">
            <div class="child-row"><span class="row-tag tag-child">子: Albums</span><code>2500, 1</code></div>
          </div>
        </div>

        <div class="more-rows">... 複数の Singers と子行が同居 ...</div>
      </div>
    </div>

    <!-- スプリット境界 -->
    <div class="split-sep">
      <div class="sep-line"></div>
      <div class="sep-badge">スプリット境界<br><code>SingerId = 5000</code><br>（親行の間で分割）</div>
      <div class="sep-line"></div>
    </div>

    <!-- Split 2 (Node 2): 複数の Singers とその Albums が同居 -->
    <div class="split-box box-n2">
      <div class="split-head">
        <span class="split-label lbl-n2">Split 2 (Node 2)</span>
        <span class="split-range"><code>SingerId: [5000, ∞)</code></span>
      </div>
      <div class="parent-list">
        <div class="parent-row">
          <div class="row-line">
            <span class="row-tag tag-parent">親: Singers</span>
            <span class="row-data"><code>6400</code> ("Bob")</span>
          </div>
          <div class="child-group">
            <div class="child-row"><span class="row-tag tag-child">子: Albums</span><code>6400, 1</code> / <code>6400, 2</code></div>
          </div>
        </div>

        <div class="parent-row">
          <div class="row-line">
            <span class="row-tag tag-parent">親: Singers</span>
            <span class="row-data"><code>8200</code> ("Dave")</span>
          </div>
          <div class="child-group">
            <div class="child-row"><span class="row-tag tag-child">子: Albums</span><code>8200, 1</code></div>
          </div>
        </div>

        <div class="more-rows">... 複数の Singers と子行が同居 ...</div>
      </div>
    </div>
  </div>
</div>

<style>
  .diagram {
    --bg: #f8fafc; --surface: #ffffff; --surface-sub: #f1f5f9;
    --text: #0f172a; --text-muted: #475569; --border: #cbd5e1;
    --n1: #0284c7; --n1-bg: #e0f2fe;
    --n2: #7c3aed; --n2-bg: #ede9fe;
    --green: #15803d; --green-bg: #dcfce7;
    --amber: #b45309; --amber-bg: #fef3c7;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.35;
  }
  [data-theme="dark"] .diagram {
    --bg: #0f172a; --surface: #1e293b; --surface-sub: #162032;
    --text: #f1f5f9; --text-muted: #94a3b8; --border: #334155;
    --n1: #38bdf8; --n1-bg: #0c4a6e;
    --n2: #a78bfa; --n2-bg: #3b0764;
    --green: #4ade80; --green-bg: #064e3b;
    --amber: #fbbf24; --amber-bg: #451a03;
  }
  .diagram *, .diagram *::before, .diagram *::after { box-sizing: border-box; }

  .il-layout { display: grid; grid-template-columns: 1fr auto 1fr; gap: 10px; align-items: stretch; }
  .split-box { background: var(--surface); border: 1.5px solid var(--border); border-radius: 8px; padding: 8px 10px; }
  .box-n1 { border-color: var(--n1); }
  .box-n2 { border-color: var(--n2); }
  .split-head {
    display: flex; justify-content: space-between; align-items: center; flex-wrap: wrap; gap: 4px;
    padding-bottom: 5px; margin-bottom: 6px; border-bottom: 1px solid var(--border); font-size: 12px;
  }
  .split-label { font-weight: 700; }
  .lbl-n1 { color: var(--n1); }
  .lbl-n2 { color: var(--n2); }
  .split-range { color: var(--text-muted); font-size: 12px; }

  .parent-list { display: flex; flex-direction: column; gap: 5px; }
  .parent-row {
    background: var(--surface-sub); border: 1px solid var(--border); border-radius: 6px; padding: 5px 7px;
  }
  .row-line { display: flex; align-items: center; gap: 6px; flex-wrap: wrap; margin-bottom: 4px; }
  .row-data { font-size: 12px; font-weight: 600; }
  .child-group { display: flex; flex-direction: column; gap: 3px; padding-left: 8px; border-left: 2px solid var(--green); }
  .child-row {
    background: var(--surface); border: 1px solid var(--border); border-radius: 4px;
    padding: 3px 6px; font-size: 12px; display: flex; align-items: center; flex-wrap: wrap; gap: 6px;
  }
  .row-tag { display: inline-block; padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; }
  .tag-parent { background: var(--n1-bg); color: var(--n1); }
  .box-n2 .tag-parent { background: var(--n2-bg); color: var(--n2); }
  .tag-child { background: var(--green-bg); color: var(--green); }
  .more-rows { text-align: center; font-size: 12px; color: var(--text-muted); padding: 2px 0; }

  .split-sep { display: flex; flex-direction: column; align-items: center; justify-content: center; gap: 6px; }
  .sep-line { flex: 1; width: 0; border-left: 2px dashed var(--amber); }
  .sep-badge {
    background: var(--amber-bg); color: var(--amber); border: 1px solid var(--amber);
    border-radius: 6px; padding: 4px 6px; text-align: center; font-size: 12px; font-weight: 700; line-height: 1.25;
  }

  @media (max-width: 640px) {
    .diagram { padding: 8px; }
    .il-layout { grid-template-columns: 1fr; gap: 6px; }
    .split-box { padding: 6px 8px; }
    .parent-list { gap: 4px; }
    .parent-row { padding: 4px 6px; }
    .split-sep { flex-direction: row; gap: 6px; }
    .sep-line { width: auto; height: 0; border-left: none; border-top: 1.5px dashed var(--amber); }
    .sep-badge br { display: none; }
    .sep-badge { padding: 2px 6px; }
    .more-rows { display: none; }
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
