.. _hello_tf_job_api:

Job API を使った Hello TensorFlow
==================================

始める前に
----------------
`NVIDIA FLARE <https://pypi.org/project/nvflare/>`_ の詳細については、いつでも
:doc:`詳細ドキュメント <../developer_guide>` を参照してください。

`NVIDIA FLARE <https://pypi.org/project/nvflare/>`_ のフェデレーテッドラーニングの概念を紹介しているため、
まず :doc:`Hello NumPy <hello_numpy>` の演習を終えておくことを推奨します。

NVIDIA FLARE がインストールされた環境を用意してください。

Python 仮想環境 (推奨環境) のセットアップと NVIDIA FLARE のインストール方法という一般的な概念については、
:ref:`getting_started` を参照してください。

ここでは、Python 仮想環境の中に NVIDIA FLARE をすでにインストールし、
リポジトリもすでにクローンしているものとします。

はじめに
-------------
この演習を通じて、NVIDIA FLARE を人気のディープラーニングフレームワークである
`TensorFlow <https://www.tensorflow.org/>`_ と統合し、:class:`FedAvg<nvflare.app_common.workflows.fedavg.FedAvg>` ワークフローを使って
MNIST データセットで畳み込みネットワークを学習する方法を学びます。

また、フィルタ、アグリゲータ、イベントハンドラなど、いくつかの新しいコンポーネントと概念も紹介します。

この演習のセットアップは、1つの **サーバー** と2つの **クライアント** で構成されます。

次のステップが、**ラウンド** と呼ばれる重み更新の1サイクルを構成します。

 #. クライアントは、自身の MNIST データセットを使ってモデルの重み更新を個別に生成する役割を担います。
 #. これらの更新はサーバーに送られ、サーバーはそれらを集約して新しい重みを持つモデルを生成します。
 #. 最後に、サーバーはこの更新されたモデルを各クライアントに送り返します。

この演習では、examples フォルダにある ``hello-tf`` アプリケーションを使って作業します。

では始めましょう。このタスクは TensorFlow を使用するため、まず仮想環境内にライブラリをインストールしましょう。

.. code-block:: shell

  (nvflare-env) $ python3 -m pip install tensorflow

必要な依存関係がすべてインストールできたら、2つのクライアントと1つのサーバーからなるフェデレーテッドラーニングシステムを
実行する準備が整いました。今すぐ演習を実行したい場合は、Job API でジョブを構築し、FLARE Simulator でジョブを実行する
``fedavg_script_runner_hello-tf.py`` スクリプトを実行できます。

NVIDIA FLARE Job API
--------------------
この hello-tf の例の ``fedavg_script_runner_hello-tf.py`` スクリプトは、:doc:`Hello NumPy <hello_numpy>` の例の
``fedavg_script_runner_hello-numpy.py`` スクリプトや、:doc:`Hello PyTorch <hello_pt_job_api>` の例のスクリプトと
非常によく似ています。ジョブ名とクライアントスクリプト名の変更以外の唯一の違いは、サーバーの初期グローバルモデルを
定義する行です。

.. code-block:: python

   # Define the initial global model and send to server
   job.to(TFNet(), "server")


NVIDIA FLARE クライアント学習スクリプト
----------------------------------------
この例の学習スクリプト ``hello-tf_fl.py`` は、クライアント上で実行されるメインスクリプトです。学習のための TensorFlow 固有の
ロジックが含まれています。

ニューラルネットワーク
^^^^^^^^^^^^^^^^^^^^^^^
この例で使われている簡略化された MNIST モデルを見てみましょう。

- :github_nvflare_link:`model.py <examples/hello-world/hello-tf/model.py>`

この ``TFNet`` クラスは、MNIST データセットで学習する畳み込みニューラルネットワークです。
これは NVIDIA FLARE とは関係がなく、``tf_net.py`` というファイルに実装されています。

データセットとセットアップ
^^^^^^^^^^^^^^^^^^^^^^^^^^^
学習を開始する前に、データセットをセットアップする必要があります。
この演習では、``tf.keras`` の datasets モジュールを通じてインターネットからダウンロードし、
クライアントごとに別々のデータセットを作るために半分に分割します。これはあくまで例であり、実際のシナリオでは
クライアントごとに異なるデータセットを持つことになる点に注意してください。

さらに、オプティマイザと損失関数も設定する必要があります。

これらはすべて ``client.py`` の ``while flare.is_running():`` の行より前で行われます。
次を参照してください。

- :github_nvflare_link:`client.py <examples/hello-world/hello-tf/client.py>`

クライアントのローカル学習
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
クライアントのコードは、サーバーから受け取った input_model から重みを取得し、単純に :code:`self.model.fit` を実行して、
クライアントのモデルが自身のデータセットで学習されるようにします。

ローカル学習の完全な実装は次を参照してください。

- :github_nvflare_link:`client.py <examples/hello-world/hello-tf/client.py>`

ローカル学習が終わると、新しく学習された重みが
:mod:`FLModel<nvflare.app_common.abstract.fl_model>` の params に入れられて NVIDIA FLARE サーバーへ送り返されます。


NVIDIA FLARE サーバーとアプリケーション
----------------------------------------
この例では、サーバーはデフォルト設定で :class:`FedAvg<nvflare.app_common.workflows.fedavg.FedAvg>` を実行します。

:func:`export<nvflare.job_config.api.FedJob.export>` 関数でジョブをエクスポートすると、サーバーと各クライアントの
設定を確認できます。サーバーの設定は、app_server の config フォルダにある ``config_fed_server.json`` です。

.. code-block:: json

   {
      "format_version": 2,
      "workflows": [
         {
               "id": "controller",
               "path": "nvflare.app_common.workflows.fedavg.FedAvg",
               "args": {
                  "num_clients": 2,
                  "num_rounds": 3
               }
         }
      ],
      "components": [
         {
               "id": "json_generator",
               "path": "nvflare.app_common.widgets.validation_json_generator.ValidationJsonGenerator",
               "args": {}
         },
         {
               "id": "model_selector",
               "path": "nvflare.app_common.widgets.intime_model_selector.IntimeModelSelector",
               "args": {
                  "aggregation_weights": {},
                  "key_metric": "accuracy"
               }
         },
         {
               "id": "persistor",
               "path": "nvflare.app_opt.tf.model_persistor.TFModelPersistor",
               "args": {
                  "model": {
                     "path": "src.tf_net.TFNet",
                     "args": {}
                  }
               }
         }
      ],
      "task_data_filters": [],
      "task_result_filters": []
   }

これは Job API によって自動的に作成されます。サーバーアプリケーションの設定は、NVIDIA FLARE の組み込みコンポーネントを活用しています。

``persistor`` が ``TFModelPersistor`` を指していることに注目してください。これは :func:`to<nvflare.job_config.api.FedJob.to>` 関数で
モデルをサーバーに追加したときに自動的に構成されます。Job API はモデルが TensorFlow のモデルであることを検出し、
:class:`TFModelPersistor<nvflare.app_opt.tf.model_persistor.TFModelPersistor>` を自動的に構成します。


クライアントの設定
^^^^^^^^^^^^^^^^^^^^^^^^^^

クライアントの設定は、各クライアントアプリのフォルダの config フォルダにある ``config_fed_client.json`` です。

.. code-block:: json

   {
      "format_version": 2,
      "executors": [
         {
            "tasks": [
               "*"
            ],
            "executor": {
               "path": "nvflare.app_opt.tf.in_process_client_api_executor.TFInProcessClientAPIExecutor",
               "args": {
                  "task_script_path": "src/hello-tf_fl.py"
               }
            }
         }
      ],
      "components": [
         {
               "id": "event_to_fed",
               "path": "nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent",
               "args": {
                  "events_to_convert": [
                     "analytix_log_stats"
                  ]
               }
         }
      ],
      "task_data_filters": [],
      "task_result_filters": []
   }

``task_script_path`` にはクライアントの学習スクリプトのパスが設定されています。

この演習の完全なソースコードは
:github_nvflare_link:`examples/hello-tf <examples/hello-world/hello-tf>` にあります。


GPU での実行に関する注意
-------------------------

GPU を使用したい場合は、`NVIDIA TensorFlow Docker container <https://catalog.ngc.nvidia.com/orgs/nvidia/containers/tensorflow>`_ の使用を推奨します。

GPU を使ってこの例を実行する場合、デフォルトでは TensorFlow が開始時に利用可能な GPU メモリをすべて確保しようとする点に
注意することが重要です。
複数のクライアントが関わるシナリオでは、次のフラグを設定して TensorFlow がすべての GPU メモリを確保しないように
する必要があります。

.. code-block:: bash

   TF_FORCE_GPU_ALLOW_GROWTH=true TF_GPU_ALLOCATOR=cuda_malloc_async

クライアント数より多くの GPU がある場合は、1つの GPU につき1つのクライアントを実行するのが良い方法です。
これはシミュレーション時に `--gpu` 引数を使うことで実現できます。例: `nvflare simulator -n 2 --gpu 0,1 [job]`

Hello TensorFlow (旧 Hello TensorFlow 2) の過去バージョン
-----------------------------------------------------------

   - `hello-tf2 for 2.0 <https://github.com/NVIDIA/NVFlare/tree/2.0/examples/hello-tf2>`_
   - `hello-tf2 for 2.1 <https://github.com/NVIDIA/NVFlare/tree/2.1/examples/hello-tf2>`_
   - `hello-tf2 for 2.2 <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/hello-tf2>`_
   - `hello-tf2 for 2.3 <https://github.com/NVIDIA/NVFlare/tree/2.3/examples/hello-world/hello-tf2>`_
   - `hello-tf2 for 2.4 <https://github.com/NVIDIA/NVFlare/tree/2.4/examples/hello-world/hello-tf2>`_
