********************
Flower ジョブ構造
********************
Flower のプログラミング自体は FLARE/Flower 連携の対象範囲外ですが、FLARE にジョブを送信する際には
Flower のジョブ構造を十分に理解しておく必要があります。

Flower ジョブは、以下に示すように ``custom`` ディレクトリに対する特別な要件を持つ通常の FLARE ジョブです。

.. code-block:: none

    ├── flwr_pt
    │   ├── client.py   # <-- contains `ClientApp`
    │   ├── __init__.py # <-- to register the python module
    │   ├── server.py   # <-- contains `ServerApp`
    │   └── task.py     # <-- task-specific code (model, data)
    └── pyproject.toml  # <-- Flower project file

プロジェクトフォルダ
======================
Flower アプリのコードはすべて、ジョブの ``custom`` ディレクトリ内のサブフォルダに配置する必要があります。
このサブフォルダをアプリのプロジェクトフォルダと呼びます。この例では、プロジェクトフォルダの名前は ``flwr_pt`` です。
通常、このフォルダには ``server.py`` 、 ``client.py`` 、および ``__init__.py`` が含まれます。
別の構成にすることもできますが (以下の説明を参照)、Python のバージョンに関係なくプロジェクトフォルダが
確実に有効な Python パッケージとなるよう、常に ``__init__.py`` を含めることを推奨します。

Pyproject.toml
--------------
``pyproject.toml`` ファイルはジョブの ``custom`` フォルダに存在します。これはサーバーアプリとクライアント
アプリの定義および設定情報を含む重要なファイルです。この情報は Flower システムがサーバーアプリと
クライアントアプリを見つけ、アプリ固有の設定をアプリに渡すために使用されます。

以下は :github_nvflare_link:`この例 <examples/hello-world/hello-flower/flwr-pt/pyproject.toml>` から引用した ``pyproject.toml`` の例です。

.. code-block:: toml

    [build-system]
    requires = ["hatchling"]
    build-backend = "hatchling.build"

    # Tested with:
    #   flwr==1.27.0
    #   nvflare==2.8.0rc1
    #   torch==2.11.0
    #   torchvision==0.26.0
    #   tensorboard==2.20.0

    [project]
    name = "flwr-pt"
    version = "1.0.0"
    description = ""
    license = "Apache-2.0"
    dependencies = [
        "flwr>=1.26",
        "nvflare~=2.8.0rc",
        "torch",
        "torchvision",
        "tensorboard"
    ]

    [tool.hatch.build.targets.wheel]
    packages = ["."]

    [tool.flwr.app]
    publisher = "nvidia"

    [tool.flwr.app.components]
    serverapp = "flwr_pt.server:app"
    clientapp = "flwr_pt.client:app"

    [tool.flwr.app.config]
    num-server-rounds = 3
    learning-rate = 0.001
    momentum = 0.9


.. note:: pyproject.toml で定義される情報は、プロジェクトフォルダ内のコードと一致していなければならない点に注意してください。

.. note:: NVFlare が管理する Flower ジョブでは、NVFlare が SuperLink の接続情報を含むジョブスコープの
   ``$FLWR_HOME/config.toml`` を作成します。FLARE での実行にあたって、 ``pyproject.toml`` に
   ``[tool.flwr.federations]`` セクションを定義する必要はありません。

.. note:: Flower 1.26 以降のサポートには、NVFlare 2.8 のリリース候補系列 ( ``nvflare~=2.8.0rc`` )、
   または現在の ``main`` からインストールした NVFlare が必要です。リリース済みの NVFlare 2.7.x を
   使用している場合は、 ``flwr>=1.16,<1.26`` と Flower サンプルの 2.7 ブランチまたはタグを使用してください。

プロジェクト名
~~~~~~~~~~~~~~~~
プロジェクト名はプロジェクトフォルダの名前と一致させるべきですが、必須ではありません。この例では ``flwr_pt`` です。
サーバーアプリの指定

この値は次の形式で指定します。

.. code-block::

    <server_app_module>:<server_app_var_name>

ここで:

    - <server_app_module> は、サーバーアプリのコードを含むモジュールです。このモジュールは通常、プロジェクトフォルダ (この例では flwr_pt) 内の ``server.py`` として定義されます。
    - <server_app_var_name> は、<server_app_module> 内で ServerApp オブジェクトを保持する変数の名前です。この変数は通常 ``app`` として定義されます。

.. code-block:: python

    app = ServerApp(server_fn=server_fn)


クライアントアプリの指定
~~~~~~~~~~~~~~~~~~~~~~~~~~
この値は次の形式で指定します。

.. code-block::

	<client_app_module>:<client_app_var_name>

ここで:

	- <client_app_module> は、クライアントアプリのコードを含むモジュールです。このモジュールは通常、プロジェクトフォルダ (この例では flwr_pt) 内の ``client.py`` として定義されます。
	- <client_app_var_name> は、<client_app_module> 内で ClientApp オブジェクトを保持する変数の名前です。この変数は通常 ``app`` として定義されます。

.. code-block:: python

    app = ClientApp(client_fn=client_fn)


アプリの設定
~~~~~~~~~~~~~~
pyproject.toml ファイルには、 ``[tool.flwr.app.config]`` セクションでアプリの設定情報を含めることができます。
この例では、ラウンド数を定義しています。

.. code-block:: toml

    [tool.flwr.app.config]
    num-server-rounds = 3

このセクションの内容はサーバーアプリのコードに固有のものです。この例の ``server.py`` は、その使い方を示しています。

.. code-block:: python

    def server_fn(context: Context):
        # Read from config
        num_rounds = context.run_config["num-server-rounds"]

        # Define config
        config = ServerConfig(num_rounds=num_rounds)

        return ServerAppComponents(strategy=strategy, config=config)

なお、 `FlowerRecipe(..., run_config={"num-server-rounds": 5})` のようにジョブ定義を通じて `run_config` の引数を
直接渡し、 `pyproject.toml` に記載された既定値を上書きすることもできます。

シミュレーションプロファイル
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
Flower ジョブを FLARE ジョブとして送信するのではなく、Flower のシミュレーションで直接実行する場合は、
Flower の現行のシミュレーション設定メカニズムを使ってシミュレーションプロファイルを設定してください。
これは、NVFlare が ``$FLWR_HOME/config.toml`` を通じて SuperLink 接続を管理する NVFlare の実行経路とは別のものです。
