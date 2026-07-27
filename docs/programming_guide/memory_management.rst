.. _memory_management:

###################################
メモリ管理
###################################

このガイドでは、Python、PyTorch、glibc/jemalloc を使用する長時間実行のフェデレーテッド
ラーニングジョブ向けのメモリ管理手法を説明します。

.. contents:: 目次
   :local:
   :depth: 2

概要
====================

フェデレーテッドラーニングのジョブは数時間から数日にわたって実行されることがあります。
適切なメモリ管理を行わないと、以下の理由で RSS (Resident Set Size) が増加し続ける
可能性があります。

- ラウンド間で大きなモデルパラメータを保持し続ける長寿命の参照(クライアント側の主な原因)
- glibc のメモリアリーナの断片化(解放されたメモリが OS に返却されない)
- PyTorch の CUDA キャッシュの保持
- Python のガベージコレクションを遅延させる循環参照(副次的要因。通常は主因ではありません)

NVFlare は、サーバ側とクライアント側の双方でメモリを効果的に管理するためのユーティリティ
と設定オプションを提供します。フレームワークは使用中のメモリアロケータ (glibc または
jemalloc) を自動的に検出し、それに応じてクリーンアップ戦略を適応させます。

アロケータのサポート
========================

NVFlare は2つのメモリアロケータをサポートしています。

**glibc (ほとんどの Linux でのデフォルト)**
    ``malloc_trim()`` を使用して、空きヒープページを OS に返却します。
    最適なメモリ挙動のために ``MALLOC_ARENA_MAX`` の設定が必要です。

**jemalloc (PyTorch に推奨)**
    メモリ管理に auto-decay を使用します。``MALLOC_CONF`` で設定します。
    ``malloc_trim()`` の呼び出しは不要です (jemalloc が自動的に処理します)。

NVFlare は、実行時にどちらのアロケータが使用されているかを自動的に検出します。

プラットフォーム互換性
==========================

すべてのメモリ管理機能がすべてのプラットフォームで動作するわけではありません。
以下の表に互換性をまとめます。

+---------------------------+-------------+-------------+-------------+
| 機能                      | Linux/glibc | Linux/musl  | macOS       |
+===========================+=============+=============+=============+
| ``gc.collect()``          | ✓           | ✓           | ✓           |
+---------------------------+-------------+-------------+-------------+
| ``MALLOC_ARENA_MAX``      | ✓           | ✗           | ✗           |
+---------------------------+-------------+-------------+-------------+
| ``malloc_trim()``         | ✓           | ✗           | ✗           |
+---------------------------+-------------+-------------+-------------+
| ``torch.cuda.empty_cache``| ✓           | ✓           | ✓           |
+---------------------------+-------------+-------------+-------------+

**注記:**

- **Linux/glibc**: 標準的な Linux ディストリビューション (Ubuntu、RHEL、Debian など)
- **Linux/musl**: Alpine Linux およびその他の musl ベースのディストリビューション
- **macOS**: ``malloc_trim()`` は警告なくスキップされます (安全な no-op です)

.. warning::

   メモリ効率を最大化するには、glibc を使用する Linux を利用してください。
   Alpine Linux (musl) と macOS でも、クライアント側のパラメータ参照の解放
   (および任意の ``gc.collect()``) による恩恵は得られますが、``malloc_trim()`` を
   介して断片化したヒープメモリを OS に返却することはできません。

環境変数
====================

NVFlare のプロセスを起動する前に、以下の環境変数を設定してください。

クライアント (学習ノード)
-----------------------------

.. code-block:: bash

    export MALLOC_ARENA_MAX=2

**理由:** クライアントは通常、CPU メモリが限られています。``MALLOC_ARENA_MAX=2``
を設定することで、アリーナの増大を防ぎ、メモリの断片化を軽減します。

サーバ (集約ノード)
-----------------------------

.. code-block:: bash

    export MALLOC_ARENA_MAX=4

**理由:** サーバは CPU メモリの消費が大きく (モデルサイズの4〜7倍)、マルチスレッドの
ネットワーク処理を行います。``MALLOC_ARENA_MAX=4`` はスループットとメモリのバランスを
取ります。並列度が高い場合は ``8`` を使用してください。

サーバ側のメモリクリーンアップ
==================================

FedAvg の Controller は、``server_memory_gc_rounds`` パラメータによる自動メモリ
クリーンアップをサポートしています。

サーバ側の設定
--------------------------

.. code-block:: python

    from nvflare.recipe.fedavg import FedAvgRecipe

    recipe = FedAvgRecipe(
        name="my_job",
        min_clients=4,
        num_rounds=100,
        train_script="client.py",
        server_memory_gc_rounds=5,  # Cleanup every 5 rounds
    )

**値:**

- ``0`` = 無効 (FedAvg ベースのレシピのデフォルト)
- ``1`` = 毎ラウンドでクリーンアップ (FedOpt、FedAvgHE、Cyclic レシピのデフォルト)
- ``5`` = 5ラウンドごとにクリーンアップ (サーバでの推奨値)

サーバのクリーンアップの効果
--------------------------------

有効な場合、N ラウンドごとの終了時に以下が実行されます。

1. Python のガベージコレクションを実行します (``gc.collect()``)
2. 空きヒープページを OS に返却します (``malloc_trim()``、Linux/glibc のみ)

パフォーマンスへの影響
------------------------------

一般的なフェデレーテッドラーニングのワークロードでは、メモリクリーンアップの
オーバーヘッドはごくわずかです。

+---------------------------+------------------+------------------------------------+
| 処理                      | 所要時間の目安   | 備考                               |
+===========================+==================+====================================+
| ``gc.collect()``          | 10-500 ms        | Python オブジェクト数に依存        |
+---------------------------+------------------+------------------------------------+
| ``malloc_trim()``         | < 1 ms           | 非常に高速 (ページテーブル操作)    |
+---------------------------+------------------+------------------------------------+

**オーバーヘッドの分析:**

- **学習ラウンドの所要時間**: 通常30秒から10分以上
- **クリーンアップの所要時間**: 合計10〜500 ms
- **1ラウンドあたりのオーバーヘッド**: 通常1%未満

``server_memory_gc_rounds=5`` の**場合**:

- クリーンアップは5ラウンドに1回実行されます
- 合計オーバーヘッド: 学習時間の0.2%未満

**推奨事項**: ``server_memory_gc_rounds=5`` を使用すると、パフォーマンスへの影響を
ほとんど与えずに良好なメモリ管理が得られます。無効化 (``=0``) するのは、クリーンアップ
なしでも RSS が安定していることを実測して確認した場合のみにしてください。

クライアント側のメモリクリーンアップ
========================================

クライアント側の主要なメモリ制御は、``flare.send()`` の ``clear_cache=True``
(デフォルト) であり、シリアライズ後に直ちにパラメータの参照を解放します。
CPython では、この参照の解放こそが実際に大きなテンソル/配列のメモリを解放するものであり、
そのために明示的な GC 呼び出しは必要ありません。

``client_memory_gc_rounds`` と ``cuda_empty_cache`` は、参照の解放に加えて実行される
*補助的な*クリーンアップを提供します。すなわち、循環参照のオブジェクトに対する定期的な
``gc.collect()``、解放済みページを OS に返却する ``malloc_trim()``、および任意の
CUDA キャッシュのクリアです。

クライアント側の設定
--------------------------

.. code-block:: python

    from nvflare.recipe.fedavg import FedAvgRecipe

    recipe = FedAvgRecipe(
        name="my_job",
        min_clients=4,
        num_rounds=100,
        train_script="client.py",

        # Server-side cleanup
        server_memory_gc_rounds=5,

        # Client-side cleanup
        client_memory_gc_rounds=1,   # Cleanup every round
        cuda_empty_cache=True, # Clear GPU cache
    )

Swarm Learning の設定
------------------------------

Swarm Learning では、``SwarmLearningRecipe`` において
(``client_memory_gc_rounds`` ではなく) ``memory_gc_rounds`` と
``cuda_empty_cache`` を使用します。

.. code-block:: python

    from nvflare.app_opt.pt.recipes.swarm import SwarmLearningRecipe

    recipe = SwarmLearningRecipe(
        name="swarm_job",
        model=MyModel(),
        min_clients=3,
        num_rounds=10,
        train_script="train.py",
        memory_gc_rounds=1,    # Cleanup every round on trainer and aggregator roles
        cuda_empty_cache=True,
        round_timeout=3600,    # P2P model-transfer ACK budget; increase for large models
    )

.. note::

   ``memory_gc_rounds`` と ``cuda_empty_cache`` は Swarm レシピのトップレベル引数です。
   これらを ``train_args`` の中に渡さないでください (予約キーです)。

**パラメータ:**

- ``client_memory_gc_rounds``: クライアント側で N ラウンドごとに*補助的な*クリーンアップ (``gc.collect()`` + ``malloc_trim()``) を実行します (0 = 無効)。主要なクリーンアップは ``flare.send()`` の ``clear_cache=True`` による参照の解放です。
- ``cuda_empty_cache``: True の場合、クリーンアップ時に ``torch.cuda.empty_cache()`` を呼び出します
- ``memory_gc_rounds`` (Swarm): N ラウンドごとに補助的なクリーンアップを実行します (0 = 無効)

``client_memory_gc_rounds > 1`` を使用すべき場合
--------------------------------------------------

``1`` より大きい値は、メモリがすでに安定していて、クリーンアップのオーバーヘッドを
下げるためにチューニングする場合にのみ使用してください。

- RSS の推移がラウンド間で平坦/上限内に収まっている
- CPU/GPU の OOM 圧力がない
- スループット/レイテンシをわずかでも改善したい

まず ``client_memory_gc_rounds=1`` から始め、RSS を監視しながら ``2``、必要に応じて
``5`` へとチューニングしてください。RSS が上昇し始めたり OOM のリスクが高まったりした
場合は、``1`` に戻してください。

クライアントのクリーンアップの効果
--------------------------------------

クライアントで ``flare.send()`` を実行するたびに (デフォルトの ``clear_cache=True``
の場合)、以下が行われます。

1. FLARE が送信済みおよび受信済みのモデルパラメータへの参照を解放します。
2. CPython では、この参照の解放が大きなテンソル/配列を回収する主要な仕組みです。

さらに、補助的なクリーンアップも利用可能で、設定できます。

3. Python のガベージコレクション (``gc.collect()``) を実行します。主に循環参照のためです。
4. glibc の場合: 空きヒープページを OS に返却します (``malloc_trim()``)。
5. jemalloc の場合: auto-decay に依存します (手動の操作は不要です)。
6. 任意で PyTorch の CUDA キャッシュをクリアします。

.. note::

   アロケータが再利用のためにメモリを保持することがあるため、オブジェクトを解放しても
   RSS がすぐに下がるとは限りません。ラウンド間で RSS の推移が平坦であることが、通常は
   期待される健全な挙動です。

.. note::

   ライフサイクルの処理はユーザーの学習スクリプトに対して透過的です。デフォルトの
   動作のために ``train.py`` にコード変更を加える必要はありません。

クライアント学習プロセスのメモリクリーンアップ
--------------------------------------------------

サブプロセスモードのジョブ (``launch_external_process=True``) では、メモリクリーン
アップは学習サブプロセスだけでなく、クライアント学習プロセスのすべてのステージに
わたって実行されます。各ラウンドの結果が転送された後、同じ GC およびヒープトリムの
サイクルがクライアント学習プロセスの各ステージに適用され、長時間のジョブでの RSS の
増加を防ぎます。

クリーンアップの頻度と GPU キャッシュの挙動は、上ですでに説明した同じ
``memory_gc_rounds`` / ``client_memory_gc_rounds`` および ``cuda_empty_cache``
パラメータで制御されます。

すべてのステージにわたる RSS のプロファイリングは、環境変数
``NVFLARE_CLIENT_MEMORY_PROFILE=1`` で有効化できます。これにより、送信と受信のたびに
ステージごとの RSS ログマーカーが出力され、grep による分析が容易になります。

外部プロセスの設定
--------------------------

外部プロセスでの実行 (``launch_external_process=True``) では、メモリ設定は環境変数を
介して渡されます。

- ``NVFLARE_CLIENT_MEMORY_GC_ROUNDS``: クリーンアップの間隔
- ``NVFLARE_CUDA_EMPTY_CACHE``: GPU キャッシュのクリーンアップ (``true``/``false``)
- ``NVFLARE_CLIENT_MEMORY_PROFILE``: ラウンドごとの RSS ログを有効にするには ``1`` を設定

推奨設定
====================

+---------------+-----------------------------+-----------------------------+----------------------+----------------------+
| ロール        | ``server_memory_gc_rounds`` | ``client_memory_gc_rounds`` | ``MALLOC_ARENA_MAX`` | ``cuda_empty_cache`` |
+===============+=============================+=============================+======================+======================+
| サーバ        | 5                           | N/A                         | 4                    | N/A                  |
+---------------+-----------------------------+-----------------------------+----------------------+----------------------+
| クライアント  | N/A                         | 1                           | 2                    | True (GPU の場合)    |
+---------------+-----------------------------+-----------------------------+----------------------+----------------------+

jemalloc の使用
====================

PyTorch のワークロードでは、glibc の malloc よりも jemalloc が推奨されます。NVFlare の
起動スクリプトは、``NVFLARE_ENABLE_JEMALLOC_PRELOAD=true`` によって明示的に有効化され、
かつ jemalloc が利用可能な場合にのみ jemalloc をプリロードします。

起動スクリプト
--------------------

生成される ``sub_start.sh`` スクリプトには、オプトイン方式の jemalloc プリロードが
含まれています。

.. code-block:: bash

    # Enable jemalloc preload only when opted in
    if [ "${NVFLARE_ENABLE_JEMALLOC_PRELOAD:-false}" = "true" ]; then
        for JEMALLOC in /usr/lib/x86_64-linux-gnu/libjemalloc.so.2 \
                        /usr/lib64/libjemalloc.so.2 \
                        /usr/local/lib/libjemalloc.so; do
            if [ -f "$JEMALLOC" ]; then
                export LD_PRELOAD="${LD_PRELOAD:+$LD_PRELOAD:}$JEMALLOC"
                export MALLOC_CONF="${MALLOC_CONF:-dirty_decay_ms:5000,muzzy_decay_ms:5000}"
                break
            fi
        done
    fi

jemalloc のインストール
------------------------------

.. code-block:: bash

    # Ubuntu/Debian
    apt-get install libjemalloc2

    # RHEL/CentOS
    yum install jemalloc

API リファレンス
====================

cleanup_memory
--------------

.. code-block:: python

    from nvflare.fuel.utils.memory_utils import cleanup_memory

    cleanup_memory(cuda_empty_cache=True)

**シグネチャ:** ``cleanup_memory(cuda_empty_cache: bool = False) -> None``

アロケータを考慮したメモリクリーンアップを実行します。

1. ``gc.collect()`` を実行します
2. glibc の場合: ``malloc_trim(0)`` を呼び出します
3. jemalloc の場合: auto-decay に依存します (操作は不要です)
4. 任意で ``torch.cuda.empty_cache()`` を呼び出します

get_allocator_type
------------------

.. code-block:: python

    from nvflare.fuel.utils.memory_utils import get_allocator_type

    allocator = get_allocator_type()  # "glibc", "jemalloc", or "unknown"

**シグネチャ:** ``get_allocator_type() -> str``

実行時にどのメモリアロケータが使用されているかを検出します。結果はキャッシュされます。

try_malloc_trim
---------------

.. code-block:: python

    from nvflare.fuel.utils.memory_utils import try_malloc_trim

    result = try_malloc_trim()

**シグネチャ:** ``try_malloc_trim() -> Optional[int]``

空きヒープページを OS に返却する低レベル関数です。

**戻り値:**

- メモリが解放された場合は ``1``
- 解放するメモリがない場合は ``0``
- 利用できない場合 (非 Linux または非 glibc) は ``None``

トラブルシューティング
==========================

サーバでの RSS 過大
------------------------

1. ``MALLOC_ARENA_MAX`` が設定されているか確認します
2. ``server_memory_gc_rounds=5`` を有効にします
3. jemalloc (LD_PRELOAD) の使用を検討します
4. ``top`` または ``htop`` で監視します

クライアントでの RSS 過大
------------------------------

1. ``flare.send()`` がデフォルトの ``clear_cache=True`` を使用していることを確認します (または明示的に設定します)
2. ``MALLOC_ARENA_MAX=2`` が設定されているか確認します
3. ``client_memory_gc_rounds=1`` から始めます
4. RSS がすでに安定していてパフォーマンスをチューニングする場合にのみ ``2`` または ``5`` に増やします
5. GPU の場合は ``cuda_empty_cache=True`` を有効にします
6. jemalloc の使用を検討します

OOM エラー
--------------------

1. バッチサイズを小さくします
2. ``flare.send()`` がデフォルトの ``clear_cache=True`` を使用していることを確認します — これがクライアント側の主要な対処です
3. 毎ラウンドの補助的なクリーンアップを有効にします (``client_memory_gc_rounds=1`` または ``server_memory_gc_rounds=1``)
4. 学習コードにメモリリークがないか確認します
5. 適切な decay 設定で jemalloc を使用します
