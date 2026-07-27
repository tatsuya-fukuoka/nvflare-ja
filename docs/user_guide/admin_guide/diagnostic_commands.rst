.. _diagnostic_commands:

################
診断コマンド
################

NVIDIA FLARE は、CellNet レイヤーにおける通信統計の監視とデバッグのための診断コマンドを提供しています。これらのコマンドは、ネットワークの問題のトラブルシューティング、メッセージパターンの分析、システムの性能特性の理解に特に役立ちます。

.. note::
   これらの診断コマンドは、NetManager コンポーネントで diagnose モードが有効になるようにシステムが設定されている場合にのみ利用できます。

概要
====

診断コマンドを使用すると、管理者は次のことが行えます。

* CellNet システム内のアクティブなセルを検出する
* メッセージサイズとタイミングに関する統計を表示する
* セル間の通信パターンを監視する
* 利用可能な統計プールを調査する
* さまざまな統計モードでヒストグラムデータを分析する

これらのコマンドは、CellNet レイヤーの統計トラッキングシステムに問い合わせます。このシステムは、システム通信のさまざまな側面を監視するための多様な統計プールを保持しています。

統計プール
==========

NVIDIA FLARE の統計システムは、さまざまな種類のメトリクスを整理するために「プール」を使用します。

* **ヒストグラムプール** : 設定可能なビンを用いて値の分布 (メッセージサイズやタイミングなど) を追跡します
* **カウンタープール** : 特定のイベントに対する単純なカウンターを追跡します

各プールには名前、種類、説明があります。システムはメッセージ統計を追跡するためのプールを自動的に作成し、アプリケーションはドメイン固有のメトリクスを追跡するためのカスタムプールを作成できます。

統計プールの保存設定
=====================

デフォルトでは、統計プールはジョブの実行中にメモリ上に保持されます。ただし、後からの分析や記録保持のために、プールの統計をディスクへ保存するよう NVFLARE を設定することもできます。

meta.json での設定
-------------------

ジョブで統計プールの保存を有効にするには、ジョブの ``meta.json`` ファイルに次の設定を追加します。

.. code-block:: json

   {
     "stats_pool_config": {
       "save_pools": [
         "request_processing",
         "request_response",
         "*"
       ]
     }
   }

**設定オプション:**

* ``save_pools``: 保存するプール名のリストです。次の指定をサポートします。

  * **特定のプール名** : 例 ``"request_processing"``、``"msg_sizes"``
  * **ワイルドカード** : ``"*"`` を使用してすべてのプールを保存します
  * **混在** : 特定の名前とワイルドカードを組み合わせます

**例:**

1. 特定のプールのみを保存する場合:

.. code-block:: json

   {
     "stats_pool_config": {
       "save_pools": ["request_processing", "request_response"]
     }
   }

2. すべてのプールを保存する場合:

.. code-block:: json

   {
     "stats_pool_config": {
       "save_pools": ["*"]
     }
   }

出力ファイル
-------------

統計プールの保存を有効にすると、NVFLARE はジョブの実行終了時にジョブごとに 2 つのファイルを生成します。

stats_pool_summary.json
^^^^^^^^^^^^^^^^^^^^^^^^

**場所:** ジョブのワークスペースディレクトリ

**内容:** 保存された各プールについて、ヒストグラムのサマリと集計統計が含まれます。

**形式:** 次の構造を持つ JSON ファイルです。

.. code-block:: json

   {
     "pool_name": {
       "name": "pool_name",
       "type": "hist",
       "description": "Pool description",
       "marks": [0, 10, 100, 1000],
       "bins": [
         {"range": "0-10", "count": 150, "total": 750.5, "min": 0.1, "max": 9.9},
         {"range": "10-100", "count": 80, "total": 4200.0, "min": 10.2, "max": 99.8},
         {"range": "100-1000", "count": 20, "total": 12000.0, "min": 105.0, "max": 950.0}
       ]
     },
     "another_pool": {
       ...
     }
   }

**ユースケース:**

* ジョブ完了後の通信パターンの分析
* 複数のジョブ実行にまたがる履歴比較
* システム性能に関するレポートの生成
* 時系列でのトレンドの特定

stats_pool_records.csv
^^^^^^^^^^^^^^^^^^^^^^^

**場所:** ジョブのワークスペースディレクトリ

**内容:** 保存されたプールで収集された各データポイントについて、タイムスタンプ付きの生の記録が含まれます。

**形式:** プールの種類によって列が異なる CSV ファイルです。

ヒストグラムプールの場合:

.. code-block:: text

   timestamp,pool_name,value,additional_metadata
   2024-01-15T10:30:45.123Z,request_processing,0.025,
   2024-01-15T10:30:45.456Z,request_processing,0.031,
   2024-01-15T10:30:46.789Z,request_response,0.120,

カウンタープールの場合:

.. code-block:: text

   timestamp,pool_name,counter_name,value
   2024-01-15T10:30:45.123Z,event_counts,task_received,1
   2024-01-15T10:30:46.456Z,event_counts,task_completed,1

**ユースケース:**

* 詳細なタイムライン分析
* 独自のデータ処理と可視化
* 外部の分析ツールとの連携
* システムの挙動パターンに対する機械学習
* 特定のイベントや異常のデバッグ

ワークフローの例
-----------------

**ステップ 1: ジョブの設定**

ジョブの ``meta.json`` を作成または編集します。

.. code-block:: json

   {
     "name": "my_federated_job",
     "resource_spec": {},
     "min_clients": 2,
     "stats_pool_config": {
       "save_pools": ["*"]
     }
   }

**ステップ 2: ジョブの投入**

.. code-block:: shell

   > submit_job my_job_folder

**ステップ 3: ジョブの実行**

ジョブは通常どおり実行され、その裏で統計が収集されます。

**ステップ 4: ジョブ完了後の統計の取得**

.. code-block:: shell

   > download_job job_abc-123-def

**ステップ 5: 出力の分析**

ダウンロードしたジョブのワークスペースに移動して、次を確認します。

.. code-block:: shell

   cd downloaded_job/workspace/

   # View summary statistics
   cat stats_pool_summary.json

   # Analyze raw records
   cat stats_pool_records.csv

**ステップ 6: 統計を用いた分析**

.. code-block:: python

   import json
   import pandas as pd

   # Load summary data
   with open('stats_pool_summary.json', 'r') as f:
       summary = json.load(f)

   # Load raw records
   records = pd.read_csv('stats_pool_records.csv')

   # Analyze timing patterns
   timing_data = records[records['pool_name'] == 'request_processing']
   print(f"Average: {timing_data['value'].mean()}")
   print(f"95th percentile: {timing_data['value'].quantile(0.95)}")

Stats Viewer ツールの使用
--------------------------

NVFLARE は、統計ファイルを対話的に調べるための ``stats_viewer`` という便利なコマンドラインツールを提供しています。このツールを使うと、独自のスクリプトを書かずに ``stats_pool_summary.json`` ファイルを表示・分析できます。

**Stats Viewer の起動:**

.. code-block:: shell

   python -m nvflare.fuel.f3.qat.stats_viewer -f stats_pool_summary.json

これにより、統計データを調べられる対話型シェルが起動します。

**利用可能なコマンド:**

stats viewer では次のコマンドが利用できます。

* ``list_pools``: 利用可能なすべての統計プールを、その種類と説明とともに表示します
* ``show_pool <pool_name> [mode]``: 特定のプールの詳細な統計を表示します

  * ``pool_name``: 表示するプールの名前
  * ``mode`` (省略可): ヒストグラムの表示モード。``count``、``total``、``min``、``max``、``avg`` のいずれか

* ``help`` または ``?``: 利用可能なコマンドを一覧表示します
* ``bye``: stats viewer を終了します

**セッションの例:**

.. code-block:: shell

   $ python -m nvflare.fuel.f3.qat.stats_viewer -f stats_pool_summary.json
   Type help or ? to list commands.

   > list_pools
   Name                  Type    Description
   -------------------- ------- ------------------------------
   request_processing   hist    Request processing time
   request_response     hist    Request-response round trip
   msg_sizes            hist    Message size distribution

   > show_pool request_processing avg
   Range         Count    Average
   ------------ ------- -----------
   0-10ms           150     5.2ms
   10-100ms          80    45.3ms
   100-1000ms        20   425.8ms

   > show_pool msg_sizes count
   Range         Count
   ------------ -------
   0-1KB           200
   1KB-10KB        150
   10KB-100KB       50

   > bye

**サーバ側統計とクライアント側統計:**

``stats_viewer`` ツールは、サーバ側とクライアント側の両方の統計を分析できます。

* **サーバ側統計** : ジョブ完了後、サーバのジョブワークスペースで利用できます。``download_job`` コマンドで取得できます。
* **クライアント側統計** : 現在は、各クライアントサイトのそれぞれのジョブワークスペースにローカルで保存されます。

.. note::
   現時点では、クライアント側の統計ファイルはジョブ完了後に自動的にサーバへ送信されません。クライアントの統計を分析するには、各クライアントサイトのジョブワークスペースにある ``stats_pool_summary.json`` ファイルに直接アクセスする必要があります。

一般的なプール名
-----------------

NVFLARE のジョブでは、次のプールが一般的に利用できます。

通信関連のプール
^^^^^^^^^^^^^^^^^

* ``request_processing``: リクエストの処理に要した時間
* ``request_response``: エンドツーエンドのリクエスト・レスポンス時間
* ``msg_sizes``: メッセージサイズの分布
* ``msg_travel_time``: メッセージの伝送時間

ジョブ固有のプール
^^^^^^^^^^^^^^^^^^^

ジョブによっては、そのワークフローに基づいてカスタムプールが作成されることがあります。ジョブの実行中に ``list_pools`` コマンドを使用して、利用可能なプールを確認してください。

.. code-block:: shell

   > cells
   server.job_abc-123
   site1.job_abc-123

   > list_pools server.job_abc-123


モニタリングとの連携
----------------------

統計プールのデータは、外部のモニタリングシステムを補完します。

* **統計プール** : ジョブのアーティファクトとともに保存される、詳細でジョブ固有のメトリクス
* **外部モニタリング** (Prometheus/Grafana): リアルタイムでのシステム全体の監視

両方のアプローチを組み合わせて使用してください。

1. リアルタイムのアラートとダッシュボードには外部モニタリングを使用します
2. 詳細なジョブ完了後の分析と履歴の記録には統計プールの保存を使用します

外部モニタリングのセットアップについては :ref:`monitoring` を参照してください。

利用可能なコマンド
===================

cells
-----

**説明:** CellNet システム内のアクティブなすべてのセルを FQCN (Fully Qualified Cell Name) とともに一覧表示します。このコマンドは、他の診断コマンドで使用できるターゲットを見つけるために不可欠です。

**使い方:**

.. code-block:: shell

   cells

**パラメータ:**

なし。このコマンドはパラメータを取りません。

**出力:**

システム内のアクティブなすべてのセルの一覧を表示します。各セルの FQCN が 1 行ずつ表示され、最後に有効なセルの総数を示すサマリ行が続きます。

**例:**

.. code-block:: shell

   > cells

**出力例:**

.. code-block:: text

   server
   site1
   site2
   site3
   server.abc-123-def
   site1.abc-123-def
   site2.abc-123-def
   Total Cells: 7

**出力の理解:**

一覧表示されるセルには次のものが含まれます。

* **親セル** : 各サイトのベースとなるセル (例: ``server``、``site1``、``site2``)

  * サーバの親セルの名前は常に ``server`` です
  * クライアントの親セルはそのサイト名を使用します

* **ジョブセル** : アクティブなジョブのために作成されたセル (例: ``server.abc-123-def``、``site1.abc-123-def``)

  * 形式: ``<site_name>.<job_id>``
  * ジョブがデプロイされたときに作成されます
  * ジョブが完了すると削除されます

* **リレーセル** : 階層型のデプロイにおけるリレーノード (例: ``relay1``、``relay1.site1``)

  * 通信階層における中間ノードです
  * ジョブの実行中は、独自のジョブセルを持つことがあります

**ユースケース:**

* **利用可能なターゲットの検出** : ``list_pools``、``show_pool``、``msg_stats`` などの診断コマンドで使用できる有効な FQCN を見つけます
* **システムトポロジの確認** : 想定されるすべてのサイトが接続され、アクティブであることを確認します
* **ジョブセルの監視** : ジョブセルの FQCN を特定して、現在実行中のジョブを把握します
* **接続性のトラブルシューティング** : 欠落しているセルや切断されたセルを特定します
* **階層構造の理解** : 階層型のデプロイにおけるセル構造を可視化します

**後続コマンドとの組み合わせ例:**

``cells`` を実行してターゲットを検出した後、その FQCN を他のコマンドで使用できます。

.. code-block:: shell

   # First, discover all cells
   > cells
   server
   site1
   site2
   server.job123
   site1.job123
   Total Cells: 5

   # Then query specific cells
   > msg_stats server
   > msg_stats site1
   > msg_stats server.job123
   > list_pools site1.job123

**さまざまなセルタイプの解釈:**

1. **サーバ親セル** (``server``):

   * FL システムが稼働している間は常に存在します
   * 管理操作を処理します
   * サーバ上のすべてのジョブセルの親となります

2. **クライアント親セル** (``site1``、``site2`` など):

   * 接続された FL クライアントサイトごとに 1 つ存在します
   * クライアントが接続されている間はアクティブです
   * 複数のジョブをまたいで存続します

3. **ジョブサーバセル** (``server.<job_id>``):

   * サーバ上でジョブがデプロイされたときに作成されます
   * ジョブ固有のサーバ側ワークフローを含みます
   * ジョブが完了すると削除されます

4. **ジョブクライアントセル** (``<site_name>.<job_id>``):

   * ジョブに参加するクライアントごとに 1 つ存在します
   * クライアント側のジョブロジックを実行します
   * 対応するサーバのジョブセルと通信します

5. **階層型セル** (``relay1``、``relay1.site1``):

   * 階層型デプロイにおけるリレーノードです
   * 入れ子にできます (例: ``relay1.relay2.site1``)
   * 大規模なデプロイの管理に役立ちます

**ヒント:**

* 他の診断コマンドを実行する前に ``cells`` を実行して、有効なターゲットを特定してください
* セルの一覧を時系列で比較して、システムの変化を追跡してください
* 想定されるセルが見つからない場合は、接続性とサイトのステータスを確認してください
* ジョブセルはジョブの開始時に現れ、完了時に消えます

list_pools
----------

**説明:** ターゲットセルで利用可能なすべての統計プールを一覧表示します。

**使い方:**

.. code-block:: shell

   list_pools target

**パラメータ:**

* ``target`` - 問い合わせ先となるターゲットセルの FQCN (Fully Qualified Cell Name) (例: "server"、"client1"、"server.job_id")

**出力:**

3 つの列を持つテーブルを表示します。

* **pool** - 統計プールの名前
* **type** - プールの種類 (ヒストグラムの場合は "hist"、カウンターの場合は "counter")
* **description** - そのプールが何を追跡しているかの説明

**例:**

.. code-block:: shell

   > list_pools server

**出力例:**

.. code-block:: text

   +------------------+----------+--------------------------------+
   | pool             | type     | description                    |
   +------------------+----------+--------------------------------+
   | msg_travel_time  | hist     | Message travel time in seconds |
   | msg_sizes        | hist     | Message size distribution      |
   | request_counts   | counter  | Request counts by channel      |
   +------------------+----------+--------------------------------+

**ユースケース:**

* セル上で利用可能な統計プールを検出する
* 想定される統計トラッキングが設定されていることを確認する
* ``show_pool`` で詳細に調査するプールを特定する

show_pool
---------

**説明:** ターゲットセル上の特定のプールについて詳細な統計を表示します。

**使い方:**

.. code-block:: shell

   show_pool target pool_name [mode]

**パラメータ:**

* ``target`` - 問い合わせ先となるターゲットセルの FQCN
* ``pool_name`` - 表示する統計プールの名前
* ``mode`` - (省略可) ヒストグラムプールの表示モード。有効な値は次のとおりです。

  * ``count`` - 各ビンに含まれる値の個数を表示します (デフォルト)
  * ``percent`` - 各ビンに含まれる値の割合を表示します
  * ``avg`` - 各ビンの平均値を表示します
  * ``min`` - 各ビンの最小値を表示します
  * ``max`` - 各ビンの最大値を表示します

**出力:**

ヒストグラムプールの場合は、ビンごとの値の分布を示すテーブルを表示します。正確な列はプールの種類と設定によって異なります。

カウンタープールの場合は、カウンター名と現在の値を含むテーブルを表示します。

**例:**

.. code-block:: shell

   # Show message size distribution with counts
   > show_pool server msg_sizes count

   # Show message timing with averages
   > show_pool server msg_travel_time avg

   # Show message size percentages
   > show_pool site1 msg_sizes percent

**出力例 (count モード):**

.. code-block:: text

   +---------------+-------+
   | Range         | Count |
   +---------------+-------+
   | 0-1KB         | 150   |
   | 1KB-10KB      | 450   |
   | 10KB-100KB    | 80    |
   | 100KB-1MB     | 20    |
   | >1MB          | 5     |
   +---------------+-------+

**出力例 (avg モード):**

.. code-block:: text

   +---------------+-----------+
   | Range         | Avg (sec) |
   +---------------+-----------+
   | 0-10ms        | 5.2e-03   |
   | 10ms-100ms    | 4.5e-02   |
   | 100ms-1s      | 3.2e-01   |
   | >1s           | 2.1e+00   |
   +---------------+-----------+

**ユースケース:**

* メッセージサイズの分布を分析して外れ値を特定する
* リクエストのタイミング特性を監視する
* 異なるセル間で統計を比較する
* 性能のボトルネックや異常なパターンを特定する

msg_stats
---------

**説明:** ターゲットセルのメッセージリクエスト統計を表示します。これは、あらかじめ設定されたメッセージ統計プールを表示する便利コマンドです。

**使い方:**

.. code-block:: shell

   msg_stats target [mode]

**パラメータ:**

* ``target`` - 問い合わせ先となるターゲットセルの FQCN
* ``mode`` - (省略可) 表示モード。有効な値は次のとおりです。

  * ``count`` - メッセージの件数を表示します (デフォルト)
  * ``percent`` - メッセージの割合を表示します
  * ``avg`` - メッセージサイズまたはタイミングの平均を表示します
  * ``min`` - 最小値を表示します
  * ``max`` - 最大値を表示します

**出力:**

リクエストメッセージに関する統計を表示します。通常は、メッセージサイズの分布やタイミング情報、あるいはその両方が示されます。正確な形式は、システムでメッセージ統計プールがどのように設定されているかによって異なります。

**例:**

.. code-block:: shell

   # Show message counts
   > msg_stats server

   # Show average message characteristics
   > msg_stats server avg

   # Show maximum values
   > msg_stats client1 max

**出力例:**

.. code-block:: text

   Message Statistics for server:
   +---------------+-------+----------+
   | Size Range    | Count | Avg Time |
   +---------------+-------+----------+
   | 0-1KB         | 245   | 12ms     |
   | 1KB-10KB      | 180   | 25ms     |
   | 10KB-100KB    | 45    | 150ms    |
   | >100KB        | 10    | 500ms    |
   +---------------+-------+----------+

**ユースケース:**

* メッセージトラフィックのパターンをすばやく概観する
* 通信の健全性を監視する
* 異常なメッセージパターンを特定する
* システムの性能特性のベースラインを取得する

一般的なワークフロー
=====================

利用可能なターゲットの検出
---------------------------

診断コマンドを使用する前に、利用可能なセルを検出します。

1. **アクティブなセルをすべて一覧表示する:**

   .. code-block:: shell

      > cells

2. **対象となるターゲットセルを特定する:**

   * システム全体の監視には親セル (``server``、``site1`` など)
   * ジョブ固有の監視にはジョブセル (``server.job_id``、``site1.job_id``)
   * 階層型デプロイにおけるリレーセル

3. **セルの接続性を確認する:**

   想定されるセルが一覧に現れていることを確認します。セルが見つからない場合、接続の問題を示している可能性があります。

通信の問題の調査
-----------------

セル間の通信の問題を調査する場合:

1. **アクティブなセルを検出する:**

   .. code-block:: shell

      > cells

2. **利用可能なプールを一覧表示する:**

   .. code-block:: shell

      > list_pools server
      > list_pools client1

3. **メッセージ統計を確認する:**

   .. code-block:: shell

      > msg_stats server count
      > msg_stats client1 count

4. **特定のプールを詳しく調べる:**

   .. code-block:: shell

      > show_pool server msg_travel_time avg
      > show_pool client1 msg_sizes percent

性能分析
---------

システムの性能特性を分析するには:

1. **メッセージのタイミング分布を確認する:**

   .. code-block:: shell

      > show_pool server msg_travel_time count
      > show_pool server msg_travel_time avg

2. **メッセージサイズのパターンを分析する:**

   .. code-block:: shell

      > show_pool server msg_sizes count
      > show_pool server msg_sizes max

3. **セル間で比較する:**

   .. code-block:: shell

      > msg_stats server avg
      > msg_stats client1 avg
      > msg_stats client2 avg

ジョブ実行の監視
-----------------

ジョブの実行中に通信パターンを監視します。

1. **ジョブセルを特定する:**

   .. code-block:: shell

      > cells
      # Look for cells with format: <site_name>.<job_id>

2. **ジョブセルの統計を確認する:**

   .. code-block:: shell

      > list_pools server.job_abc123
      > msg_stats server.job_abc123 count

3. **親セルとジョブセルを比較する:**

   .. code-block:: shell

      > msg_stats server avg
      > msg_stats server.job_abc123 avg

統計モードの解説
=================

統計モードが異なると、データに対する異なる視点が得られます。

count
-----
各ビンに含まれるデータポイントの数を表示します。分布を把握し、ほとんどの値がどこに位置するかを特定するのに役立ちます。

**ユースケース:** 「1KB〜10KB の範囲にはいくつのメッセージがあるか?」

percent
-------
すべてのデータポイントのうち、各ビンに含まれる割合を表示します。これにより分布が正規化され、異なる期間やセル間での比較が容易になります。

**ユースケース:** 「100KB を超えるメッセージの割合はどれくらいか?」

avg
---
各ビン内のデータポイントの平均値を表示します。各範囲における典型的な特性を把握するのに役立ちます。

**ユースケース:** 「10ms〜100ms のレイテンシ範囲にあるメッセージの典型的なレイテンシはどれくらいか?」

min
---
各ビンで観測された最小値を表示します。ベストケースのシナリオを把握するのに役立ちます。

**ユースケース:** 「1KB〜10KB のメッセージ範囲で観測された最速のレスポンス時間はどれくらいか?」

max
---
各ビンで観測された最大値を表示します。ワーストケースのシナリオや外れ値を特定するのに役立ちます。

**ユースケース:** 「小さなメッセージで観測された最長のレイテンシはどれくらいか?」

ターゲットセルのアドレッシング
===============================

これらのコマンドの ``target`` パラメータは、FQCN (Fully Qualified Cell Name) によるアドレッシングを使用します。

サーバセル
-----------

.. code-block:: shell

   > msg_stats server

クライアントセル
-----------------

.. code-block:: shell

   > msg_stats site1
   > msg_stats client_alpha

ジョブセル
-----------

ジョブの実行中は、各サイトが ``<site_name>.<job_id>`` という形式の FQCN を持つ専用のジョブセルを持ちます。

.. code-block:: shell

   > msg_stats server.abc-123-def
   > msg_stats site1.abc-123-def

階層型セル
-----------

リレーを使用する階層型デプロイでは:

.. code-block:: shell

   > msg_stats relay1
   > msg_stats relay1.site1

通信階層の詳細については :ref:`hierarchical_communication` を参照してください。

ヒントとベストプラクティス
===========================

1. **定期的な監視:** 通常運用時のベースライン統計を確立しておくと、異常の特定に役立ちます。

2. **セルの比較:** 異なるセル間で統計を比較して、不整合や特定のサイトに固有の問題を特定します。

3. **異なるモードの活用:** 統計モードを切り替えることで、同じデータから異なる洞察が得られます。

4. **時系列での追跡:** コマンドを定期的に実行して出力を保存し、時系列のトレンドを追跡します。

5. **ジョブ単位の分析:** ジョブセルを親セルとは別に監視して、ジョブ固有の通信パターンを把握します。

6. **ログとの相関:** 包括的なトラブルシューティングのために、診断コマンドとログ分析を組み合わせて使用します。

トラブルシューティング
=======================

コマンドが見つからない
-----------------------

診断コマンドが利用できない場合:

* NetManager コンポーネントが ``diagnose=True`` で設定されていることを確認してください
* これらのコマンドを実行する適切な権限があるか確認してください
* これらのコマンドを含むバージョンの NVIDIA FLARE を使用していることを確認してください

cells コマンドで表示されるセルが想定より少ない
-----------------------------------------------

``cells`` コマンドで想定されるすべてのセルが表示されない場合:

* **接続性の確認** : すべてのサイトがサーバに接続されていることを確認してください
* **サイトのステータス確認** : ``check_status`` を使用して、クライアントが正しく接続されているか確認してください
* **初期化の待機** : サイトは起動後、表示されるまで少し時間がかかる場合があります
* **ログの確認** : サーバとクライアントのログで接続エラーを確認してください
* **ネットワークの確認** : ネットワークの問題やファイアウォールによるブロックがないことを確認してください

cells コマンドに古いジョブセルが表示される
-------------------------------------------

ジョブ完了後もジョブセルが一覧に残っている場合:

* クリーンアップに遅延がある可能性があります。少し待ってから再度 ``cells`` を実行してください
* ``list_jobs`` を使用して、そのジョブが実際にまだ実行中かどうか確認してください
* ジョブのシャットダウン中にエラーが発生していないか、ログを確認してください

無効なモードのエラー
---------------------

"invalid mode" というエラーが表示される場合:

* 有効なモードである ``count``、``percent``、``avg``、``min``、``max`` のいずれかを使用しているか確認してください
* mode パラメータにタイプミスがないか確認してください
* モードは大文字小文字を区別します (小文字を使用してください)

ターゲットが見つからない
-------------------------

ターゲットセルに到達できない場合:

* FQCN が正しいか確認してください
* ターゲットセルが稼働し、接続されていることを確認してください
* ``cells`` コマンドを使用して、利用可能なセルを一覧表示してください

プールが存在しない
-------------------

"pool does not exist" というエラーが表示される場合:

* ``list_pools`` を使用して、そのセルで利用可能なプールを確認してください
* プール名のつづりが正しいか確認してください
* プール名は大文字小文字を区別します

関連項目
========

* :ref:`cellnet_architecture` - FLARE の通信レイヤーについて学ぶ
* :ref:`communication_configuration` - 通信設定を構成する
* :ref:`monitoring` - Prometheus と Grafana による外部モニタリングをセットアップする
* :ref:`hierarchical_communication` - 階層型のセルトポロジを理解する

