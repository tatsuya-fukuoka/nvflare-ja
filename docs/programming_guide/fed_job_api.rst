.. _fed_job_api:

##########
FedJob API
##########

FLARE の :class:`FedJob<nvflare.job_config.api.FedJob>` API を使うと、ジョブの設定を Python で定義・作成できます。

基本概念
========

* :func:`to<nvflare.job_config.api.FedJob.to>` メソッドを使って、オブジェクト (Controller、ScriptRunner、Executor、PTModel、Filter、Component など) をサーバーまたはクライアントに割り当てます。
* オブジェクトは ``add_to_fed_job`` を実装することで、自身がジョブにどのように追加されるかを定義できます。実装されていない場合はコンポーネントとして追加されます。
* :func:`export_job<nvflare.job_config.api.FedJob.export_job>` でジョブを設定としてエクスポートします。
* :func:`simulator_run<nvflare.job_config.api.FedJob.simulator_run>` でシミュレータ上でジョブを実行します。

:class:`FedJob<nvflare.job_config.api.FedJob>` API の一覧は次のとおりです。

.. list-table:: FedJob API
   :widths: 25 35 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントへのリンク
   * - to
     - オブジェクトをターゲットに割り当てます。
     - :func:`to<nvflare.job_config.api.FedJob.to>`
   * - to_server
     - オブジェクトをサーバーに割り当てます。
     - :func:`to_server<nvflare.job_config.api.FedJob.to_server>`
   * - to_clients
     - オブジェクトをすべてのクライアントに割り当てます。
     - :func:`to_clients<nvflare.job_config.api.FedJob.to_clients>`
   * - set_up_client
     - FedJob のサブクラスで使用します。クライアントのターゲットへ最初にオブジェクトを送る際に FedJob から呼び出されるセットアップ処理です。
     - :func:`set_up_client<nvflare.job_config.api.FedJob.set_up_client>`
   * - as_id
     - オブジェクトに対して生成された uuid を返します。参照された場合、そのオブジェクトはコンポーネントとして追加されます。
     - :func:`as_id<nvflare.job_config.api.FedJob.as_id>`
   * - simulator_run
     - シミュレータでジョブを実行します。
     - :func:`simulator_run<nvflare.job_config.api.FedJob.simulator_run>`
   * - export_job
     - ジョブの設定をエクスポートします。
     - :func:`export_job<nvflare.job_config.api.FedJob.export_job>`


以下は、:class:`FedJob<nvflare.job_config.api.FedJob>` API を使ってシンプルな cifar10_fedavg ジョブを作成する例です。
FedAvg の Controller と初期の PyTorch モデルをサーバーに割り当て、学習スクリプト用の ScriptExecutor をクライアントに割り当てます。
そしてシミュレータでジョブを実行します。

.. code-block:: python

  from src.net import Net

  from nvflare.app_common.widgets.intime_model_selector import IntimeModelSelector
  from nvflare.app_common.workflows.fedavg import FedAvg
  from nvflare.app_opt.pt.job_config.model import PTModel

  from nvflare.job_config.api import FedJob
  from nvflare.job_config.script_runner import ScriptRunner

  if __name__ == "__main__":
      n_clients = 2
      num_rounds = 2
      train_script = "src/cifar10_fl.py"

      # Create the FedJob
      job = FedJob(name="cifar10_fedavg")

      # Define the FedAvg controller workflow and send to server
      controller = FedAvg(
          num_clients=n_clients,
          num_rounds=num_rounds,
      )
      job.to_server(controller)

      # Define the initial global model with PTModel wrapper and send to server
      job.to_server(PTModel(Net()))

      # Add model selection widget and send to server
      job.to_server(IntimeModelSelector(key_metric="accuracy"))

      # Send ScriptRunner to all clients
      runner = ScriptRunner(
          script=train_script, script_args="f--batch_size 32 --data_path /tmp/data/site-{i}"
      )
      job.to_clients(runner)

      # job.export_job("/tmp/nvflare/jobs/job_config")
      job.simulator_run("/tmp/nvflare/jobs/workdir", n_clients=n_clients)


FedJob の初期化
===============================

:class:`FedJob<nvflare.job_config.api.FedJob>` オブジェクトは次の引数で初期化します。

* ``name`` (str): ジョブ名です。
* ``min_clients`` (int): ジョブに必要なクライアント数で、meta.json に設定されます。
* ``mandatory_clients`` (List[str]): ジョブの実行に必須のクライアントで、meta.json に設定されます。
* ``fail_fast`` (bool、デフォルトは ``False``): 開発モード用のフラグです。``True`` の場合、サーバー設定の
  ``dead_client_grace_period`` を ``0`` に設定し、すでに停止が報告されたクライアントを、デフォルトの 60 秒の
  猶予期間の後ではなく、次の監視周期 (約 0.2 秒) で切断済みと判定します。ジョブが中断されるのは、通常の
  デプロイメントポリシーに違反した場合 (稼働中のクライアントが ``min_clients`` を下回る、すべてのクライアントが
  停止する、必須クライアントが失われる) のみである点は変わりません。``min_clients`` が登録済みクライアントの
  総数と等しい典型的な開発シナリオでは、これはクライアントの障害が発生した時点で即座に中断されることを意味し、
  クラッシュを早期に発見しやすくなります。``False`` (デフォルト) の場合は、標準の猶予期間の挙動が適用されます。

例:

.. code-block:: python

  job = FedJob(name="cifar10_fedavg", min_clients=2, mandatory_clients=["site-1", "site-2"])

開発モード — いずれかのクライアントが停止したら即座に中断する:

.. code-block:: python

  job = FedJob(name="dev_job", min_clients=2, fail_fast=True)

:func:`to<nvflare.job_config.api.FedJob.to>` によるオブジェクトの割り当て
========================================================================================

特定の ``target`` に対しては :func:`to<nvflare.job_config.api.FedJob.to>` で、
サーバーに対しては :func:`to_server<nvflare.job_config.api.FedJob.to_server>` で、
すべてのクライアントに対しては :func:`to_clients<nvflare.job_config.api.FedJob.to_clients>` でオブジェクトを割り当てます。

これらの関数には次のパラメータがあり、オブジェクトの種類に応じて使用されます。

* ``obj`` (any): 割り当てるオブジェクトです。id が指定されない場合、その型に基づいてデフォルトの id が与えられます。
* ``target`` (str): (:func:`to<nvflare.job_config.api.FedJob.to>` の場合) オブジェクトの割り当て先です。"server" またはクライアント名 (例: "site-1") を指定できます。
* ``**kwargs``: オブジェクトが ``add_to_fed_job`` メソッドを実装している場合、``kwargs`` はその関数へ渡される追加の引数です。詳細は各オブジェクトのセクションを参照してください。

.. warning::

    重要: FedJob が ``obj`` に渡された引数の値を利用できるようにするには、それらの引数をコンストラクタ内で同名 (または先頭に "_" を付けた名前) のインスタンス変数として設定する必要があります。

以下では、:func:`to<nvflare.job_config.api.FedJob.to>` を使用したときに、さまざまな種類のオブジェクトがどのように扱われるかを詳しく説明します。


Controller
----------

オブジェクトがサーバーへ送られる :class:`Controller<nvflare.apis.impl.controller.Controller>` の場合、その Controller はサーバーアプリのワークフローに追加されます。

例:

.. code-block:: python

  controller = FedAvg(
      num_clients=n_clients,
      num_rounds=num_rounds,
  )
  job.to(controller, "server")

オブジェクトがクライアントへ送られる :class:`Controller<nvflare.apis.impl.controller.Controller>` の場合、その Controller はクライアント側の Controller としてクライアントアプリのコンポーネントに追加されます。
この Controller は :class:`ClientControllerExecutor<nvflare.app_common.ccwf.client_controller_executor.ClientControllerExecutor>` から利用できます。

ScriptRunner
------------

:class:`ScriptRunner<nvflare.job_config.script_runner.ScriptRunner>` はクライアントに追加でき、スクリプトの実行や起動に使用されます。
``tasks`` パラメータは、そのスクリプトが処理するタスクを指定します (デフォルトはすべてのタスクを表す "[*]" です)。

ScriptRunner の引数:

* ``script``: 実行するスクリプトです。自動的に custom フォルダに追加されます。
* ``script_args``: スクリプトの末尾に付加される引数です。
* ``launch_external_process``: ClientAPIExecutor のバックエンドを選択します。デフォルトはインプロセス
  (``False``)、外部プロセスの場合は (``True``) です。
* ``command``: 外部プロセスモードにおいて、スクリプトの前に付加されるコマンドです (デフォルトは "python3")。
* ``framework``: スクリプトに使用する :class:`FrameworkType<nvflare.job_config.script_runner.FrameworkType>` を決定します。


例:

.. code-block:: python

  # in-process: runs `__main__` of "src/cifar10_fl.py" with argv "--batch_size 32"
  in_process_runner = ScriptRunner(
      script="src/cifar10_fl.py",
      script_args="--batch_size 32"
  )
  job.to(in_process_runner, "site-1", tasks=["train"])

  # subprocess: runs `python3 -u custom/src/cifar10_fl.py --batch_size 32`
  external_process_runner = ScriptRunner(
      script="src/cifar10_fl.py",
      script_args="--batch_size 32",
      launch_external_process=True,
      command="python3 -u"
  )
  job.to(external_process_runner, "site-2", tasks=["train"])


ScriptRunner がインプロセスまたは外部プロセスのバックエンドで ``ClientAPIExecutor`` をどのように構成するかの詳細については、
:func:`add_to_fed_job<nvflare.job_config.script_runner.ScriptRunner.add_to_fed_job>` の実装を参照してください。
``pipe_connect_type`` を明示的に渡すコードや、独自の ``task_pipe`` を指定するコードは
``BaseScriptRunner`` を使用する必要があります。``ScriptRunner`` はこれらの引数を受け付けません。


Executor
--------

オブジェクトが :class:`Executor<nvflare.apis.executor.Executor>` の場合、クライアントへ送る必要があります。その Executor はクライアントアプリの executors に追加されます。
``tasks`` パラメータは、その Executor が処理するタスクを指定します (デフォルトはすべてのタスクを表す "[*]" です)。

例:

.. code-block:: python

  executor = MyExecutor()
  job.to(executor, "site-1", tasks=["train"])


リソース (str)
--------------------

オブジェクトが str の場合、外部リソースとして扱われ、custom ディレクトリに含められます。

* オブジェクトがスクリプトの場合、custom ディレクトリにコピーされます。
* オブジェクトがディレクトリの場合、そのディレクトリはフラットに custom ディレクトリへコピーされます。

例:

.. code-block:: python

  job.to("src/cifar10_fl.py", "site-1") # script
  job.to("content_dir", "site-1") # directory


Filter
------

オブジェクトが :class:`Filter<nvflare.apis.filter.Filter>` の場合、

* ユーザーは ``filter_type`` として FilterType.TASK_RESULT (Executor から Controller への流れ) または FilterType.TASK_DATA (Controller から Executor への流れ) のいずれかを指定する必要があります。
* そのフィルタは task_data_filters または task_result_filters に応じて追加され、指定された ``tasks`` に適用されます (デフォルトはすべてのタスクを表す "[*]" です)。

例:

.. code-block:: python

  pp_filter = PercentilePrivacy(percentile=10, gamma=0.01)
  job.to(pp_filter, "site-1", tasks=["train"], filter_type=FilterType.TASK_RESULT)


モデルラッパー
--------------------

モデルラッパーである :class:`PTModel<nvflare.app_opt.pt.job_config.model.PTModel>` と :class:`TFModel<nvflare.app_opt.tf.job_config.model.TFModel>` は、persistor 付きでモデルを追加するために使用します。

* :class:`PTModel<nvflare.app_opt.pt.job_config.model.PTModel>`: PyTorch のモデル (torch.nn.Module) に対して :class:`PTFileModelPersistor<nvflare.app_opt.pt.file_model_persistor.PTFileModelPersistor>` と :class:`PTFileModelLocator<nvflare.app_opt.pt.file_model_locator.PTFileModelLocator>` を追加し、追加されたこれらのコンポーネント id の辞書を返します。
* :class:`TFModel<nvflare.app_opt.tf.job_config.model.TFModel>`: TensorFlow のモデル (tf.keras.Model) に対して :class:`TFModelPersistor<nvflare.app_opt.tf.model_persistor.TFModelPersistor>` を追加し、追加された persistor の id を返します。

例:

.. code-block:: python

  component_ids = job.to(PTModel(Net()), "server")

その他の種類のモデルについては、モデルと persistor を明示的にコンポーネントとして追加できます。


コンポーネント
--------------------
これまでのいずれの種類にも該当せず、``add_to_fed_job`` も実装していないオブジェクトは、``id`` を持つコンポーネントとして追加されます。

* ``id`` はパラメータとして指定するか、自動的に割り当てられます。
* すでに使用済みの id でコンポーネントを追加した場合、id は連番が付与され (例: "component_id1"、"component_id2")、その id が返されます。
* コンポーネントは id によって他のコンポーネントを参照できます。

例:

.. code-block:: python

  job.to_server(IntimeModelSelector(key_metric="accuracy"))


:func:`as_id<nvflare.job_config.api.FedJob.as_id>` で生成された id が、追加された別のオブジェクトから参照されている場合、その参照先のオブジェクトもコンポーネントとして追加されます。
以下の例では、comp2 がサーバーに割り当てられています。comp1 は :func:`as_id<nvflare.job_config.api.FedJob.as_id>` によって comp2 から参照されているため、comp1 もコンポーネントとしてサーバーに追加されます。

例:

.. code-block:: python

  comp1 = Component1()
  comp2 = Component2(sub_component_id=job.as_id(comp1))
  job.to(comp2, "server")


add_to_fed_job
===============

obj が ``add_to_fed_job`` メソッドを実装している場合、そのメソッドが kwargs とともに呼び出されます。add_to_fed_job の実装は、追加されるオブジェクトごとに固有です。
このメソッドは次のシグネチャに従う必要があります。

.. code-block:: python

    add_to_fed_job(job, ctx, ...)

上記のセクションで扱ったオブジェクトの多くは、特別な扱いが必要であるか、関連する追加コンポーネントを加えるラッパーとして機能するため、add_to_fed_job を実装しています。

以下の表に示すように、オブジェクト開発者向けの FedJob API は、コンポーネント、Controller、Executor、Filter、リソースを追加するための関数を提供しています。
ジョブコンテキスト ``ctx`` はこれら "add_xxx" メソッドにそのまま渡せばよく、内容にアクセスする必要はありません。
さらに、check_kwargs 関数を使うと、kwargs 内の必須引数をチェックして強制できます。

.. note::

    他のコンポーネントを追加する際は、それらが他の場所で必要になる場合に備えて、追加した追加コンポーネントの id を返すのが良い習慣です。


:class:`TFModel<nvflare.app_opt.tf.job_config.model.TFModel>` の :func:`add_to_fed_job<nvflare.app_opt.tf.job_config.model.TFModel.add_to_fed_job>` の例:

.. code-block:: python

    def add_to_fed_job(self, job, ctx):
        """This method is used by Job API.

        Args:
            job: the Job object to add to
            ctx: Job Context

        Returns:
            dictionary of ids of component added
        """
        if isinstance(self.model, tf.keras.Model):  # if model, create a TF persistor
            persistor = TFModelPersistor(model=self.model)
            persistor_id = job.add_component(comp_id="persistor", obj=persistor, ctx=ctx)
            return persistor_id
        else:
            raise ValueError(
                f"Unable to add {self.model} to job with TFModelPersistor. Expected tf.keras.Model but got {type(self.model)}."
            )


.. list-table:: FedJob オブジェクト開発者向け API
   :widths: 25 35 50
   :header-rows: 1

   * - API
     - 説明
     - API ドキュメントへのリンク
   * - add_component
     - ジョブにコンポーネントを追加します。
     - :func:`add_component<nvflare.job_config.api.FedJob.add_component>`
   * - add_controller
     - ジョブに Controller オブジェクトを追加します。
     - :func:`add_controller<nvflare.job_config.api.FedJob.add_controller>`
   * - add_executor
     - ジョブに Executor を追加します。
     - :func:`add_executor<nvflare.job_config.api.FedJob.add_executor>`
   * - add_filter
     - ジョブにフィルタを追加します。
     - :func:`add_filter<nvflare.job_config.api.FedJob.add_filter>`
   * - add_resources
     - ジョブにリソースを追加します。
     - :func:`add_resources<nvflare.job_config.api.FedJob.add_resources>`
   * - check_kwargs
     - kwargs の引数をチェックします。必須の引数が欠けている場合や、想定外の引数が渡された場合はエラーを発生させます。
     - :func:`check_kwargs<nvflare.job_config.api.FedJob.check_kwargs>`


ジョブパターンの継承
============================

多くのジョブで再利用できる共通のパターンがある場合、ジョブの継承が役立ちます。

FedJob をサブクラス化する際は、__init__ の中で任意の数のオブジェクトをサーバーへ送ることができ、
:func:`set_up_client<nvflare.job_config.api.FedJob.set_up_client>` を実装することでクライアントへオブジェクトを送れます。
``set_up_client`` は、対象となるクライアントが状況によって異なるため、クライアントのターゲットへ最初にオブジェクトを送るときに FedJob から呼び出されます。

ジョブパターンの例として、:class:`FedAvgJob<nvflare.app_opt.pt.job_config.fed_avg.FedAvgJob>` を使うと FedAvg ジョブの作成を簡略化できます。
FedAvgJob は FedAvg の Controller、PTFileModelPersistor、IntimeModelSelector を自動的に追加するため、次のような記述で済みます。

.. code-block:: python

    # Model can be class instance or dict config
    # For pre-trained weights: initial_ckpt="/server/path/to/pretrained.pt"
    job = FedAvgJob(name="cifar10_fedavg", num_rounds=num_rounds, n_clients=n_clients, initial_model=Net())

ジョブパターンのさらなる例については、以下を参照してください。

* :class:`BaseFedJob<nvflare.app_opt.pt.job_config.base_fed_job.BaseFedJob>`
* :class:`FedAvgJob<nvflare.app_opt.pt.job_config.fed_avg.FedAvgJob>` (pytorch)
* :class:`FedAvgJob<nvflare.app_opt.tf.job_config.fed_avg.FedAvgJob>` (tensorflow)
* :class:`CCWFJob<nvflare.app_common.ccwf.ccwf_job.CCWFJob>`
* :class:`FlowerJob<nvflare.app_opt.flower.flower_job.FlowerJob>`

.. note::

  これらのパターンに含まれるデフォルトのコンポーネントには異なるものもあるため、各サイトで使用されるコンポーネントの
  完全な一覧については、必ずエクスポートされたジョブ設定を参照してください。


ジョブの実行
====================

シミュレータ
------------------

:func:`simulator_run<nvflare.job_config.api.FedJob.simulator_run>` を使って、``workspace`` を指定し、``n_clients``、``threads``、``gpu`` の割り当てとともにシミュレータで FedJob を実行します。

.. note::

    ``n_clients`` を設定するのは、:func:`to<nvflare.job_config.api.FedJob.to>` でクライアントを指定していない場合のみにしてください。

例:

.. code-block:: python

  job.simulator_run(workspace="/tmp/nvflare/jobs/workdir", n_clients=2, threads=2, gpu="0,1")


設定のエクスポート
------------------------
:func:`export_job<nvflare.job_config.api.FedJob.export_job>` を使うと、他のモードで使用するためにジョブの設定を ``job_root`` ディレクトリへエクスポートできます。

例:

.. code-block:: python

  job.export_job(job_root="/tmp/nvflare/jobs/job_config")

サンプル
==============

FedJob API がさまざまなアプリケーションでどのように使えるかの例については、:github_nvflare_link:`Hello World <examples/hello-world>` と :github_nvflare_link:`Job API <examples/advanced/job_api>` のサンプルを参照してください。
