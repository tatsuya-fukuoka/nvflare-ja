.. _installation:

########################
インストール
########################

このガイドでは、NVIDIA FLAREとその依存関係のインストール方法を説明します。
先に進む前に、:ref:`fl_introduction` で連合学習(Federated Learning)の基礎を理解し、
:ref:`flare_overview` を確認して、これからインストールするものを把握しておいてください。

前提条件
================
- Python 3.10以上(Python 3.13を含む、Python 3.14までテスト済み)
- pip
- Git

.. note::
   注記: nvflareのサーバーとクライアントのバージョンは一致している必要があります。バージョンをまたいだ互換性はサポートしていません。

サポートされているオペレーティングシステム
------------------------------------------------------------------------------------
- Linux
- OSX (注: tensealやopenmined.psiなど、一部のオプション依存関係には互換性がありません)

インストール方法
================================

仮想環境のセットアップ
--------------------------------------------

:ref:`containerized_deployment` を使用しない場合は、NVIDIA FLAREを仮想環境にインストールすることを強く推奨します。
このガイドでは、venvで仮想環境を作成する方法を簡単に説明します。

仮想環境とパッケージ
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Pythonの公式ドキュメントでは、仮想環境に関する基本的な考え方が説明されています。
仮想環境の作成と管理に使用されるモジュールは `venv <https://docs.python.org/3/library/venv.html>`_ と呼ばれます。
詳細はそちらを参照してください。ここでは、NVIDIA FLARE用の仮想環境に必要ないくつかの手順のみを説明します。

OSとPythonディストリビューションによっては、Pythonのvenvパッケージを別途インストールする必要がある場合があります。たとえばUbuntu
20.04では、venvで仮想環境を作成し続けるために、次のコマンドを実行する必要があります。

.. code-block:: shell

   $ sudo apt update
   $ sudo apt-get install python3-venv

venvがインストールされたら、次のコマンドで仮想環境を作成できます:

.. code-block:: shell

    $ python3 -m venv nvflare-env

これにより、現在の作業ディレクトリに ``nvflare-env`` ディレクトリが(存在しない場合)作成され、
その中にPythonインタープリターのコピー、標準ライブラリ、
各種サポートファイルを含むディレクトリも作成されます。

次のコマンドを実行して、仮想環境をアクティベートします:

.. code-block:: shell

    $ source nvflare-env/bin/activate

venv内のpipとsetuptoolsのバージョンを更新する必要がある場合があります:

.. code-block:: shell

  (nvflare-env) $ python3 -m pip install -U pip
  (nvflare-env) $ python3 -m pip install -U setuptools

安定版リリースのインストール
--------------------------------------------------------

安定版リリースは `NVIDIA FLARE PyPI <https://pypi.org/project/nvflare>`_ で入手できます:

.. code-block:: shell

  $ python3 -m pip install nvflare

オプションの依存関係
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

NVFlareは、ニーズに応じてインストールできる複数のオプション依存関係グループを提供しています:

* **HE** - 準同型暗号のサポート:

  .. code-block:: shell

     $ pip install nvflare[HE]

  .. note::

     注記: ``nvflare[HE]`` は現在Python 3.10-3.13をサポートしています。
     Python 3.14では ``tenseal`` がまだ利用できないため、HE extraはインストールされません。

* **PSI** - Private Set Intersectionのサポート:

  .. code-block:: shell

     $ pip install nvflare[PSI]

  .. note::

     注記: ``nvflare[PSI]`` は現在Python 3.10-3.13をサポートしています。
     Python 3.14では ``openmined.psi`` がまだ利用できないため、PSI extraはインストールされません。

* **PT** - PyTorchのサポート:

  .. code-block:: shell

     $ pip install nvflare[PT]

* **SKLEARN** - Scikit-learnのサポート:

  .. code-block:: shell

     $ pip install nvflare[SKLEARN]

* **TRACKING** - MLflow、Weights & Biases、TensorBoardのサポート:

  .. code-block:: shell

     $ pip install nvflare[TRACKING]

* **MONITORING** - Datadogモニタリングのサポート:

  .. code-block:: shell

     $ pip install nvflare[MONITORING]

* **CONFIG** - OmegaConf構成のサポート:

  .. code-block:: shell

     $ pip install nvflare[CONFIG]

複数のオプション依存関係を一度にインストールすることもできます:

.. code-block:: shell

  $ pip install nvflare[PT,SKLEARN,TRACKING]  # Install PyTorch, Scikit-learn, and tracking support

開発用には、すべての依存関係(macOSではHEとPSIを除く)をインストールできます:

.. code-block:: shell

  # On Linux
  $ pip install nvflare[dev]

  # On macOS
  $ pip install nvflare[dev_mac]

ソースからのインストール
------------------------------------------------

NVFlareリポジトリをクローンしてソースからインストールします(最新のナイトリー機能へのアクセスやカスタムビルドのテストに便利です):

.. code-block:: shell

  $ git clone https://github.com/NVIDIA/NVFlare.git
  $ cd NVFlare
  $ pip install -e .  # Install in editable mode

ソースからオプションの依存関係付きでインストールすることもできます:

.. code-block:: shell

  $ pip install -e ".[dev]"  # Install all development dependencies
  $ pip install -e ".[PT,SKLEARN]"  # Install specific optional dependencies

ブランチに関する注記:

* `main <https://github.com/NVIDIA/NVFlare/tree/main>`_ ブランチは、デフォルトの(不安定な)開発ブランチです
* 2.1、2.2、2.3、2.4、2.5、2.6、2.7などのブランチは各メジャーリリースのブランチであり、これらをベースに3桁目の数字でマイナーパッチを表すタグが付けられています

特定のブランチに切り替えるには:

.. code-block:: shell

  $ git switch 2.7  # Replace with desired version

Wheelのビルド
--------------------------

次の手順でNVFlareのwheelパッケージをビルドできます:

1. ビルド依存関係をインストールします:

.. code-block:: shell

  $ pip install build wheel

2. wheelをビルドします:

.. code-block:: shell

  $ python -m build

これにより、`dist/` ディレクトリにwheelファイルが作成されます。wheelファイルはpipでインストールできます:

.. code-block:: shell

  $ pip install dist/nvflare-*.whl

.. note::
   注記: wheelのビルドには、すべてのビルド依存関係がインストールされている必要があります。問題が発生した場合は、
   pip、setuptools、wheelの最新バージョンがインストールされていることを確認してください。

特定プラットフォーム向けのビルド
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

特定のプラットフォームやPythonバージョン向けにwheelをビルドするには、次の環境変数を使用できます:

.. code-block:: shell

  # For a specific Python version
  $ PYTHON=python3.14 python -m build

  # For a specific platform
  $ PLATFORM=linux_x86_64 python -m build

.. note::
   注記: プラットフォーム固有のビルドは、異なるアーキテクチャやPythonバージョンのシステムに
   wheelを配布する必要がある場合に便利です。

次のステップ
========================
インストールが完了したら:

1. :ref:`quickstart` ガイドに従って、最初の連合学習サンプルを実行します
2. :ref:`getting_started` ガイドでNVFlareのさまざまな使い方を学びます
3. :ref:`example_applications` セクションでさらに多くのサンプルを探索します
4. 本番環境の準備ができたら、:ref:`deployment_overview` でデプロイメントのガイダンスを確認します
