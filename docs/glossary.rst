.. _glossary:

################
用語集
################
以下は、NVIDIA FLARE における用語・概念とその定義の一覧です。

Aggregator
==========
:ref:`aggregator` は、クライアントの Shareable オブジェクトでサーバーに返されたデータをサーバー上で集約するために使用するアルゴリズムを定義します。

Application (app)
=================
:ref:`Application <application>` は、クライアントとサーバーの設定、および Controller/Worker ワークフローの実装に必要なカスタムコードを定義する、名前付きのディレクトリ構造です。2.1.0 以降、実験におけるアプリのデプロイメントを管理するために :ref:`ジョブ <job>` が導入されました。

Controller
==========
:ref:`Controller <controllers>` は、FL サーバー側でタスクを実行する Worker を制御・調整する Python オブジェクトです。Controller は協調コンピューティングワークフロー全体を定義します。その制御ロジックの中で、Controller は Worker にタスクを割り当て、Worker からのタスク結果を処理します。

Events
======
:ref:`イベント <event_system>` により、:ref:`FLComponent <fl_component>` のサブクラスであるすべてのオブジェクトに動的な通知を送ることができます。すべての FLComponent はイベントハンドラーです。イベントメカニズムは pub-sub(出版-購読)のような仕組みで、データ共有のためのコンポーネント間の間接的な通信を可能にします。

Executor
========
:ref:`executor` は、FL サーバー上の Controller から受け取ったタスクを実行する FL クライアント側のコンポーネントです。例えば DL トレーニングでは、:ref:`executor` がトレーニングループを実装します。クライアント上には、異なるタスク(トレーニング、検証/評価、データ準備など)を実行するように設計された複数のエグゼキューターを配置できます。

Filter
======
:ref:`Filters <filters>` は、サーバーとクライアント間で相互に転送される Shareable オブジェクト内のデータの変換を定義するために使用されます。フィルターは、クライアントまたはサーバーのいずれかがデータを送信または受信する際に適用できます。

FLAdminAPI
==========
FLAdminAPI は、FL サーバーに admin コマンドを発行するためのレガシーな Python ラッパーでした。現在のランタイムからは削除され、:ref:`flare_api` に置き換えられています。

FLARE API
=========
:ref:`flare_api` は、FL サーバーへの admin コマンドの発行において、より良いユーザー体験を提供することを目的として FLAdminAPI を再設計したものです。

FLARE Console (previously referred to as Admin Console or Admin Client)
=======================================================================
:ref:`FLARE Console <operating_nvflare>` は、サーバーとクライアントの起動・停止とステータス確認、アプリケーションのデプロイ、FL 実験の管理など、FL スタディのオーケストレーションに使用されます。

FLComponent
===========
ほとんどのコンポーネントタイプは :ref:`FLComponent <fl_component>` のサブクラスです。特定のイベントのリッスンやデータの処理など、さまざまな目的のために独自の FLComponent サブクラスを作成できます。

FLContext
=========
:ref:`FLContext <fl_context>` は NVIDIA FLARE の主要な機能の1つで、すべての :ref:`FLComponent <fl_component>` タイプ(Controller、Aggregator、Executor、Filter、Widget など)のあらゆるメソッドから利用できます。FLContext オブジェクトには、FL 環境のコンテキスト情報、すなわちシステム全体の設定(ピア名、ジョブ ID / 実行番号、ワークスペースの場所など)が含まれます。FLContext には Engine と呼ばれる重要なオブジェクトも含まれており、これを通じてシステムが提供する重要なサービス(イベントの発火、利用可能な全クライアント名の取得、補助メッセージの送信など)にアクセスできます。

Job
===
:ref:`ジョブ <job>` には、すべてのアプリと、どのアプリをどのクライアントまたはサーバーにデプロイするかの情報、実験のリソース要件、その実験に関するすべてが含まれます。

Learnable
=========
Learnable は、サーバーが管理する連合学習アプリケーションの成果物です。DL ワークフローでは、Learnable は学習対象となる DL モデルの側面です。例えば、一般的にはモデルの構造ではなくモデル重みが Learnable の対象となります。スタディの目的に応じて、Learnable は関心のある任意のコンポーネントになり得ます。Learnable は、クライアントの Shareable オブジェクトから集約される抽象オブジェクトであり、DL 固有のものではありません。任意のモデルやオブジェクトにすることができます。Learnable は Controller ワークフローの中で管理されます。

Learner
=======
:class:`Learner <nvflare.app_common.abstract.learner_spec.Learner>` は、トレーニング固有のタスクだけに焦点を当てたクラスで、:class:`LearnerExecutor <nvflare.app_common.executors.learner_executor.LearnerExecutor>` とともに使用するコンポーネントの構築に利用できます。これにより、NVFLARE 固有の通信構造やエラーコード処理などは LearnerExecutor で処理され、トレーニングと検証のロジックに Learner で集中できます。

LearnerExecutor
===============
:class:`LearnerExecutor <nvflare.app_common.executors.learner_executor.LearnerExecutor>` は、実行フローを抽象化し、実際のトレーニング作業を :class:`Learner <nvflare.app_common.abstract.learner_spec.Learner>` に委譲する特別なタイプの Executor です。

ModelLocator
============
:class:`nvflare.app_common.np.np_model_locator.NPModelLocator` は、サーバー上にあるクロスサイト評価の対象に含めるモデルを見つけるためのコンポーネントです。

NVIDIA FLARE
============
NVIDIA FLARE は NVIDIA Federated Learning Application Runtime Environment の略で、協調コンピューティングのために設計された汎用フレームワークです。

Peer Context
============
:ref:`Peer Context <peer_context>` は、FL の参加者同士が通信する際に、メッセージの通常のペイロードに加えて送信される、メッセージ送信者のコンテキスト情報です。

Persistor
=========
何らかの状態を保存するためのコンポーネントです。:class:`LearnablePersistor<nvflare.app_common.abstract.learnable_persistor.LearnablePersistor>` は、Learnable オブジェクトの状態を保存するために FL サーバー向けに実装されるメソッドです。例えば、グローバルモデルをディスクに書き込みます。

POC mode
========
:ref:`setting_up_poc` を参照してください。

Project yaml
============
:ref:`project.yaml <project_yml>` は、プロビジョニングプロセスで使用されるファイルで、FL サーバー、FL クライアント、Admin ユーザーを含むプロジェクトの仕様と、スタートアップキットを組み立てるための :ref:`Builders <bundled_builders>` が記述されています。

Provisioning
============
:ref:`プロビジョニング <provisioning>` は、FL サーバー、FL クライアント、Admin ユーザーを含むさまざまな参加者向けのスタートアップキットを用いて、安全なプロジェクトをセットアップするプロセスです。

Roles in NVIDIA FLARE
=====================
NVIDIA FLARE の :ref:`ユーザーロール <nvflare_roles>` には、:ref:`Project Admin <project_admin_role>`、Org Admin、Lead researcher、Member researcher があり、ユーザーごとにシステム運用に関する特定の権限を設定するために使用できます。詳細は :ref:`セキュリティ概要 <security>` ページを参照してください。

Scatter and Gather Workflow
===========================
:ref:`scatter_and_gather_workflow` は、FL サーバーが FL クライアントからの結果を集約する、NVIDIA FLARE の以前のバージョンにおけるデフォルトワークフローのリファレンス実装として含まれているものです。

Shareable
=========
:ref:`Shareable <shareable>` は、2つのピア(サーバーとクライアント)間の通信です。タスクベースのやり取りにおいて、サーバーからクライアントへの Shareable は、クライアントが実行するタスクのデータを運びます。クライアントからサーバーへの Shareable は、タスク実行の結果を運びます。これを DL モデルのトレーニングに適用すると、タスクデータには通常、クライアントがトレーニングするためのモデル重みが含まれ、タスク結果にはクライアントからの更新されたモデル重みが含まれます。Shareable の概念は非常に汎用的で、タスクにとって意味のあるものであれば何でも構いません。

ShareableGenerator
==================
.. currentmodule:: nvflare.app_common.abstract.shareable_generator.ShareableGenerator

:class:`ShareableGenerator <nvflare.app_common.abstract.shareable_generator.ShareableGenerator>` は、Shareable オブジェクトとモデルオブジェクトを相互に変換するコンポーネントです。ShareableGenerator は2つのメソッドを実装しており、:meth:`learnable_to_shareable` は Learnable オブジェクトを FL クライアントに共有するデータ形式に変換し、:meth:`shareable_to_learnable` は FL クライアントからの Shareable データ(または集約された Shareable データ)を使用して Learnable オブジェクトを更新します。

Startup kit
===========
スタートアップキットはプロビジョニングプロセスの成果物であり、FL サーバー、FL クライアント、Admin クライアント間の安全な接続を確立するために必要な設定と証明書が含まれます。これらのファイルは、サーバーとクライアント間のアイデンティティと認可ポリシーを確立するために使用されます。スタートアップキットは、ロールに応じて FL サーバー、クライアント、Admin クライアントに配布されます。

Study
=====
スタディは、単一の NVFlare デプロイメント内でマルチテナントの分離を提供します。各スタディは、どのサイトが参加するか、各 admin ユーザーがどのロールを持つかを定義します。ジョブ、ジョブ一覧、およびスタディ対応のクライアント向け admin 操作は、アクティブなスタディセッションにスコープされます。``"default"`` スタディはフォールバックのセッションコンテキストであり、証明書ベースのロールの挙動を維持します。
スタディは ``project.yml`` の ``studies:`` セクションで設定され、``api_version: 4`` が必要です。
詳細は :ref:`multi_study_guide` を参照してください。

Task
====
:ref:`Task <tasks>` は、:ref:`Controller <controllers>` からクライアントの Worker に割り当てられる作業(Python コード)の単位です。タスクの割り当て方法(broadcast、send、relay)に応じて、タスクは1つ以上のクライアントで実行されます。タスクで実行されるロジックは :ref:`Executor <executor>` で定義されます。

TB Analytics Receiver
=====================
Tensorboard Analytics Receiver は、ML 実験トラッキングの一部です。NVFLARE は、Tensorboard を ML トラッキングツールとして、サーバー側の ML 実験トラッキングを実装しました。クライアント側がログを収集し、FL サーバーは Tensorboard Summary Writer を持ち、ログを Tensorboard に送信します。TB Analytics Receiver は、さまざまなクライアントからのログを受信して Tensorboard に書き込むコンポーネントです。

Worker
======
Worker はタスク(トレーニング、検証/評価、データ準備など)を実行できるものです。Worker は FL クライアント上で動作します。
