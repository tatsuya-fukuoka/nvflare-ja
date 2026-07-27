.. _hello_pt_job_api:

Job APIを使ったHello PyTorch
========================================

この例では、NVIDIA FLAREとPyTorchを使用し、連合平均(FedAvg)によって画像分類器をトレーニングする方法を示します。
完全なサンプルコードは :github_nvflare_link:`hello-ptディレクトリ <examples/hello-world/hello-pt/>` にあります。

はじめる前に
------------------------

`NVIDIA FLARE <https://pypi.org/project/nvflare/>`_ の詳細について学ぶには、
いつでも :doc:`詳細ドキュメント <../developer_guide>` を参照してください。

まず :doc:`Hello NumPy <hello_numpy>` の演習を終えることをお勧めします。この演習では
`NVIDIA FLARE <https://pypi.org/project/nvflare/>`_ の連合学習(Federated Learning)の概念を紹介しています。

NVIDIA FLAREがインストールされた環境があることを確認してください。

Python仮想環境(推奨環境)のセットアップの一般的な考え方とNVIDIA FLAREのインストール方法については、
:ref:`getting_started` を参照してください。

はじめに
----------------

この演習では、NVIDIA FLAREを人気のディープラーニングフレームワークである
`PyTorch <https://pytorch.org/>`_ と統合し、組み込みの :class:`FedAvg<nvflare.app_common.workflows.fedavg.FedAvg>` ワークフローを使用して、
CIFAR10データセットで畳み込みネットワークをトレーニングする方法を学びます。

この演習のセットアップは、1つの\ **サーバー**\ と2つの\ **クライアント**\ で構成されます。

以下のステップが、\ **ラウンド**\ と呼ばれる重み更新の1サイクルを構成します:

 #. クライアントは、自身のCIFAR10データセットを使用して、モデルの個別の重み更新を生成する役割を担います。
 #. これらの更新はサーバーに送信され、サーバーはそれらを集約して新しい重みを持つモデルを生成します。
 #. 最後に、サーバーはこの更新されたモデルを各クライアントに送り返します。

サンプルの実行
------------------------
このサンプルを実行するには:

1. リポジトリをクローンし、サンプルディレクトリに移動します:

.. code-block:: shell

   $ git clone https://github.com/NVIDIA/NVFlare.git
   $ cd NVFlare/examples/hello-world/hello-pt

2. 必要な依存関係をインストールします:

.. code-block:: shell

   $ pip install -r requirements.txt

3. サンプルを実行します:

.. code-block:: shell

   $ python job.py

このスクリプトは、NVFlareのジョブレシピを作成し、FLシミュレーターを使用して実行します。

実行中のFLシステムに提出するためにジョブフォルダをエクスポートするには、標準のRecipe APIエクスポートフラグを使用します:

.. code-block:: shell

   $ python job.py --export --export-dir /tmp/nvflare/jobs/job_config

エクスポートされたジョブは ``/tmp/nvflare/jobs/job_config/hello-pt`` に書き出されます。
エクスポートフラグは、サンプル固有のオプションと組み合わせることができます。例:

.. code-block:: shell

   $ python job.py --export --export-dir /tmp/nvflare/jobs/job_config \
       --enable_log_streaming --synthetic_data --train_size 2048 --test_size 256 \
       --num_rounds 2 --epochs 1 --batch_size 64 --num_workers 0

NVIDIA FLARE Job API
--------------------

このhello-ptサンプルの ``job.py`` スクリプトは、:class:`FedAvgRecipe<nvflare.app_opt.pt.recipes.fedavg.FedAvgRecipe>` を定義します。
このレシピは、PyTorchモデル、クライアントトレーニングスクリプト、およびシミュレーター/エクスポートの動作を組み合わせます:

.. code-block:: python

   recipe = FedAvgRecipe(
       name="hello-pt",
       min_clients=n_clients,
       num_rounds=num_rounds,
       model=SimpleNetwork(),
       train_script="client.py",
       train_args=train_args,
   )


NVIDIA FLAREクライアントトレーニングスクリプト
------------------------------------------------------------
このサンプルのトレーニングスクリプト ``client.py`` は、クライアント上で実行されるメインのスクリプトです。
トレーニングのためのPyTorch固有のロジックが含まれています。

ニューラルネットワーク
--------------------------------

トレーニング手順とネットワークアーキテクチャは、
`Training a Classifier <https://pytorch.org/tutorials/beginner/blitz/cifar10_tutorial.html>`_ を基に変更したものです。

このサンプルで使用される簡略化されたCIFAR10モデルを見てみましょう:

- :github_nvflare_link:`model.py <examples/hello-world/hello-pt/model.py>`

この ``SimpleNetwork`` クラスが、CIFAR10データセットでトレーニングする畳み込みニューラルネットワークです。
これはNVIDIA FLAREとは関係がないため、``model.py`` というファイルに実装しています。

データセットとセットアップ
--------------------------------------

実際のFL実験では、各クライアントはローカルトレーニングに使用する独自のデータセットを持ちます。
CIFAR10データセットはtorchvisionのdatasetsモジュールを介してインターネットからダウンロードできるため、
簡単のために、各クライアントでこのデータセットを使用します。
さらに、オプティマイザー、損失関数、およびデータを処理するための変換をセットアップする必要があります。
すべてのディープラーニングトレーニングには同様のセットアップがあるため、これらのコードはすべてローカルトレーニングループの一部と考えることができます。

``client.py`` スクリプトでは、``flare.init()`` の前にこれらすべてのセットアップを行います。

ローカルトレーニング
--------------------------------

ネットワークとデータセットのセットアップができたので、NVFlareのClient APIを使ってローカルトレーニングループも実装しましょう:

.. code-block:: python

   flare.init()

   summary_writer = SummaryWriter()
   while flare.is_running():
      input_model = flare.receive()

      model.load_state_dict(input_model.params)

      steps = epochs * len(train_loader)
      for epoch in range(epochs):
         running_loss = 0.0
         for i, batch in enumerate(train_loader):
               images, labels = batch[0].to(device), batch[1].to(device)
               optimizer.zero_grad()

               predictions = model(images)
               cost = loss(predictions, labels)
               cost.backward()
               optimizer.step()

               running_loss += cost.cpu().detach().numpy() / images.size()[0]

      output_model = flare.FLModel(params=model.cpu().state_dict(), meta={"NUM_STEPS_CURRENT_ROUND": steps})

      flare.send(output_model)


上記のコードは、トレーニングワークフローを実現するためのNVFlareのClient APIの3つの重要なメソッドに焦点を当てるため、
``client.py`` スクリプトを簡略化したものです:

   - `init()`: NVFlare Client API環境を初期化します。
   - `receive()`: FLサーバーからモデルを受信します。
   - `send()`: FLサーバーにモデルを送信します。

NVIDIA FLAREサーバーとアプリケーション
----------------------------------------------------
このサンプルでは、サーバーはデフォルト設定で :class:`FedAvg<nvflare.app_common.workflows.fedavg.FedAvg>` を実行します。

``python job.py --export --export-dir <job_folder>`` でジョブをエクスポートすると、
サーバーと各クライアントの構成を確認できます。サーバー構成は、エクスポートされたappフォルダ内の
configフォルダにある ``config_fed_server.json`` です:

.. code-block:: json

   {
      "format_version": 2,
      "workflows": [
         {
               "id": "controller",
               "path": "nvflare.app_common.workflows.fedavg.FedAvg",
               "args": {
                  "aggregation_weights": {},
                  "num_clients": 2,
                  "num_rounds": 2
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
               "id": "receiver",
               "path": "nvflare.app_opt.tracking.tb.tb_receiver.TBAnalyticsReceiver",
               "args": {
                  "events": [
                     "analytix_log_stats",
                     "fed.analytix_log_stats"
                  ]
               }
         },
         {
               "id": "persistor",
               "path": "nvflare.app_opt.pt.file_model_persistor.PTFileModelPersistor",
               "args": {
                  "model": {
                     "path": "model.SimpleNetwork",
                     "args": {}
                  }
               }
         },
         {
               "id": "locator",
               "path": "nvflare.app_opt.pt.file_model_locator.PTFileModelLocator",
               "args": {
                  "pt_persistor_id": "persistor"
               }
         }
      ],
      "task_data_filters": [],
      "task_result_filters": []
   }

これはJob APIによって自動的に作成されます。サーバーアプリケーション構成は、NVIDIA FLAREの組み込みコンポーネントを活用しています。

``persistor`` が ``PTFileModelPersistor`` を指していることに注意してください。これは、レシピに渡された
``SimpleNetwork`` モデルから自動的に構成されます。Job APIはモデルがPyTorchモデルであることを検出し、
:class:`PTFileModelPersistor<nvflare.app_opt.pt.file_model_persistor.PTFileModelPersistor>` と
:class:`PTFileModelLocator<nvflare.app_opt.pt.file_model_locator.PTFileModelLocator>` を自動的に構成します。


クライアント構成
--------------------------------

クライアント構成は、各クライアントのappフォルダ内のconfigフォルダにある ``config_fed_client.json`` です:

.. code-block:: json

   {
      "format_version": 2,
      "executors": [
         {
            "tasks": [
               "*"
            ],
            "executor": {
               "path": "nvflare.app_opt.pt.in_process_client_api_executor.PTInProcessClientAPIExecutor",
               "args": {
                  "task_script_path": "client.py",
                  "task_script_args": "--batch_size 16 --epochs 2 --num_workers 2"
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

``task_script_path`` には、クライアントトレーニングスクリプトのパスが設定されます。

この演習の完全なソースコードは
:github_nvflare_link:`examples/hello-world/hello-pt <examples/hello-world/hello-pt/>` にあります。

Hello PyTorchの以前のバージョン
------------------------------------------------

   - `hello-pt for 2.0 <https://github.com/NVIDIA/NVFlare/tree/2.0/examples/hello-pt>`_
   - `hello-pt for 2.1 <https://github.com/NVIDIA/NVFlare/tree/2.1/examples/hello-pt>`_
   - `hello-pt for 2.2 <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/hello-pt>`_
   - `hello-pt for 2.3 <https://github.com/NVIDIA/NVFlare/tree/2.3/examples/hello-world/hello-pt>`_
   - `hello-pt for 2.4 <https://github.com/NVIDIA/NVFlare/tree/2.4/examples/hello-world/hello-pt>`_
   - `hello-pt for 2.5 <https://github.com/NVIDIA/NVFlare/tree/2.5/examples/hello-world/hello-pt>`_
