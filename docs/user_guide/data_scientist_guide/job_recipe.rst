
.. _job_recipe:

NVFlare ジョブレシピ
========================================

このチュートリアルでは、NVFlare のジョブレシピ(Job Recipe)を使用して、連合学習(Federated Learning)ジョブの作成と実行を簡素化する方法を説明します。
ジョブレシピは、低レベルのジョブ設定の複雑さを隠蔽し、ユーザーが気にすべき主要な引数のみを公開する、簡素化された抽象化を提供します。

.. note::
   注記: これはテクニカルプレビューです。現時点では、すべてのアルゴリズムがレシピとして実装されているわけではありません。

安定版の公開 Recipe API の仕様については、:ref:`recipe_api` を参照してください。


JobRecipe を使う動機
----------------------------------------

**Job API** は、設定ファイルを手動で編集することなく、FLARE の FL ワークフローと設定を Python で定義できる、強力で柔軟な方法を提供します。この API により従来のアプローチと比べてプロセスは簡素化されましたが、まだ十分にシンプルとは言えません。標準的なパイプラインを扱う新規ユーザーやデータサイエンティストにとって、コントローラー、エグゼキューター、ワークフローといった詳細な概念や、それらをどのように接続するかを学ぶことは不要です。

これに対処するため、NVFlare は **ジョブレシピ** という概念を導入しました。``JobRecipe`` は、以下を備えた高レベル API を提供するために設計された、簡素化された抽象化です:

* クライアント数、ラウンド数、トレーニングスクリプト、モデル定義など、データサイエンティストが気にすべき **主要な引数のみ**。
* **FedAvg** や **Cyclic Training** といった一般的な連合学習パターンに対する **一貫したエントリポイント**。
* 同じジョブをシミュレーションから本番環境まで実行できる **実行環境**。

これにより、``JobRecipe`` は、標準的なパイプラインを扱う新規ユーザーやデータサイエンティストにとっての **最初の入り口** として特に有用です:

* Job API 全体を学ぶ代わりに、レシピから始めて高レベルのパラメータ(例: ``min_clients``、``num_rounds``)のみに集中できます。
* レシピは必要なジョブ構造と実行ロジックをカプセル化しており、正しさを保証しつつ設定ミスの可能性を減らします。
* 必要に応じて、基本に慣れた後で完全な Job API のカスタマイズへと段階的に進むことができます。

モデル入力のオプション
----------------------------------------

レシピは 2 つの形式でモデル入力を受け付けます。それぞれにトレードオフがあります:

**オプション 1: クラスインスタンス(シンプルさを重視する場合に推奨)**

.. code-block:: python

   from nvflare.app_opt.pt.recipes import FedAvgRecipe
   from model import SimpleNetwork

   recipe = FedAvgRecipe(
       name="hello-pt",
       model=SimpleNetwork(),  # Instantiated model
       train_script="client.py",
       ...
   )

**オプション 2: 辞書による設定(大規模モデルに推奨)**

.. code-block:: python

   recipe = FedAvgRecipe(
       name="hello-pt",
       model={
           "class_path": "model.SimpleNetwork",
           "args": {"num_classes": 10, "hidden_dim": 256}
       },
       train_script="client.py",
       ...
   )

.. important::

   **モデルのシリアライズについて理解する**

   クラスインスタンス(例: ``SimpleNetwork()``)を渡した場合でも、NVFlare は Python オブジェクトを直接送信するわけでは **ありません**。
   代わりに、モデルはジョブ送信前に設定ファイルへと変換されます。実際のモデルは、この設定からサーバー/クライアント上で再インスタンス化されます。

   これは以下を意味します:

   * **大規模モデル**: レシピを作成するためだけに大規模モデル(例: 数十億パラメータの LLM)をインスタンス化するのは非効率です。不要なインスタンス化時間とメモリ使用を避けるため、辞書形式を使用してください。
   * **シリアライズ不可能な状態**: モデルが JSON 設定から再構築できない状態(例: ロード済みデータ、開いているファイルハンドル)を持つ場合、その状態は失われます。
   * **TensorFlow/Keras のクラスインスタンス**: クラスパスと引数からモデルを再構築できるように、ユーザー定義のサブクラス(例えば ``tf.keras.Model`` や ``tf.keras.Sequential`` のサブクラス化)を使用してください。生のインライン Keras モデルオブジェクトを渡すと、ジョブのエクスポート時に失敗する可能性があります。
   * **トレードオフ**: クラスインスタンスはより Python 的でエラーを早期に検出できます。辞書形式は大規模モデルに対してより高いパフォーマンスを発揮します。

事前学習済みチェックポイントのパス
----------------------------------------------------------------

事前学習済みモデル重みへのパスを指定するには ``initial_ckpt`` を使用します:

.. code-block:: python

   recipe = FedAvgRecipe(
       name="hello-pt",
       model=SimpleNetwork(),
       initial_ckpt="/data/models/pretrained_model.pt",  # Absolute path
       train_script="client.py",
       ...
   )

.. important::

   **チェックポイントパスの要件**

   * **絶対パスが必須**: パスは相対パスではなく絶対パス(例: ``/data/models/model.pt``)でなければなりません。
   * **ローカルに存在しなくてもよい**: チェックポイントファイルは、レシピを作成するマシン上に存在する必要は **ありません**。ジョブ実行中にモデルが実際にロードされる時点で、**サーバー** 上に存在すれば十分です。
   * **PyTorch はモデルアーキテクチャが必要**: PyTorch の場合、``initial_ckpt`` と併せて ``model``(クラスインスタンスまたは辞書設定)を指定する必要があります。PyTorch のチェックポイントには重みのみが含まれ、アーキテクチャは含まれないためです。
   * **PyTorch の更新スキーマ**: サーバー側の PyTorch モデルまたはチェックポイントが、クライアント更新として受け付けられる ``state_dict()`` のキースキーマを定義します。クライアントは自身がトレーニングしたキーのサブセットのみを返すことができますが、返されるすべてのキーはサーバースキーマに既に存在していなければなりません。クライアント側にのみ存在する新しいキーは拒否されます。
   * **TensorFlow/Keras はチェックポイント単独で使用可能**: Keras の ``.h5`` 形式や SavedModel 形式にはアーキテクチャと重みの両方が含まれるため、``initial_ckpt`` を ``model`` なしで使用できます。``model`` を指定する場合は、サブクラス化された Keras クラスのインスタンス(または辞書設定)を使用してください。

**例: 事前学習済み重みからトレーニングを再開する**

.. code-block:: python

   # PyTorch: requires both model and checkpoint
   recipe = FedAvgRecipe(
       model=SimpleNetwork(),
       initial_ckpt="/server/path/to/pretrained.pt",
       ...
   )

   # TensorFlow: checkpoint alone works (Keras saves full model)
   recipe = FedAvgRecipe(
       initial_ckpt="/server/path/to/pretrained.h5",
       framework=FrameworkType.TENSORFLOW,
       ...
   )

基本的な例
--------------------

まずは PyTorch 向けの ``FedAvgRecipe`` を使ったシンプルな例から始めましょう。このレシピは、連合平均(federated averaging)ワークフローのセットアップに伴うすべての複雑さを自動的に処理します。

``../hello-world/hello-pt/model.py`` にある既存のトレーニングネットワークとスクリプト ``client.py`` を使用してレシピを生成します:

.. code-block:: python

   import os
   import sys
   sys.path.append("../hello-world/hello-pt")

   from nvflare.app_opt.pt.recipes.fedavg import FedAvgRecipe
   from model import SimpleNetwork

   # Create a FedAvg recipe
   recipe = FedAvgRecipe(
       name="hello-pt",
       min_clients=2,
       num_rounds=3,
       model=SimpleNetwork(),
       train_script="client.py",
       train_args="--batch_size 32",
   )

   print("Recipe created successfully!")
   print(f"Recipe name: {recipe.name}")
   print(f"Min clients: {recipe.min_clients}")
   print(f"Number of rounds: {recipe.num_rounds}")

メトリクスアーティファクト
----------------------------------------------------

トレーニング集約系のレシピは、そのサーバーワークフローがラウンド単位の集約メトリクスを報告する際に、標準のメトリクスアーティファクトを書き出します。スキーマ、セキュリティ上の動作、ツールがアーティファクトを見つける方法については、:ref:`recipe_metrics_artifacts` を参照してください。

サイトごとの設定
--------------------------------

一部のレシピは、各サイトが異なる引数、スクリプト、データローダーを使用できるように、サイト名をキーとする設定を受け付けます。``set_per_site_config`` は、レシピを構築した直後、クライアント設定・ファイル・フィルター・コンポーネント・トラッキングを追加する前に呼び出してください:

.. code-block:: python

   from nvflare.recipe import SimEnv, set_per_site_config

   set_per_site_config(
       recipe,
       {
           "site-1": {"train_args": "--data_path xxx --batch_size 4"},
           "site-2": {"train_args": "--data_path yyy --batch_size 2"},
       },
   )

   env = SimEnv(clients=recipe.configured_sites())

このヘルパーはマッピングを検証して保存するだけであり、クライアントアプリを構築してから置き換えるわけではありません。レシピは、最初のクライアント対象のカスタマイズの前、またはエクスポートや実行の前に、クライアントトポロジーを一度だけ具体化します。組み込みの FedAvg 系レシピと ``FedEvalRecipe`` は、設定された各サイトに対して直接 1 つのアプリを作成します。サイトごとの設定が省略された場合は、同じ準備タイミングでデフォルトの ``@ALL`` アプリを作成します。XGBoost の bagging、horizontal、vertical の各レシピは、設定された各サイトに必要なデータローダーとエグゼキューターのコンポーネントを追加するため、クライアントのカスタマイズ、エクスポート、実行の前に設定しておく必要があります。マッピングは空であってはならず、少なくとも ``min_clients`` 個のサイトを定義する必要があります。``server`` や ``@ALL`` といった予約済みターゲットはサイト名として使用できません。

``configured_sites()`` は、設定されたトップレベルのサイト名を返します。レシピのメタデータからサイトを推測したり、どのクライアントが接続中かを示したり、本番環境への登録を検証したり、実行環境を置き換えたりするものではありません。

.. important::

   各サイトの辞書の内容はレシピ固有です。FedAvg 系レシピは
   ``train_script``、``train_args``、``launch_external_process``、``command``、
   ``framework``、``server_expected_format``、``params_transfer_type``、
   ``launch_once``、``shutdown_timeout`` をサポートします。``FedEvalRecipe`` は
   対応する ``eval_script`` と ``eval_args`` フィールドに加え、起動、コマンド、
   交換フォーマットのオーバーライドをサポートします。XGBoost 系レシピはすべての
   サイトに ``data_loader`` が必要です。bagging は ``lr_scale`` も受け付けます。

   旧来のコンストラクタ引数 ``per_site_config=...`` は互換性のために一時的に
   利用可能ですが、``FutureWarning`` を発行し、このヘルパーの動作に委譲します。
   新しいコードでは ``set_per_site_config`` を使用してください。

レシピパラメータにシークレットを入れない
--------------------------------------------------------------------------------

レシピパラメータはジョブ定義であり、シークレットの保管場所ではありません。``train_args``、``task_args``、``eval_args``、``per_site_config``、設定オーバーライド辞書、実行パラメータ、``add_client_config`` や ``add_server_config`` に渡す辞書などの値は、生成されるジョブに平文でシリアライズされる可能性があります。これらに実際のパスワード、API キー、トークン、秘密鍵、その他の認証情報を決して含めてはいけません。

レシピは、指定された値が実際のシークレットのように見える場合に ``PotentialSecretWarning`` を発行しますが、このヒューリスティックなチェックは値が安全であることを証明できません。値は実行サイト側に留めてください。サポートされているランタイム境界においてのみ、サイトの環境変数には ``secret_ref`` を、マウントされたシークレットファイルには ``secret_file_ref`` を使用してください。サポートされている場所、例、デプロイメントのガイダンスについては、:ref:`recipe_secrets` を参照してください。

レシピメタデータ
--------------------------------

生成されるジョブのメタデータをレシピから追加するには、ネストされた生成済みジョブのメタデータを直接変更するのではなく、``set_recipe_meta`` を使用します。このヘルパーは 1 回の呼び出しで 1 つの ``JobMetaKey`` メタデータエントリを設定します:

.. code-block:: python

   from nvflare.apis.job_def import JobMetaKey
   from nvflare.recipe import set_recipe_meta

   set_recipe_meta(
       recipe,
       JobMetaKey.SCOPE,
       "private",
   )
   set_recipe_meta(
       recipe,
       JobMetaKey.RESOURCE_SPEC,
       {
           "site-1": {"num_of_gpus": 1, "mem_per_gpu_in_GiB": 4},
           "site-2": {"num_of_gpus": 1, "mem_per_gpu_in_GiB": 2},
       },
   )
   set_recipe_meta(
       recipe,
       JobMetaKey.JOB_LAUNCHER_SPEC,
       {
           "site-1": {"docker": {"image": "nvflare-site1:latest"}},
           "site-2": {"docker": {"image": "nvflare-site2:latest"}},
       },
   )

設定可能なキーは、:data:`nvflare.apis.job_def.USER_SETTABLE_JOB_META_KEYS` のメンバーと完全に一致します。その他の列挙型メンバーや生の文字列は受け付けられません。各キーは特定の値の形を期待します:

* ``JobMetaKey.RESOURCE_SPEC`` (``resource_spec``): サイトごとのリソース要件 -- サイト名をキーとし、辞書を値とする辞書。
* ``JobMetaKey.JOB_LAUNCHER_SPEC`` (``launcher_spec``): サイトごとのランチャー要件 -- サイト名をキーとし、辞書を値とする辞書。
* ``JobMetaKey.SCOPE`` (``scope``): ジョブのスコープ名 -- 文字列。
* ``JobMetaKey.CUSTOM_PROPS`` (``custom_props``): ネストされたカスタムメタデータ -- 辞書。

以下の 2 グループのキーは、意図的にこのヘルパーで設定 **できない** ようになっています:

* ``FedJob`` コンストラクタに専用フィールドがあるキー -- ``min_clients`` と
  ``mandatory_clients``。これらはレシピ/``FedJob`` を構築する際に設定してください
  (例: ``FedJob(..., min_clients=2, mandatory_clients=[...])``)。そうすることで、
  コントローラー、スケジューラー、生成されるメタデータのすべてが同じ値を使用します。
  ``meta_props`` を通じて設定するとメタデータのみが変わり、レシピが既にコントローラーの
  構築に使用した値と食い違ってしまいます。
* ``study``: サーバーがジョブ送信時に管理者セッションのアクティブな study から割り当てるため、
  レシピで設定した値は暗黙的に上書きされます。study は実行環境を通じて選択してください
  (例: 後述の ``PocEnv(study=...)`` や ``ProdEnv(study=...)``)。

辞書の値は、ネストされたすべての辞書およびリストの内容を含めて JSON シリアライズ可能でなければなりません。辞書のキーは ``meta.json`` に現れる形として文字列に強制変換され、``NaN`` や ``Infinity`` のような有限でない浮動小数点値は拒否されます。このヘルパーはキーと値のペアを ``meta_props`` を通じて書き込みます。生成される ``meta.json`` にも同じキーが含まれる場合は、ジョブジェネレーターによって ``meta_props`` の値が最後に書き込まれます。

.. note::

   注記: サイトごとのリソース仕様は、基盤となる生成済みジョブ側にも存在する可能性があります
   (低レベルのジョブオブジェクトの ``add_resource_spec`` を通じて登録されるもので、これは
   内部的なパスです -- レシピスクリプトでは ``set_recipe_meta`` を優先してください。
   :ref:`recipe_api` を参照)。``set_recipe_meta`` を通じて ``RESOURCE_SPEC`` を設定した場合、
   ``meta_props`` の値が生成される ``meta.json`` 内のサイトごとの仕様を置き換えます。
   ヘルパー呼び出し時点で既に登録されている仕様に対しては警告が発行されますが、その後に
   追加された仕様は警告なしに上書きされます。

同じキーが ``meta_props`` に既に存在する場合、``set_recipe_meta`` はその値を置き換えます。

このヘルパーは、実行時のリソース可用性、本番環境への登録、メタデータで指定されたサイトが実行時に存在するかどうかを検証しません。どのサイトが存在するかは、引き続き実行環境とデプロイメントによって決まります。

完全な本番環境の例については、:github_nvflare_link:`Kubernetes クライアント上の Recipe ジョブ <examples/advanced/recipe-k8s>` を参照してください。この例では ``ProdEnv`` を使用して、別々の Kubernetes クラスターにある ``site-1`` と ``site-2`` に PyTorch CIFAR-10 ジョブを送信し、GPU 要件を ``resource_spec`` に、クラスターごとのジョブイメージとコンテナ設定を ``launcher_spec`` に記述しています。

実行環境
----------------

**ジョブレシピ** は連合学習において *何を* 実行するかを定義しますが、*どこで* 実行するかも知る必要があります。NVFlare は、同じレシピを異なるコンテキストで実行できるようにする、いくつかの **実行環境** を提供します:

* **シミュレーション(** ``SimEnv`` **)** – 単一マシン上または 1 つのバッチジョブ内でのローカルテストと実験用
* **概念実証(** ``PocEnv`` **)** – 単一マシン上で実際のデプロイメントを模倣する、小規模なマルチプロセスセットアップ用
* **本番環境(** ``ProdEnv`` **)** – 複数の組織・サイトにまたがるフルスケールの分散デプロイメント用

この分離により、コアとなるジョブ定義を変更することなく、**一度プロトタイプを作ればどこでもデプロイできる** ようになります。

SimEnv – シミュレーション環境
----------------------------------------------------------

ローカルの FL シミュレーターバックエンドでジョブを実行します。プロビジョニングされたプロジェクトや、常駐のサーバー/クライアントデーモンは不要です。シミュレートされたクライアントはローカルのワーカープロセスを使用します。``num_threads`` は、ワーカープロセスの並行数を表す歴史的な名前です。以下の用途に最適です:

* 素早い実験
* スクリプトとモデルのデバッグ
* 教育用途
* 送信した 1 つのジョブで連合ワークフロー全体を実行して終了する、バッチスケジュールされた実験

**引数:**

* ``num_clients`` (int): シミュレートするクライアントの数
* ``clients``: クライアント名のリスト(両方指定する場合、長さは ``num_clients`` と一致する必要があります)
* ``num_threads``: 並行して実行するシミュレートクライアントのワーカープロセス数
* ``gpu_config`` (str): GPU デバイス ID のリスト(カンマ区切り)
* ``log_config`` (str): ログ設定モード(``'concise'``、``'full'``、``'verbose'``)、ファイルパス、またはレベル
* ``workspace_root`` (str): シミュレーションアーティファクトのルートディレクトリ。デフォルトは ``/tmp/nvflare/simulation``

.. note::

   注記: ``NVFLARE_SIMULATOR_WORKSPACE_ROOT`` は、プロセスレベルのオーケストレーション用
   オーバーライドです。これが設定されている場合、``SimEnv`` は ``workspace_root``
   (コンストラクタで明示的に指定された値を含む)の代わりにこれを使用します。Auto-FL は、
   並行するシミュレーター実行がアーティファクトを共有しないように、各トライアルの
   子プロセス内でのみこのオーバーライドを使用します。オーバーライドによって設定済みの
   パスが変更される場合、``SimEnv`` は ``RuntimeWarning`` を発行します。通常の Recipe
   アプリケーションではこれを未設定のままにし、``workspace_root`` を直接設定してください。

それでは、準備したレシピを ``SimEnv`` で実行してみましょう:

.. code-block:: python

   from nvflare.recipe.sim_env import SimEnv
   # Create a simulation environment
   env = SimEnv(
       num_clients=2, 
       num_threads=2,
   )
   # Execute the recipe
   run = recipe.execute(env=env)
   run.get_status()
   run.get_result()

結果は ``/tmp/nvflare/simulation/hello-pt`` の下に保存されます。

PocEnv – 概念実証環境
------------------------------------------

サーバーとクライアントを同一マシン上の **別々のプロセス** として実行します。これは、サーバーとクライアントが異なるプロセスで動作する形で、単一ノード内で実際のデプロイメントをシミュレートします。``SimEnv`` より現実に近い一方で、単一ノードで動かせる程度に軽量です。

以下の用途に最適です:

* デモンストレーション
* 本番デプロイメント前の小規模な検証
* オーケストレーションロジックのデバッグ

**引数:**

* ``num_clients`` (int, オプション): POCモードで使用するクライアント数。デフォルトは 2。
* ``clients`` (List[str], オプション): クライアント名のリスト。``None`` の場合、``site-1``、``site-2`` などが生成されます。
* ``gpu_ids`` (List[int], オプション): クライアントに割り当てる GPU ID のリスト。``None`` の場合、CPU のみを使用します。
* ``auto_stop`` (bool, オプション): ジョブ完了後に POC サービスを自動的に停止するかどうか。
* ``use_he`` (bool, オプション): HE(準同型暗号)を使用するかどうか。デフォルトは ``False``。
* ``docker_image`` (str, オプション): deploy Docker 準備パスで用意された Docker POCモード用の SP/CP Docker イメージ。このモードで送信されるジョブは、``launcher_spec`` に SJ/CJ の Docker イメージを指定する必要があります。
* ``project_conf_path`` (str, オプション): プロジェクト設定ファイルへのパス。
* ``study`` (str, オプション): この実行環境の study コンテキスト。ジョブはこの study 内で送信・監視されます。デフォルトは ``"default"``。名前付きの study を使うには、``project_conf_path`` が ``api_version: 4`` と ``studies:`` を持つプロジェクトを指している必要があります。:ref:`multi_study_guide` を参照してください。

まず POC 環境へのパスを設定しましょう:

.. code-block:: shell

   %env NVFLARE_POC_WORKSPACE=/tmp/nvflare/poc

.. code-block:: python

   from nvflare.recipe.poc_env import PocEnv

   # Create a POC environment
   env = PocEnv(
       num_clients=2
   )
   # Execute the recipe
   run = recipe.execute(env=env)
   run.get_status()
   run.get_result()

結果はディレクトリ ``/tmp/nvflare/poc`` の下に保存されます。

名前付きの study を使用するには、``studies:`` を定義したカスタムプロジェクトファイルを ``PocEnv`` に指定します:

.. code-block:: python

   env = PocEnv(
       num_clients=2,
       project_conf_path="/tmp/nvflare/poc_project.yml",
       study="cancer-research"  # omit for the default study
   )

``project_conf_path`` が指定されていない場合、またはプロジェクトが ``studies:`` を定義していない場合、POC デプロイメントはシングルテナントとして動作し、``default`` study のみが有効です。

ProdEnv – 本番環境
------------------------------------

サーバーとクライアントからなるシステムが、**複数のマシンとサイト** にまたがって稼働していることを前提とします。この環境は、セキュアな通信チャネルと実際の NVFlare デプロイメントインフラを使用します。``ProdEnv`` は管理者のスタートアップパッケージを利用して既存の NVFlare システムと通信し、ジョブの実行と監視を行います。

以下の用途に最適です:

* エンタープライズ向け連合学習デプロイメント
* 複数機関のコラボレーション
* 本番スケールのワークロード

**引数:**

* ``startup_kit_location`` (str): 管理者のスタートアップキット(nvflare のプロビジョニングによって生成)を含むディレクトリ
* ``login_timeout`` (float): 管理者がシステムにログインする際のタイムアウト値
* ``monitor_job_duration`` (int): ジョブ実行を監視する時間。``None`` は監視を行わないことを意味します
* ``study`` (str): この実行環境の study コンテキスト。ジョブはこの study 内で送信・監視されます。デフォルトは ``"default"``。:ref:`multi_study_guide` を参照してください。

まずスタートアップキットをプロビジョニングしましょう:

.. code-block:: shell

   !nvflare provision -p project.yml -w /tmp/nvflare/prod_workspaces

次に、すべての参加者を起動します(以下のスクリプトをノートブック内で直接実行するのではなく、ターミナルから実行してください):

.. code-block:: shell

   bash /tmp/nvflare/prod_workspaces/example_project/prod_00/start_all.sh

それでは、環境の作成とレシピの実行に進みましょう。

.. code-block:: python

   from nvflare.recipe.prod_env import ProdEnv
   import os
   import sys
   sys.path.append("../hello-world/hello-pt")

   from nvflare.app_opt.pt.recipes.fedavg import FedAvgRecipe
   from model import SimpleNetwork

   # Create a FedAvg recipe
   recipe = FedAvgRecipe(
       name="hello-pt",
       min_clients=2,
       num_rounds=3,
       model=SimpleNetwork(),
       train_script="client.py",
       train_args="--batch_size 32",
   )
   # Create a Prod environment
   env = ProdEnv(
       startup_kit_location="/tmp/nvflare/prod_workspaces/example_project/prod_00/admin@nvidia.com",
       study="cancer-research"  # omit for the default study
   )
   # Execute the recipe
   run = recipe.execute(env=env)
   run.get_status()
   run.get_result()

環境抽象化の利点
--------------------------------

* **一貫性** – 一度定義したレシピを、変更することなくすべての環境で再利用できます。
* **段階的なワークフロー** – プロトタイピングは ``SimEnv`` で始め、検証は ``PocEnv`` に移行し、最終的に ``ProdEnv`` でデプロイします。
* **スケーラビリティ** – 同じトレーニングロジックが、ラップトップでの実験からグローバルな本番デプロイメントまでスケールします。

エッジアプリケーションに関する特記事項
------------------------------------------------------------------------------

新しい階層型システムで動作するエッジアプリケーションはシミュレーターではサポートされておらず、現行バージョンでは ``ProdEnv`` で実行する必要があります。より詳細な例は `こちら <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ を参照してください。特に、エッジレシピの準備と実験的な実行については `この例 <https://github.com/NVIDIA/NVFlare/blob/main/examples/advanced/edge/jobs/pt_job_adv.py>`_ を参照してください。

ベストプラクティス
------------------------------------

1. **開発は** ``SimEnv`` **で行い**、素早くイテレーションする。
2. **検証は** ``PocEnv`` **で行い**、マルチプロセスのオーケストレーションをテストする。
3. **デプロイは** ``ProdEnv`` **で行い**、実際の連合学習を実行する。
4. カスタマイズの前に、基本的なレシピで **シンプルに始める**。
5. レシピと実験には **一貫した命名** を使う。
6. 連合学習のプロセスを理解するために **実行を監視する**。

まとめ
------------

ジョブレシピと実行環境の組み合わせは、連合学習ジョブの定義と実行のための **統一された抽象化** を提供します:

* **レシピはトレーニングをどのように進めるかを定義します**\ (例: FedAvg、FedOpt、Swarm Learning)
* **環境はジョブをどこでどのように実行するかを定義します**\ (シミュレーション、概念実証、本番環境)

この分離により、同じレシピがコード変更なしに **ローカルテスト** から **エンタープライズスケールの本番環境** へシームレスに移行できます。

ジョブレシピの目標は、標準的な FL パイプラインを実行する新規ユーザーやデータサイエンティストにとって最も直感的な、NVFlare へのシンプルな入り口を作りつつ、より複雑でカスタマイズ可能なワークフローへの発展の余地を残すことです。

例
------
ジョブレシピの実際の使用例をさらに見るには、クイックスタートシリーズ :ref:`quickstart` を確認してください。いくつかのジョブレシピがデモンストレーションされています。
