.. _execution_api_type:

##################################
ローカルから連合学習へ
##################################

FLARE システムでは、連合学習(Federated Learning)アルゴリズムはジョブの形式で定義されます
(詳細は :ref:`job` を参照してください)。

ジョブは、複数の「ワークフロー」と「エグゼキューター」で構成されます。

簡略化したジョブの実行フローは次のとおりです:

- ワークフローが FL クライアントに対してタスクをスケジュールする。
- 各 FL クライアントは受信したタスクを実行し、結果を送り返す。
- ワークフローは結果を受信し、完了したかどうかを判断する。
- 完了していなければ、新しいタスクをスケジュールする
- 完了していれば、ジョブ内の次のワークフローに進む。

トレーニングや計算を連合化するには、ローカルのトレーニングまたは計算ロジックを
FLARE のタスク実行の抽象化に適合させる必要があります。

タスク実行コードの記述には、完全なカスタマイズ性から容易なユーザー適応まで、
さまざまなユースケースに対応する複数の抽象化レベルを提供しています。

実行 API の種類
==================

以下は、各種類の主要な考え方とユースケースの概要です:

Client API
----------

:ref:`client_api` は、FL コードを書く最も簡単な方法を提供し、
最小限のコード変更で集中型のコードを容易に変換できます。
Client API は、データ転送に :class:`FLModel<nvflare.app_common.abstract.fl_model.FLModel>`
オブジェクトを使用し、train、validate、submit_model といった一般的なタスクをサポートします。
PyTorch Lightning を使用するオプションも利用できます。
Client API のエグゼキューターとしては、ユースケースに応じてインプロセスと外部プロセスのエグゼキューターが提供されています。

ユーザーにはまず Client API から始め、必要に応じてより特定のケース向けに
他の種類を検討することを推奨します。

ModelLearner
------------

ModelLearner API は非推奨であり、後方互換性のために残されています。
新規プロジェクトでは、:ref:`job_recipe` と :ref:`client_api` を使用してください。

:ref:`model_learner` は、FLARE 固有の概念を最小限に抑えることで、
学習ロジックの記述を簡単にするように設計されています。
:class:`ModelLearner<nvflare.app_common.abstract.model_learner.ModelLearner>` は、
トレーニングと検証のための馴染みのある学習関数を定義し、
学習情報の転送に :class:`FLModel<nvflare.app_common.abstract.fl_model.FLModel>`
オブジェクトを使用します。
ModelLearner には、ライフサイクルやロギング情報など、
いくつかの便利な機能も含まれています。

ModelLearner は、train および validate メソッドにうまく収まり、ModelLearner の
サブクラスとメソッドの構造に容易に適応できる標準的な機械学習コードを扱う場合に
最適です。

Executor
--------

:ref:`executor` は、カスタムのロジックとタスクを定義するうえで最も柔軟であり、
カスタムのエグゼキューターとコントローラーがあれば、あらゆる形の計算を実行できます。
ただし、Executor は :class:`Shareable<nvflare.apis.shareable.Shareable>`、:class:`DXO<nvflare.apis.dxo.DXO>`、
:class:`FLContext<nvflare.apis.fl_context.FLContext>` といった FLARE 固有の通信概念を
直接扱わなければなりません。
そのため、これらの概念を抽象化してユーザーが適応しやすくするために、
多くの高レベル API が Executor の上に構築されています。

総じて、Executor を書くことが最も役立つのは、高レベル API や他の定義済み Executor の
構造に収まらないタスクやロジックを実装する場合です。

サードパーティシステムとの統合
------------------------------

FLARE クライアントに容易に適応させることができない既存の ML/DL トレーニングシステム
インフラをユーザーが持っている場合があります。

:ref:`3rd_party_integration` パターンにより、FLARE システムとサードパーティの
外部トレーニングシステムをシームレスに統合できます。

:mod:`FlareAgent <nvflare.client.flare_agent>` と
:mod:`TaskExchanger <nvflare.app_common.executors.task_exchanger>` を使用することで、
任意のサードパーティシステムがタスクを受信し、結果をサーバーに提出できるように簡単にできます。

どの抽象化を使うべきかは、以下のチャートを参考に判断してください:

.. image:: ../resources/task_execution_decision_chart.png

各種類の詳細については、以下の各ページを参照してください。

.. toctree::
   :maxdepth: 1

   execution_api_type/3rd_party_integration
   execution_api_type/client_api
   execution_api_type/model_learner
   execution_api_type/executor
