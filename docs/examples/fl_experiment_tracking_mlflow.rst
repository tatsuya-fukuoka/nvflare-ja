.. _experiment_tracking_mlflow:

MLflow によるFL実験トラッキング
================================

はじめに
-------------

MLflow による実験トラッキングの例では、クライアントがイベントを通じて統計情報をサーバーにストリーミングし、
サーバーがその統計情報を MLflow に書き込みます。これは :ref:`tensorboard_streaming` の例と似ていますが、
実験トラッキングのバックエンドとして MLflow を使用します。この例は advanced examples フォルダの
experiment-tracking 配下、"mlflow" ディレクトリにあります。

この演習のセットアップは、1つの **サーバー** と2つの **クライアント** で構成されます。クライアントは
:class:`MLflowWriter<nvflare.app_opt.tracking.mlflow.mlflow_writer.MLflowWriter>` を使って統計情報をイベントとして
サーバーにストリーミングし、MLflow トラッキングサーバーへデータを書き込むのはサーバーだけです
(:class:`MLflowReceiver<nvflare.app_opt.tracking.mlflow.mlflow_receiver.MLflowReceiver>` を使用)。これにより、
MLflow トラッキングサーバーとの認証や通信を扱う必要があるのはサーバーだけになり、送信データをバッファリングすることで
通信を効率化し削減できます。


では始めましょう。:ref:`getting_started` で説明されているとおり、NVIDIA FLARE がインストールされた環境を
用意してください。まずリポジトリをクローンします。

.. code-block:: shell

  $ git clone https://github.com/NVIDIA/NVFlare.git

インストールガイドで作成した NVIDIA FLARE の Python 仮想環境を有効化することを忘れないでください。

必要な依存関係をインストールします (NVFlare/examples/advanced/experiment-tracking/mlflow)。

.. code-block:: shell

  (nvflare-env) $ python3 -m pip install -r requirements.txt

実行する際は、この例のカスタムファイルを含めるように `PYTHONPATH` を設定してください (下記のパスは、カスタムファイルを
含む "pt" ディレクトリがあるディレクトリへの適切なパスに置き換えてください)。

.. code-block:: shell

  (nvflare-env) $ export PYTHONPATH=${YOUR PATH TO NVFLARE}/examples/advanced/experiment-tracking

設定へのMLflowロギングの追加
------------------------------------------------

この例では、ジョブ構成とトラッキングのセットアップはジョブスクリプトとクライアントスクリプトで定義されています。

- :github_nvflare_link:`job.py <examples/advanced/experiment-tracking/mlflow/hello-pt-mlflow/job.py>`
- :github_nvflare_link:`client.py <examples/advanced/experiment-tracking/mlflow/hello-pt-mlflow/client.py>`

クライアント設定の24行目にある components セクションを見てみましょう。
最初のコンポーネントは ``pt_learner`` で、初期化、学習、検証のロジックを含んでいます。
``learner_with_mlflow.py`` (NVFlare/examples/advanced/experiment-tracking/pt 配下) には、MLflowWriter の構文で書かれたコードが含まれています。

:class:`MLflowWriter<nvflare.app_opt.tracking.mlflow.mlflow_writer.MLflowWriter>` は mlflow の構文を模しており、メトリクストラッキングに
MLflow を使用している既存コードを使いやすくしています。ただし MLflow トラッキングサーバーに書き込む代わりに、MLflowWriter は
トラッキングする情報を含んだイベントを NVFlare 内で生成して送信します。

最後に、:class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` がローカルイベントをフェデレーテッドイベントに変換します。
これにより ``analytix_log_stats`` イベントは fed イベント ``fed.analytix_log_stats`` に変換され、クライアントからサーバーへストリーミングされます。

サーバー設定の components セクションには
:class:`MLflowReceiver<nvflare.app_opt.tracking.mlflow.mlflow_receiver.MLflowReceiver>` があります。このコンポーネントは
クライアントからイベントを受け取り、MLflow トラッキングサーバーに書き込む前に内部でバッファリングします。デフォルトの
"buffer_flush_time" は1秒ですが、MLflowReceiver のコンポーネント設定の引数として構成できます。

受け付けるイベントタイプ ``"fed.analytix_log_stats"`` が、クライアント設定の
:class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` の出力と一致していることに注目してください。


コードへのMLflowロギングの追加
-------------------------------------------

この演習では、MLflow に関するコードの追加はすべて学習スクリプトの中にあります。

- :github_nvflare_link:`client.py <examples/advanced/experiment-tracking/mlflow/hello-pt-mlflow/client.py>`

まず、クライアント設定で定義した MLflow writer を初期化する必要があります。

初期化とロギングの使い方は次のファイルで直接確認できます。

- :github_nvflare_link:`client.py <examples/advanced/experiment-tracking/mlflow/hello-pt-mlflow/client.py>`

``LearnerExecutor`` はコンポーネントの辞書を ``initialize()`` の ``parts`` パラメータに渡します。
``parts`` 辞書のキーとして ``self.analytic_sender_id`` を使うことで、``config_fed_client.json`` で定義した
``MLflowWriter`` コンポーネントにアクセスできます。
``self.analytic_sender_id`` のデフォルト値は ``"analytic_sender"`` ですが、
クライアント設定で定義してコンストラクタに渡すこともできます。

writer が ``MLflowWriter`` に設定されたので、
``local_train()`` の中で学習メトリクスを書き込み、サーバーへストリーミングできます。

このスクリプトは、ローカル学習中に MLflow トラッキング統合を通じて学習と検証のメトリクスを記録します。

MLflowWriter で現在サポートされているメソッドは
:class:`MLflowWriter<nvflare.app_opt.tracking.mlflow.mlflow_writer.MLflowWriter>` で確認できます。


モデルをフェデレーテッドに学習しよう！
--------------------------------------

.. |ExampleApp| replace:: hello-pt-mlflow
.. include:: run_fl_system.rst


MLflow UIの表示
---------------------------------
デフォルトでは、MLflow はワークスペース内の "mlruns" というディレクトリの下に実験ログのディレクトリを作成します。
たとえばサーバーのワークスペースが "/example_workspace/workspace/example_project/prod_00/server-1" にある場合、
次のコマンドで MLflow UI を起動できます。

.. code-block:: shell

   mlflow ui --backend-store-uri /example_workspace/workspace/example_project/prod_00/server-1


.. include:: access_result.rst

.. include:: shutdown_fl_system.rst

おめでとうございます！

これで、サーバーからストリーミングされた各クライアントのライブ学習メトリクスを MLflow で確認できるようになりました。

この演習の完全なソースコードは
:github_nvflare_link:`examples/advanced/experiment-tracking/mlflow <examples/advanced/experiment-tracking/mlflow>` にあります。
