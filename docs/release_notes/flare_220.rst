****************************
FLARE v2.2 の新機能
****************************

v2.2 の目標と新機能
====================================
FLARE v2.2 における主な目標は、次のとおりでした。
 - フェデレーテッドラーニングのワークフローを加速すること
 - 実世界でフェデレーテッドラーニングプロジェクトを展開する作業を簡素化すること
 - フェデレーテッドデータサイエンスをサポートすること
 - 他のプラットフォームとの統合を可能にすること

これらの目標を達成するために、以下を含む一連の重要な新しいツールと機能が開発されました。
 - FL Simulator
 - FLARE Dashboard
 - :ref:`動的プロビジョニング <dynamic_provisioning_cli>`
 - 改良された :ref:`POC（概念実証）コマンド <poc_command>`
 - :ref:`Docker Compose <docker_compose>`
 - :ref:`preflight_check`
 - サイトポリシー管理
 - Federated XGboost <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/xgboost>
 - Federated Statistics <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/federated_statistics>
 - MONAI Integration <https://github.com/NVIDIA/NVFlare/tree/2.2/integration/monai>

以下のセクションでは、これらの機能の概要を説明します。より詳細なドキュメントと使用方法については、:ref:`ユーザーガイド <user_guide>` および :ref:`プログラミングガイド <programming_guide>` を参照してください。

FL Simulator
~~~~~~~~~~~~
:ref:`FL Simulator <fl_simulator>` は、プロビジョニング済みの FL システムを明示的にデプロイすることなく、FLARE
アプリケーションをローカルでビルド、デバッグ、実行できる軽量なツールです。FL Simulator は、対話的に利用するための
CLI と、プログラムからワークフローを開発するための API の両方を提供します。クライアントは、クライアントごとに
スレッドを用いて実装されます。リソースが限られた環境で実行する場合は、単一のスレッド（または GPU）を使って複数の
クライアントを順次実行できます。これにより、限られたリソースであってもアプリケーションのスケーラビリティをテストできます。

ユーザーは Python 環境で FL Simulator を実行し、FLARE アプリケーションのコードを直接デバッグできます。デバッグを行わずに、
本番の FLARE デプロイメントと同じようにジョブをシミュレーターへ直接サブミットすることもできます。これにより、対話的な環境で
ビルド、デバッグ、テストを行い、同じアプリケーションを変更なしで本番環境にデプロイできます。

POC モードのアップグレード
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:ref:`POC（概念実証） <poc_command>` モードの利用を好む研究者のために、ローカルでサーバーとクライアントを
プロビジョニングして起動する際の使い勝手が改善されました。

FLARE Dashboard とプロビジョニング
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
:ref:`FLARE Dashboard <nvflare_dashboard_ui>` は、プロジェクト管理者がクライアント情報を事前に収集したり、通常の
``project.yml`` 設定を手動で構成したりすることなく、プロジェクトを設定してクライアントのスタートアップキットを配布できる
Web UI を提供します。プロジェクトの詳細を設定すると、クライアントシステムと FLARE Console ユーザーの
:ref:`プロビジョニング <provisioning>` がその場で行われます。Web UI により、ユーザーは登録を行い、承認され次第、
必要に応じてプロジェクトのスタートアップキットをダウンロードできます。手動でプロビジョニングしたい方のために、
プロビジョニング CLI も引き続きメインの nvflare CLI に含まれています。

.. code-block:: shell

  nvflare provision -h

CLI によるプロビジョニング方法も強化され、:ref:`動的プロビジョニング <dynamic_provisioning_cli>` が可能になりました。
これにより、既存のサイトを再プロビジョニングすることなく、新しいサイトやユーザーを追加できます。

プロビジョニングワークフローに対するこれらの拡張に加えて、ローカルデプロイの簡素化とクライアント接続のトラブルシューティングを
支援する新しいツールをいくつか提供します。1 つ目は ``docker-compose`` :ref:`ユーティリティ <docker_compose>` で、
管理者が一連のローカルスタートアップキットをプロビジョニングし、``docker-compose up`` を実行することでサーバーを起動し、
すべてのクライアントを接続できます。

また、リモートサイトが FL サーバーへ接続を試みる前に、潜在的な環境や接続の問題をトラブルシューティングできるよう、
新しい :ref:`プリフライトチェック <preflight_check>` も提供します。

.. code-block:: shell

  nvflare preflight-check -h

このコマンドは、利用可能なすべてのプロビジョニング済みパッケージ（サーバー、管理者、クライアント、オーバーシーア）を調査し、
各コンポーネント（サーバー、クライアント、オーバーシーア）間の接続、ポート、DNS、ストレージアクセスなどを確認して、
潜在的な問題を修正する方法についての提案を提供します。

フェデレーテッドデータサイエンス
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Federated XGBoost
"""""""""""""""""

XGBoost は、応用データサイエンティストが幅広い用途で利用している人気の高い機械学習手法です。FLARE v2.2 では、
クライアントのグループ間で分散 XGBoost 学習を実行する Controller と Executor を備えた、フェデレーテッド XGBoost 統合を
導入します。まずは :github_nvflare_link:`hello-xgboost の例 <examples/xgboost>` をご覧ください。

Federated Statistics
""""""""""""""""""""
フェデレーテッド学習アプリケーションを実装する前に、データサイエンティストはしばしばデータの探索、分析、特徴量エンジニアリングの
プロセスを行います。データ探索の 1 つの手法は、データセットの統計的分布を調べることです。
FLARE v2.2 では、フェデレーテッド統計オペレーター（サーバー Controller とクライアント Executor）を導入します。これらの
事前定義済みオペレーターを使って、ユーザーは各クライアントのデータセット上でローカルに計算する統計量を定義し、ワークフローの
Controller がグローバルおよび個々のサイトの統計量を含む出力 JSON ファイルを生成します。このデータを可視化することで、
クライアント群にまたがるサイト間・特徴量間のメトリクスやヒストグラムの比較が可能になります。

サイトポリシー管理とセキュリティ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

クライアント認可とセキュリティポリシーという概念自体は FLARE で新しいものではありませんが、バージョン 2.2 では
フェデレーテッドな :ref:`サイトポリシー管理 <site_policy_management>` へと移行しました。従来、認可ポリシーは
プロビジョニング時にプロジェクト管理者が定義するか、ジョブの仕様で定義されていました。フェデレーテッドサイトポリシーへの移行により、
個々のサイトが次の項目を制御できるようになります。

 - サイトのセキュリティポリシー
 - リソース管理
 - データプライバシー

これらの新しいフェデレーテッドな制御により、個々のサイトは認可ポリシー、クライアントのワークフローが利用できるリソース、
送受信トラフィックに適用されるセキュリティフィルターを完全に制御できます。

FLARE v2.2 向けの新しい :ref:`project.yml テンプレート <project_yml>` が用意されており、以前のバージョンのスタートアップキット
（古い TLS 証明書を含むもの）は再プロビジョニングが必要になります。

フェデレーテッドサイトポリシーに加えて、FLARE v2.2 ではセキュアロギングとセキュリティ監査も導入されます。セキュアロギングを
有効にすると、エラー発生時のクライアント出力が完全なトレースバックではなくファイル名と行番号のみに制限され、サイト固有の情報が
意図せずプロジェクト管理者に開示されるのを防ぎます。セキュア監査は、プロジェクト管理者が実行したすべてのアクセスとコマンドを
サイト固有のログとして記録します。

2.2.1 への移行: 注意点とヒント
----------------------------------------

クライアントとサーバー間のデータのシリアライズ/デシリアライズには Pickle をやめて FOBS を使用する
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
NVFLARE 2.1.4 より前のバージョンでは、NVFLARE は FL クライアントとサーバー間のデータ転送に Python の `pickle <https://docs.python.org/3/library/pickle.html>`_ を使用していました。
NVFLARE は現在、FLARE Object Serializer（FOBS）を使用します。コードが依然として Pickle を使用している場合、失敗が発生する可能性があります。
コードを移行する場合、またはこれが原因でエラーが発生する場合は、:github_nvflare_link:`Flare Object Serializer (FOBS) <nvflare/fuel/utils/fobs/README.rst>` を参照してください。

もう 1 つの失敗の要因は、FOBS がサポートしていないデータ型です。FOBS はデフォルトでいくつかのデータ型をサポートしていますが、そのデータ型（カスタムクラスやサードパーティのクラス）が
サポート対象の FOBS データ型に含まれていない場合は、
:github_nvflare_link:`Flare Object Serializer (FOBS) <nvflare/fuel/utils/fobs/README.rst>` の手順に従う必要があります。

基本的に、この種の問題に対処するには、次の手順を実行する必要があります。
  - 対象のデータ型に対する FobDecomposer クラスを作成します
  - クライアントとサーバー間でそのデータ型が転送される前に、新しく作成した FobDecomposer を登録します

以下の例は、:github_nvflare_link:`Flare Object Serializer (FOBS) <nvflare/fuel/utils/fobs/README.rst>` からそのまま引用したものです。

.. code-block:: python

    from nvflare.fuel.utils import fobs

    class Simple:

        def __init__(self, num: int, name: str, timestamp: datetime):
            self.num = num
            self.name = name
            self.timestamp = timestamp


    class SimpleDecomposer(fobs.Decomposer):

        @staticmethod
        def supported_type() -> Type[Any]:
            return Simple

        def decompose(self, obj) -> Any:
            return [obj.num, obj.name, obj.timestamp]

        def recompose(self, data: Any) -> Simple:
            return Simple(data[0], data[1], data[2])

データ型が使用される前に FOBS へそのデータ型を登録します。これにより、新しく作成した FOBDecomposer を登録できます。

.. code-block:: python

    fobs.register(SimpleDecomposer)

.. note::

  Decomposer は、FOBS を使用する前にサーバー側とクライアント側の両方のコードで登録する必要があります。
  登録に適した場所は、Controller や Executor のコンストラクターです。START_RUN イベントハンドラーで行うこともできます。

shareable を使用する前に FOBS でデータをシリアライズする
""""""""""""""""""""""""""""""""""""""""""""""""""""""""""""
カスタムオブジェクトを shareable に直接格納することはできません。まず FOBS を使ってシリアライズする必要があります。
custom_data がカスタム型を含んでいると仮定すると、shareable にデータを格納する方法は次のとおりです。

.. code-block:: python

    shareable[CUSTOM_DATA] = fobs.dumps(custom_data)

受信側では次のようにします。

.. code-block:: python

    custom_data = fobs.loads(shareable[CUSTOM_DATA])


.. note::

  次の方法は機能しません。

  .. code-block:: python

    shareable[CUSTOM_DATA] = custom_data

TLS 証明書の置き換え
~~~~~~~~~~~~~~~~~~~~~~~~~~
2.2.1 では認可モデルが変更されたため、以前のスタートアップキット（古い TLS 証明書を含むもの）は動作しなくなります。古いスタートアップキットを
クリーンアップし、プロジェクトを再プロビジョニングする必要があります。

新しい Project.yml テンプレートの使用
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
2.2.1 では、フェデレーテッドサイトポリシーに新しい project.yml テンプレートが必要です。:ref:`project_yml` を参照してください。

新しい local ディレクトリ
~~~~~~~~~~~~~~~~~~~~~~~~~~~~
2.2.1 では、provision コマンドは ``startup`` ディレクトリだけでなく ``local`` ディレクトリも生成します。
これまで ``project.yml`` にあったリソース割り当ては、この新しい ``local`` ディレクトリ内の ``resources.json`` ファイルに記述することが想定されており、
各サイト/クライアントが場所ごとに個別に管理する必要があります。
デフォルトのポリシーを変更したい場合は、自サイトの ``authorization.json`` および ``privacy.json`` ファイルも ``local`` ディレクトリに
配置または修正する必要があります。

デフォルトの設定は、各サイトの local ディレクトリに提供されています。

.. code-block::

    local
    ├── authorization.json.default
    ├── log.config.default
    ├── privacy.json.sample
    └── resources.json.default

これらのデフォルトは、default というサフィックスを削除し、対象サイトに合わせて設定を修正することで上書きできます。
