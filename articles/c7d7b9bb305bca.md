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
        <div class="arrow-down" aria-hidden="true">
          <span class="flow-track"><span class="flow-dot dot-muted"></span></span>
          <span>↓ 手動振り分け</span>
        </div>
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
        <div class="arrow-down arrow-primary" aria-hidden="true">
          <span class="flow-track"><span class="flow-dot dot-primary"></span></span>
          <span>↓ Spanner API 自動ルーティング</span>
        </div>
        <div class="server-grid-3">
          <div class="srv-box srv-active srv-anim-1">
            <div class="srv-name">Server A</div>
            <div class="range-pill pill-primary">Split 1<br><code>[1, 3500)</code></div>
          </div>
          <div class="srv-box srv-active srv-anim-2">
            <div class="srv-name">Server B</div>
            <div class="range-pill pill-primary">Split 2<br><code>[3500, 7200)</code></div>
          </div>
          <div class="srv-box srv-active srv-anim-3">
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
  .dot-muted { background: var(--text-muted); }
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

負荷の集中を検知してからスプリット境界を設定して分割し、既存ノード間での負荷分散、さらにノード追加（スケールアウト）による負荷分散へと推移する一連の流れを1つの図解で見てみます（デフォルトで自動再生され、上部のボタンで各フェーズに固定することもできます）。

```html-embed
<div class="diagram">
  <input type="radio" name="sp-sim" id="sim-auto" class="sim-radio" checked>
  <input type="radio" name="sp-sim" id="sim-p1" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p2" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p3" class="sim-radio">
  <input type="radio" name="sp-sim" id="sim-p4" class="sim-radio">

  <div class="sim-toolbar">
    <div class="sim-tabs">
      <label for="sim-auto" class="sim-tab tab-auto">自動再生 (20秒)</label>
      <label for="sim-p1" class="sim-tab tab-p1">1. 初期状態</label>
      <label for="sim-p2" class="sim-tab tab-p2">2. 負荷集中・分割</label>
      <label for="sim-p3" class="sim-tab tab-p3">3. Node 2へ移動</label>
      <label for="sim-p4" class="sim-tab tab-p4">4. Node 3追加・移動</label>
    </div>
    <div class="tl-bar"><div class="tl-progress"></div></div>
    <div class="phase-banner">
      <div class="ph-text ph-1"><strong>Phase 1 (初期状態):</strong> 境界のない1つの <code>Split 1</code> に全キーがあり、<code>Node 1</code> だけで処理しています。</div>
      <div class="ph-text ph-2"><strong>Phase 2 (境界設定・分割):</strong> <code>[400, 600)</code> への集中で <code>Node 1</code> が高負荷になり、境界が入って4つのスプリットに割れます。</div>
      <div class="ph-text ph-3"><strong>Phase 3 (スプリット移動):</strong> 割れた <code>1b</code> が真下の <code>Node 2</code> レーンへ移動し、負荷が2台に分かれます。</div>
      <div class="ph-text ph-4"><strong>Phase 4 (ノード追加・再分散):</strong> 新たに <code>Node 3</code> が追加され、<code>1c</code> が <code>Node 3</code> へ移動して3台へ均等分散されます。</div>
    </div>
  </div>

  <!-- 2次元マトリクス: 横軸=キー空間 / 縦軸=ノード行（スプリットが同じ列のまま縦の別ノードへ移動） -->
  <div class="matrix-wrap">
    <div class="matrix-head">
      <div class="mh-node">ノード / CPU負荷</div>
      <div class="mh-keys">
        <span><code>[1, 400)</code></span>
        <span><code>[400, 500)</code></span>
        <span><code>[500, 600)</code></span>
        <span><code>[600, 1000)</code></span>
      </div>
    </div>

    <div class="lanes">
      <!-- Node 1 行 -->
      <div class="lane-row lane-1">
        <div class="lane-info">
          <div class="li-top">
            <span class="li-name">Node 1</span>
            <span class="state-layer">
              <span class="st-item ph-1 badge-n1">25%</span>
              <span class="st-item ph-2 badge-hot">95% Hot</span>
              <span class="st-item ph-3 badge-warn">58%</span>
              <span class="st-item ph-4 badge-ok">32%</span>
            </span>
          </div>
          <div class="cpu-bar"><div class="cpu-fill fill-n1"></div></div>
        </div>

        <div class="lane-track">
          <!-- Phase 1 のシームレスな単一 Split 1 表示 -->
          <div class="unified-bar">Split 1: <code>[1, 1000)</code>（境界なし・単一スプリット）</div>

          <!-- 分割された4つのスプリット（1b と 1c が物理的に下のノード行へ移動） -->
          <div class="split-slots">
            <div class="sp-block sp-1a">
              <span class="sp-id">1a</span>
              <span class="sp-sub"><code>[1,400)</code></span>
            </div>
            <div class="sp-block sp-1b">
              <span class="sp-id">1b</span>
              <span class="sp-sub state-layer">
                <span class="st-item ph-2">高負荷</span>
                <span class="st-item ph-3">Node 2へ</span>
                <span class="st-item ph-4">Node 2</span>
              </span>
            </div>
            <div class="sp-block sp-1c">
              <span class="sp-id">1c</span>
              <span class="sp-sub state-layer">
                <span class="st-item ph-2">高負荷</span>
                <span class="st-item ph-3">残存</span>
                <span class="st-item ph-4">Node 3へ</span>
              </span>
            </div>
            <div class="sp-block sp-1d">
              <span class="sp-id">1d</span>
              <span class="sp-sub"><code>[600,1000)</code></span>
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
              <span class="st-item ph-1 badge-muted">10%</span>
              <span class="st-item ph-2 badge-muted">10%</span>
              <span class="st-item ph-3 badge-n2">52%</span>
              <span class="st-item ph-4 badge-ok">34%</span>
            </span>
          </div>
          <div class="cpu-bar"><div class="cpu-fill fill-n2"></div></div>
        </div>
        <div class="lane-track track-drop">
          <div class="drop-slot"></div>
          <div class="drop-slot slot-1b">
            <span class="state-layer">
              <span class="st-item ph-1 hint-txt">待機中</span>
              <span class="st-item ph-2 hint-txt">待機中</span>
            </span>
          </div>
          <div class="drop-slot"></div>
          <div class="drop-slot"></div>
        </div>
      </div>

      <!-- Node 3 行（Phase 4 で初めて出現） -->
      <div class="lane-row lane-3">
        <div class="lane-info">
          <div class="li-top">
            <span class="li-name">Node 3 (追加)</span>
            <span class="badge-ok">34%</span>
          </div>
          <div class="cpu-bar"><div class="cpu-fill fill-n3"></div></div>
        </div>
        <div class="lane-track track-drop">
          <div class="drop-slot"></div>
          <div class="drop-slot"></div>
          <div class="drop-slot slot-1c"></div>
          <div class="drop-slot"></div>
        </div>
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
    --n3: #15803d; --n3-bg: #dcfce7;
    --amber: #b45309; --amber-bg: #fef3c7;
    --red: #b91c1c; --red-bg: #fee2e2;
    --cycle: 20s;
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
  #sim-p4:checked ~ .sim-toolbar .tab-p4 {
    background: var(--n1-bg); color: var(--n1); border-color: var(--n1); font-weight: 700;
  }
  .tl-bar { width: 100%; height: 4px; background: var(--surface-sub); border-radius: 2px; overflow: hidden; margin-bottom: 6px; }
  .tl-progress { height: 100%; background: var(--n1); width: 100%; }
  #sim-auto:checked ~ .sim-toolbar .tl-progress { animation: tl-run var(--cycle) infinite linear; }
  #sim-p1:checked ~ .sim-toolbar .tl-progress { width: 25%; }
  #sim-p2:checked ~ .sim-toolbar .tl-progress { width: 50%; background: var(--red); }
  #sim-p3:checked ~ .sim-toolbar .tl-progress { width: 75%; background: var(--n2); }
  #sim-p4:checked ~ .sim-toolbar .tl-progress { width: 100%; background: var(--n3); }

  .phase-banner, .state-layer { display: grid; }
  .ph-text, .st-item { grid-area: 1 / 1; opacity: 0; pointer-events: none; transition: opacity 0.25s ease; }

  #sim-auto:checked ~ * .ph-1 { animation: vis-p1 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-2 { animation: vis-p2 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-3 { animation: vis-p3 var(--cycle) infinite; }
  #sim-auto:checked ~ * .ph-4 { animation: vis-p4 var(--cycle) infinite; }
  #sim-p1:checked ~ * .ph-1,
  #sim-p2:checked ~ * .ph-2,
  #sim-p3:checked ~ * .ph-3,
  #sim-p4:checked ~ * .ph-4 { opacity: 1; animation: none !important; }

  /* マトリクス本体 */
  .matrix-wrap {
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px;
    padding: 8px 10px;
  }
  .matrix-head {
    display: grid; grid-template-columns: 145px 1fr; gap: 8px;
    padding-bottom: 6px; margin-bottom: 6px; border-bottom: 1px solid var(--border);
    font-size: 12px; font-weight: 700; color: var(--text-muted);
  }
  .mh-keys { display: grid; grid-template-columns: repeat(4, 1fr); gap: 6px; text-align: center; }

  .lanes { display: flex; flex-direction: column; gap: var(--row-gap); }
  .lane-row {
    display: grid; grid-template-columns: 145px 1fr; gap: 8px; align-items: center;
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
  .cpu-bar { width: 100%; height: 7px; background: var(--surface); border-radius: 4px; overflow: hidden; border: 1px solid var(--border); }
  .cpu-fill { height: 100%; border-radius: 3px; width: 25%; background: var(--n1); transition: width 0.5s ease, background 0.5s ease; }
  .fill-n2 { width: 10%; background: var(--text-muted); }
  .fill-n3 { width: 34%; background: var(--n3); }

  .badge-n1, .badge-n2, .badge-hot, .badge-warn, .badge-ok, .badge-muted {
    display: inline-block; padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; text-align: right; white-space: nowrap;
  }
  .badge-n1 { background: var(--n1-bg); color: var(--n1); }
  .badge-n2 { background: var(--n2-bg); color: var(--n2); }
  .badge-ok { background: var(--n3-bg); color: var(--n3); }
  .badge-hot { background: var(--red-bg); color: var(--red); }
  .badge-warn { background: var(--amber-bg); color: var(--amber); }
  .badge-muted { background: var(--surface); color: var(--text-muted); border: 1px solid var(--border); }

  .lane-track { position: relative; height: 48px; display: grid; }
  .unified-bar {
    grid-area: 1 / 1; z-index: 3;
    background: var(--n1-bg); border: 2px solid var(--n1); border-radius: 6px;
    display: flex; align-items: center; justify-content: center;
    font-weight: 700; font-size: 12px; color: var(--n1);
  }
  .split-slots, .track-drop {
    grid-area: 1 / 1; display: grid; grid-template-columns: repeat(4, 1fr); gap: 6px; height: 100%;
  }
  .sp-block {
    border-radius: 6px; display: flex; flex-direction: column; align-items: center; justify-content: center;
    background: var(--n1-bg); border: 1.5px solid var(--n1); color: var(--n1);
    font-weight: 700; position: relative; z-index: 5;
    transition: transform 0.9s cubic-bezier(0.22, 1, 0.36, 1), background 0.4s ease, border-color 0.4s ease, color 0.4s ease;
  }
  .sp-id { font-size: 12px; font-weight: 700; line-height: 1.1; }
  .sp-sub { font-size: 12px; font-weight: 600; line-height: 1.1; }

  .drop-slot {
    border-radius: 6px; border: 1px dashed var(--border); background: var(--surface);
    display: flex; align-items: center; justify-content: center;
  }
  .hint-txt { font-size: 12px; color: var(--text-muted); }

  /* 自動再生アニメーション (20秒サイクル: P1 0-20%, P2 20-45%, P3 45-72%, P4 72-100%) */
  #sim-auto:checked ~ .matrix-wrap .unified-bar { animation: unibar-vis var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .split-slots { animation: slots-vis var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .sp-1b { animation: move-1b var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .sp-1c { animation: move-1c var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-1 { animation: l1-box var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-2 { animation: l2-box var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .lane-3 { animation: l3-add var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .fill-n1 { animation: cpu-n1 var(--cycle) infinite; }
  #sim-auto:checked ~ .matrix-wrap .fill-n2 { animation: cpu-n2 var(--cycle) infinite; }

  /* 手動フェーズ固定 */
  #sim-p1:checked ~ .matrix-wrap .split-slots { opacity: 0; }
  #sim-p2:checked ~ .matrix-wrap .unified-bar,
  #sim-p3:checked ~ .matrix-wrap .unified-bar,
  #sim-p4:checked ~ .matrix-wrap .unified-bar { opacity: 0; pointer-events: none; }

  #sim-p2:checked ~ .matrix-wrap .sp-1b,
  #sim-p2:checked ~ .matrix-wrap .sp-1c { background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
  #sim-p2:checked ~ .matrix-wrap .fill-n1 { width: 95%; background: var(--red); }
  #sim-p2:checked ~ .matrix-wrap .lane-1 { border-color: var(--red); }

  #sim-p3:checked ~ .matrix-wrap .sp-1b {
    transform: translateY(var(--step-y));
    background: var(--n2-bg); border-color: var(--n2); color: var(--n2);
  }
  #sim-p3:checked ~ .matrix-wrap .sp-1c { background: var(--amber-bg); border: 1.5px dashed var(--amber); color: var(--amber); }
  #sim-p3:checked ~ .matrix-wrap .fill-n1 { width: 58%; background: var(--amber); }
  #sim-p3:checked ~ .matrix-wrap .fill-n2 { width: 52%; background: var(--n2); }
  #sim-p3:checked ~ .matrix-wrap .lane-1 { border-color: var(--amber); }
  #sim-p3:checked ~ .matrix-wrap .lane-2 { border-color: var(--n2); }

  #sim-p4:checked ~ .matrix-wrap .sp-1b {
    transform: translateY(var(--step-y));
    background: var(--n2-bg); border-color: var(--n2); color: var(--n2);
  }
  #sim-p4:checked ~ .matrix-wrap .sp-1c {
    transform: translateY(calc(2 * var(--step-y)));
    background: var(--n3-bg); border-color: var(--n3); color: var(--n3);
  }
  #sim-p4:checked ~ .matrix-wrap .fill-n1 { width: 32%; background: var(--n3); }
  #sim-p4:checked ~ .matrix-wrap .fill-n2 { width: 34%; background: var(--n2); }
  #sim-p4:checked ~ .matrix-wrap .lane-1 { border-color: var(--n3); }
  #sim-p4:checked ~ .matrix-wrap .lane-2 { border-color: var(--n2); }
  #sim-p4:checked ~ .matrix-wrap .lane-3 {
    height: var(--row-h); opacity: 1; padding: 6px 8px; border-width: 1.5px; margin-top: 0;
  }

  @keyframes tl-run {
    0% { width: 0%; background: var(--n1); }
    19% { width: 25%; background: var(--n1); }
    20% { width: 25%; background: var(--red); }
    44% { width: 50%; background: var(--red); }
    45% { width: 50%; background: var(--n2); }
    71% { width: 75%; background: var(--n2); }
    72% { width: 75%; background: var(--n3); }
    100% { width: 100%; background: var(--n3); }
  }
  @keyframes vis-p1 { 0%, 18% { opacity: 1; } 20%, 100% { opacity: 0; } }
  @keyframes vis-p2 { 0%, 19% { opacity: 0; } 20%, 43% { opacity: 1; } 45%, 100% { opacity: 0; } }
  @keyframes vis-p3 { 0%, 44% { opacity: 0; } 45%, 70% { opacity: 1; } 72%, 100% { opacity: 0; } }
  @keyframes vis-p4 { 0%, 71% { opacity: 0; } 72%, 97% { opacity: 1; } 100% { opacity: 0; } }

  @keyframes unibar-vis { 0%, 18% { opacity: 1; } 20%, 97% { opacity: 0; } 100% { opacity: 1; } }
  @keyframes slots-vis { 0%, 18% { opacity: 0; gap: 0px; } 20%, 97% { opacity: 1; gap: 6px; } 100% { opacity: 0; } }

  @keyframes move-1b {
    0%, 18% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    20%, 44% { transform: translateY(0); background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
    52%, 97% { transform: translateY(var(--step-y)); background: var(--n2-bg); border: 1.5px solid var(--n2); color: var(--n2); }
    100% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
  }
  @keyframes move-1c {
    0%, 18% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
    20%, 44% { transform: translateY(0); background: var(--red-bg); border: 1.5px dashed var(--red); color: var(--red); }
    46%, 73% { transform: translateY(0); background: var(--amber-bg); border: 1.5px dashed var(--amber); color: var(--amber); }
    81%, 97% { transform: translateY(calc(2 * var(--step-y))); background: var(--n3-bg); border: 1.5px solid var(--n3); color: var(--n3); }
    100% { transform: translateY(0); background: var(--n1-bg); border-color: var(--n1); color: var(--n1); }
  }
  @keyframes l1-box {
    0%, 18% { border-color: var(--n1); }
    20%, 44% { border-color: var(--red); }
    46%, 71% { border-color: var(--amber); }
    73%, 97% { border-color: var(--n3); }
    100% { border-color: var(--n1); }
  }
  @keyframes l2-box {
    0%, 46% { border-color: var(--border); }
    50%, 97% { border-color: var(--n2); }
    100% { border-color: var(--border); }
  }
  @keyframes l3-add {
    0%, 71% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--row-gap)); }
    76%, 97% { height: var(--row-h); opacity: 1; padding-top: 6px; padding-bottom: 6px; border-width: 1.5px; margin-top: 0; }
    100% { height: 0; opacity: 0; padding-top: 0; padding-bottom: 0; border-width: 0; margin-top: calc(-1 * var(--row-gap)); }
  }
  @keyframes cpu-n1 {
    0%, 18% { width: 25%; background: var(--n1); }
    20%, 44% { width: 95%; background: var(--red); }
    48%, 73% { width: 58%; background: var(--amber); }
    78%, 97% { width: 32%; background: var(--n3); }
    100% { width: 25%; background: var(--n1); }
  }
  @keyframes cpu-n2 {
    0%, 46% { width: 10%; background: var(--text-muted); }
    52%, 73% { width: 52%; background: var(--n2); }
    78%, 97% { width: 34%; background: var(--n2); }
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
    .cpu-bar { width: 70px; flex-shrink: 0; }
    .lane-track { height: 44px; }
    .split-slots, .track-drop { gap: 3px; }
    .sp-sub { display: none; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after { animation: none !important; transition: none !important; }
    #sim-auto:checked ~ * .ph-4 { opacity: 1; }
    #sim-auto:checked ~ .matrix-wrap .unified-bar { opacity: 0; }
    #sim-auto:checked ~ .matrix-wrap .split-slots { opacity: 1; }
    #sim-auto:checked ~ .matrix-wrap .lane-3 { height: var(--row-h); opacity: 1; padding: 6px 8px; border-width: 1.5px; margin-top: 0; }
    #sim-auto:checked ~ .matrix-wrap .sp-1b { transform: translateY(var(--step-y)); background: var(--n2-bg); border-color: var(--n2); color: var(--n2); }
    #sim-auto:checked ~ .matrix-wrap .sp-1c { transform: translateY(calc(2 * var(--step-y))); background: var(--n3-bg); border-color: var(--n3); color: var(--n3); }
  }
</style>
```

まずPhase 1（初期状態）では、`Key: [1, 1000)`を受け持つ1つのスプリット（Split 1）に全データが格納され、Node 1の1台で処理されています。このうち`[400, 600)`の範囲にアクセスが集中すると、Node 1のCPU使用率が急上昇します。Spanner内部では、単一サーバーのリソース制約に対してスプリットの負荷が50%を超えた状態を「Warm」、100%に達してリクエストのレイテンシに影響が出うる状態を「Hot（ホットスポット）」として検知します[^4]。

続いてPhase 2（境界設定）では、Spannerがバックグラウンドで負荷の高いキー範囲の前後や内部（`400`、`500`、`600`）に新しいスプリット境界を追加し、`Split 1a`〜`1d`へと細かく分割します[^1][^4]。このとき、最小で1行単位までスプリットの境界を狭めることができます[^1]。

Phase 3（分割と負荷分散）では、分割された`Split 1b`をインスタンス内の余裕があるNode 2へ移動させます。さらにトラフィックが増加してノードが追加（スケールアウト）されたPhase 4では、新しく追加されたNode 3へ`Split 1c`が再配置されます。このように、当初は1つのスプリット・1台のノードだけで受けていた集中アクセスが、スプリット分割と移動によって複数のノードで並列に処理されるようになり、段階的に各ノードの負荷が下がってスループットが向上していきます[^1]。反対に、アクセスが落ち着いて負荷の低い状態が続いたスプリット同士は、再び自動的に統合（マージ）されます[^3]。

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
      <div class="pk-desc">スプリットが分割されても、次の新規書き込みは常に「新しい末尾スプリット」へ移動（Moving Hotspot）し、過去のキー範囲は使われず常に1台に負荷が集中します。</div>
      <div class="split-list">
        <div class="sp-row mono-row-1">
          <div class="sp-top">
            <span class="sp-name">Split 1 (Node 1): <code>[1, 1000)</code></span>
            <span class="mono-state">
              <span class="ms-1 load-tag tag-red">書込 100% (ID: 1..999)</span>
              <span class="ms-2 load-tag tag-zero">分割後: 新規書込 0%</span>
              <span class="ms-3 load-tag tag-zero">分割後: 新規書込 0%</span>
            </span>
          </div>
          <div class="mini-bar"><div class="mini-fill mfill-1"></div></div>
        </div>

        <div class="sp-row mono-row-2">
          <div class="sp-top">
            <span class="sp-name">Split 2 (Node 2): <code>[1000, 2000)</code></span>
            <span class="mono-state">
              <span class="ms-1 load-tag tag-zero">（分割前）</span>
              <span class="ms-2 load-tag tag-red">書込 100% (ID: 1000..1999)</span>
              <span class="ms-3 load-tag tag-zero">分割後: 新規書込 0%</span>
            </span>
          </div>
          <div class="mini-bar"><div class="mini-fill mfill-2"></div></div>
        </div>

        <div class="sp-row mono-row-3">
          <div class="sp-top">
            <span class="sp-name">Split 3 (Node 3): <code>[2000, ∞)</code></span>
            <span class="mono-state">
              <span class="ms-1 load-tag tag-zero">（分割前）</span>
              <span class="ms-2 load-tag tag-zero">（分割前）</span>
              <span class="ms-3 load-tag tag-red">書込 100% (ID: 2000..)</span>
            </span>
          </div>
          <div class="mini-bar"><div class="mini-fill mfill-3"></div></div>
        </div>
      </div>
    </div>

    <div class="pk-card card-good">
      <div class="pk-head">
        <span class="pk-badge badge-green">推奨パターン</span>
        <span class="pk-title">分散キー（UUID v4 / ビット反転 / ハッシュ先頭）</span>
      </div>
      <div class="pk-desc">新しいキー値がキー空間全体に一様に散らばります。分割されたすべてのスプリット（全ノード）へ同時に書き込みが分散され、台数に応じてスケールします。</div>
      <div class="split-list">
        <div class="sp-row sp-ok sp-even-1">
          <div class="sp-top">
            <span class="sp-name">Split 1 (Node 1): <code>[0000.., 5555..)</code></span>
            <span class="load-tag tag-green">書込 約33% (<code>2a9f..</code>)</span>
          </div>
          <div class="mini-bar"><div class="mini-fill dfill-1"></div></div>
        </div>

        <div class="sp-row sp-ok sp-even-2">
          <div class="sp-top">
            <span class="sp-name">Split 2 (Node 2): <code>[5555.., aaaa..)</code></span>
            <span class="load-tag tag-green">書込 約33% (<code>7c1b..</code>)</span>
          </div>
          <div class="mini-bar"><div class="mini-fill dfill-2"></div></div>
        </div>

        <div class="sp-row sp-ok sp-even-3">
          <div class="sp-top">
            <span class="sp-name">Split 3 (Node 3): <code>[aaaa.., ∞)</code></span>
            <span class="load-tag tag-green">書込 約33% (<code>e402..</code>)</span>
          </div>
          <div class="mini-bar"><div class="mini-fill dfill-3"></div></div>
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
    --pcycle: 9s;
    box-sizing: border-box; padding: 12px; background: var(--bg); color: var(--text);
    border: 1px solid var(--border); border-radius: 10px;
    font-family: -apple-system, BlinkMacSystemFont, "Hiragino Sans", "Noto Sans JP", sans-serif;
    font-size: 12px; line-height: 1.35;
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
    background: var(--surface); border: 1px solid var(--border); border-radius: 8px; padding: 10px;
    display: flex; flex-direction: column; justify-content: space-between;
  }
  .card-bad { border: 1.5px solid var(--red); }
  .card-good { border: 1.5px solid var(--green); }
  .pk-head { display: flex; align-items: center; gap: 6px; margin-bottom: 5px; flex-wrap: wrap; }
  .pk-badge { padding: 1px 6px; border-radius: 4px; font-weight: 700; font-size: 12px; }
  .badge-red { background: var(--red-bg); color: var(--red); }
  .badge-green { background: var(--green-bg); color: var(--green); }
  .pk-title { font-weight: 700; font-size: 13px; }
  .pk-desc { font-size: 12px; color: var(--text-muted); margin-bottom: 8px; }
  .split-list { display: flex; flex-direction: column; gap: 5px; }
  .sp-row {
    padding: 6px 8px; border-radius: 6px; border: 1px solid var(--border); background: var(--surface-sub);
    display: flex; flex-direction: column; gap: 4px;
  }
  .sp-top { display: flex; justify-content: space-between; align-items: center; gap: 4px; flex-wrap: wrap; }
  .sp-name { font-weight: 700; font-size: 12px; }
  .mono-state { display: grid; }
  .ms-1, .ms-2, .ms-3 { grid-area: 1 / 1; opacity: 0; text-align: right; }
  .ms-1 { animation: mvis-1 var(--pcycle) infinite; }
  .ms-2 { animation: mvis-2 var(--pcycle) infinite; }
  .ms-3 { animation: mvis-3 var(--pcycle) infinite; }

  .load-tag { padding: 1px 5px; border-radius: 4px; font-size: 12px; font-weight: 700; }
  .tag-zero { background: var(--surface); color: var(--text-muted); }
  .tag-red { background: var(--red-bg); color: var(--red); border: 1px solid var(--red); }
  .tag-green { background: var(--green-bg); color: var(--green); }

  .mini-bar {
    width: 100%; height: 6px; background: var(--surface); border-radius: 3px;
    overflow: hidden; border: 1px solid var(--border);
  }
  .mini-fill { height: 100%; border-radius: 2px; }
  .mfill-1 { background: var(--red); animation: mload-1 var(--pcycle) infinite; }
  .mfill-2 { background: var(--red); animation: mload-2 var(--pcycle) infinite; }
  .mfill-3 { background: var(--red); animation: mload-3 var(--pcycle) infinite; }

  .sp-ok { background: var(--surface); border-color: var(--green); }
  .dfill-1, .dfill-2, .dfill-3 {
    width: 34%; background: var(--green);
    animation: dload-pulse 2.4s infinite ease-in-out;
  }
  .dfill-2 { animation-delay: 0.8s; }
  .dfill-3 { animation-delay: 1.6s; }

  @keyframes mvis-1 { 0%, 30% { opacity: 1; } 33%, 100% { opacity: 0; } }
  @keyframes mvis-2 { 0%, 31% { opacity: 0; } 33%, 63% { opacity: 1; } 66%, 100% { opacity: 0; } }
  @keyframes mvis-3 { 0%, 64% { opacity: 0; } 66%, 97% { opacity: 1; } 100% { opacity: 0; } }

  @keyframes mload-1 { 0%, 30% { width: 100%; } 33%, 100% { width: 0%; } }
  @keyframes mload-2 { 0%, 31% { width: 0%; } 33%, 63% { width: 100%; } 66%, 100% { width: 0%; } }
  @keyframes mload-3 { 0%, 64% { width: 0%; } 66%, 97% { width: 100%; } 100% { width: 0%; } }

  @keyframes dload-pulse {
    0%, 100% { width: 32%; opacity: 0.85; }
    50% { width: 38%; opacity: 1; }
  }

  @media (max-width: 640px) {
    .diagram { padding: 8px; }
    .pk-grid { grid-template-columns: 1fr; gap: 6px; }
    .pk-card { padding: 6px 8px; }
    .pk-desc { margin-bottom: 4px; }
    .sp-row { padding: 4px 6px; gap: 2px; }
  }
  @media (prefers-reduced-motion: reduce) {
    .diagram *, .diagram *::before, .diagram *::after {
      animation: none !important; transition: none !important;
    }
    .ms-3 { opacity: 1; }
    .mfill-3 { width: 100%; }
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
