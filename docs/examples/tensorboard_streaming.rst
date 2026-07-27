.. _tensorboard_streaming:

TensorBoard ストリーミングによるFL実験トラッキング
====================================================

はじめに
-------------

この演習では、クライアントからサーバーへ TensorBoard のイベントをストリーミングし、サーバー上の一箇所から
ライブの学習メトリクスを可視化する方法を学びます。

この演習では、advanced examples フォルダの experiment-tracking 配下にある ``tensorboard`` の例を使用します。
これは :doc:`hello_pt_job_api` に TensorBoard ストリーミングを追加したものです。

この演習のセットアップは、1つの **サーバー** と2つの **クライアント** で構成されます。

.. note::

  この演習は :doc:`hello_pt_job_api` とは異なり、``Learner`` API を ``LearnerExecutor`` とともに使用します。
  簡単に言えば、実行フローは ``LearnerExecutor`` に抽象化されており、``Learner`` クラスで必要なメソッドを実装するだけで済みます。
  これらの API の詳細については、:class:`Learner<nvflare.app_common.abstract.learner_spec.Learner>`
  および :class:`LearnerExecutor<nvflare.app_common.executors.learner_executor.LearnerExecutor>` を参照してください。


では始めましょう。:ref:`getting_started` で説明されているとおり、NVIDIA FLARE がインストールされた環境を
用意してください。まずリポジトリをクローンします。

.. code-block:: shell

  $ git clone https://github.com/NVIDIA/NVFlare.git

インストールガイドで作成した NVIDIA FLARE の Python 仮想環境を有効化することを忘れないでください。
そして、example フォルダ (NVFlare/examples/advanced/experiment-tracking/tensorboard) で必要な依存関係をインストールします。

.. code-block:: shell

  (nvflare-env) $ python3 -m pip install -r requirements.txt


設定への TensorBoard ストリーミングの追加
--------------------------------------------

この例では、ジョブ構成と TensorBoard のセットアップは次のファイルで定義されています。

- :github_nvflare_link:`job.py <examples/advanced/experiment-tracking/tensorboard/job.py>`
- :github_nvflare_link:`client.py <examples/advanced/experiment-tracking/tensorboard/client.py>`

クライアント設定の24行目にある components セクションを見てみましょう。
最初のコンポーネントは ``pt_learner`` で、初期化、学習、検証のロジックを含んでいます。
``learner_with_tb.py`` (NVFlare/examples/advanced/experiment-tracking/pt 配下) が、TensorBoard ストリーミングの変更を加える場所です。

次に :class:`TBWriter<nvflare.app_opt.tracking.tb.tb_writer.TBWriter>` があります。
これは PyTorch の SummaryWriter のシグネチャに従った一般的なメソッドをいくつか実装しています。
これにより、``pt_learner`` はメトリクスの記録とイベントの送信を簡単に行えます。

最後に :class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` があり、
ローカルイベントをフェデレーテッドイベントに変換します。
これにより ``analytix_log_stats`` イベントは fed イベント ``fed.analytix_log_stats`` に変換され、
クライアントからサーバーへストリーミングされます。

サーバー設定の components セクションには、
:class:`AnalyticsReceiver<nvflare.app_common.widgets.streaming.AnalyticsReceiver>` 型の
:class:`TBAnalyticsReceiver<nvflare.app_common.pt.tb_receiver.TBAnalyticsReceiver>` があります。

このコンポーネントはクライアントから TensorBoard のイベントを受け取り、サーバーの実行フォルダ配下の
指定されたフォルダ (デフォルトは ``tb_events``) に保存します。

受け付けるイベントタイプ ``"fed.analytix_log_stats"`` が、クライアント設定の
:class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` の出力と一致していることに注目してください。


コードへの TensorBoard ストリーミングの追加
---------------------------------------------

この演習では、TensorBoard のロギングは次のファイルに実装されています。

- :github_nvflare_link:`client.py <examples/advanced/experiment-tracking/tensorboard/client.py>`

まず、クライアント設定で定義した ``AnalyticsSender`` に TensorBoard writer を初期化する必要があります。

``LearnerExecutor`` はコンポーネントの辞書を ``initialize()`` の ``parts`` パラメータに渡します。
``parts`` 辞書のキーとして ``self.analytic_sender_id`` を使うことで、``config_fed_client.json`` で定義した
``AnalyticsSender`` コンポーネントにアクセスできます。
``self.analytic_sender_id`` のデフォルト値は ``"analytic_sender"`` ですが、
クライアント設定で定義してコンストラクタに渡すこともできます。

TensorBoard writer が ``AnalyticsSender`` に設定されたので、
``local_train()`` の中で学習メトリクスを書き込み、サーバーへストリーミングできます。

このスクリプトは ``add_scalar(tag, scalar, global_step)`` を使って学習メトリクスと検証精度を送信します。

その他のサポートされている writer のメソッドについては
:class:`AnalyticsSender<nvflare.app_common.widgets.streaming.AnalyticsSender>` で詳しく学べます。


モデルをフェデレーテッドに学習しよう！
--------------------------------------

.. |ExampleApp| replace:: tensorboard-streaming
.. include:: run_fl_system.rst


学習中の TensorBoard ダッシュボードの表示
--------------------------------------------

クライアント側では、``AnalyticsSender`` は TensorBoard の SummaryWriter として機能します。
ただし TB ファイルに書き込む代わりに、実際には ``analytix_log_stats`` 型の NVFLARE イベントを生成します。

``ConvertToFedEvent`` ウィジェットは ``analytix_log_stats`` イベントを fed イベント
``fed.analytix_log_stats`` に変換し、サーバー側に配信します。

サーバー側では、``TBAnalyticsReceiver`` が ``fed.analytix_log_stats`` イベントを処理するように構成されており、
受け取った TB データをサーバー上の適切な TB ファイル
(デフォルトは ``server/[JOB ID]/tb_events``) に書き込みます。

サーバーへストリーミングされている学習メトリクスを表示するには、次を実行します。

.. code-block:: shell

   tensorboard --logdir=poc/server/[JOB ID]/tb_events

.. note::

    サーバーがリモートマシンで実行されている場合は、ポートフォワーディングを使ってブラウザで TensorBoard ダッシュボードを表示してください。
    例:

    .. code-block:: shell

       ssh -L {local_machine_port}:127.0.0.1:6006 user@server_ip

.. attention::

   ``server/[JOB ID]`` フォルダはジョブの実行中のみ存在します。
   ジョブが終了した後は、以下で説明するように `download_job [JOB ID]` を使ってワークスペースのデータを取得してください。

.. include:: access_result.rst

.. include:: shutdown_fl_system.rst

おめでとうございます！

これで、各クライアントのライブ学習メトリクスをサーバー上の一箇所から確認できるようになりました。

この演習の完全なソースコードは
:github_nvflare_link:`examples/advanced/experiment-tracking/tensorboard <examples/advanced/experiment-tracking/tensorboard>` にあります。

TensorBoard ストリーミングの過去バージョン
--------------------------------------------

   - `tensorboard-streaming for 2.3 <https://github.com/NVIDIA/NVFlare/tree/2.3/examples/advanced/experiment-tracking/tensorboard-streaming>`_
