.. _example_applications:

########################################
サンプルアプリケーション
########################################
NVIDIA FLARE には、連合学習を始めたり特定の機能を試したりするのに役立つチュートリアルとサンプルが :github_nvflare_link:`examples ディレクトリ <examples>` にいくつも用意されています。

1. Hello World サンプル
=======================================
:github_nvflare_link:`hello_world ノートブック <examples/hello-world/hello_world.ipynb>` から実行できます。

.. toctree::
  :maxdepth: 1
  :hidden:

  examples/hello_world_examples

1.1. ワークフロー
--------------------------------

  * :ref:`Hello NumPy <hello_numpy>` - NumPy のトレーナーと FedAvg ワークフローを使用したサンプル
  * :ref:`Hello Cross-Site Validation <hello_cross_val>` - Cross Site Eval ワークフローを使用したサンプル。以前のトレーニング結果を使ったクロスサイト検証の実行方法も示します。
  * :github_nvflare_link:`Hello Cyclic Weight Transfer (GitHub) <examples/hello-world/hello-cyclic>` - ディープラーニングのトレーニングフレームワークとして TensorFlow を使用し、CyclicController ワークフローで `Cyclic Weight Transfer <https://pubmed.ncbi.nlm.nih.gov/29617797/>`_ を実装するサンプル
  * :github_nvflare_link:`Swarm Learning <examples/advanced/swarm_learning>` - Swarm Learning とクライアント制御のクロスサイト評価ワークフローを使用したサンプル。

1.2. ディープラーニング
------------------------------------------

  * :ref:`Hello PyTorch <hello_pt_job_api>` - ディープラーニングのトレーニングフレームワークとして PyTorch と FedAvg を使用した画像分類のサンプル
  * :ref:`Hello TensorFlow <hello_tf_job_api>` - ディープラーニングのトレーニングフレームワークとして TensorFlow と FedAvg を使用した画像分類のサンプル
  * :ref:`Hello HuggingFace <hello_huggingface>` - HuggingFace Client API による Qwen の SFT/PEFT


2. チュートリアルノートブック
==========================================================

  * :github_nvflare_link:`Intro to the FL Simulator <examples/tutorials/flare_simulator.ipynb>` - :ref:`fl_simulator` を使用して NVFLARE デプロイメントのローカルシミュレーションを実行し、実際の FL プロジェクトをプロビジョニングせずにアプリケーションのテストとデバッグを行う方法を示します。
  * :github_nvflare_link:`Hello FLARE API <examples/tutorials/flare_api.ipynb>` - :ref:`flare_api` のさまざまなコマンドを順に取り上げ、それぞれの構文と使い方を示します。
  * :github_nvflare_link:`NVFLARE in POC Mode <examples/tutorials/setup_poc.ipynb>` - :ref:`POCモード <poc_command>` を使用して、完全な FLARE デプロイメントの機能を1台のマシンでテストする方法を示します。
  * :github_nvflare_link:`NVFlare CLI Tutorial <examples/tutorials/nvflare_cli.ipynb>` - ローカルセットアップ、レシピ、ジョブ、システム、スタディ、プロビジョニング、デプロイメントのための現行の ``nvflare`` コマンドグループを一通り紹介します。
  * :github_nvflare_link:`Job Recipe <examples/tutorials/job_recipe.ipynb>` - 高レベル API によって連合学習ジョブの作成と実行を簡素化するジョブレシピを紹介します。
  * :github_nvflare_link:`FLARE Logging <examples/tutorials/logging.ipynb>` - さまざまなユースケースとモードに合わせて FLARE のロギングを設定する方法を説明します。

3. 連合学習アルゴリズム
====================================================

  * :github_nvflare_link:`Federated Learning with CIFAR-10 (GitHub) <examples/advanced/cifar10>` - FedAvg、FedProx、FedOpt、SCAFFOLD、準同型暗号の使用例と、トレーニング中に TensorBoard メトリクスをサーバーへストリーミングする例を含みます

  .. toctree::
    :maxdepth: 2

    examples/fl_algorithms

4. プライバシー保護アルゴリズム
============================================================
NVIDIA FLARE のプライバシー保護アルゴリズムは、ピア間でデータが送受信される際に適用できる :ref:`フィルター <filters_for_privacy>` として実装されています。

  * :github_nvflare_link:`Federated Learning with CIFAR-10 (GitHub) <examples/advanced/cifar10>` - FedAvg、FedProx、FedOpt、SCAFFOLD、準同型暗号の使用例と、トレーニング中に TensorBoard メトリクスをサーバーへストリーミングする例を含みます

5. 従来型の機械学習のサンプル
==========================================================

  * :github_nvflare_link:`Federated Linear Model with Scikit-learn (GitHub) <examples/advanced/sklearn-linear>` - 教師あり学習と教師なし学習をサポートする広く使われているオープンソースの機械学習ライブラリ `scikit-learn <https://scikit-learn.org/>`_ と NVIDIA FLARE を使用した例。
  * :github_nvflare_link:`Federated K-Means Clustering with Scikit-learn (GitHub) <examples/advanced/sklearn-kmeans>` - `scikit-learn <https://scikit-learn.org/>`_ と k-Means による NVIDIA FLARE の例。
  * :github_nvflare_link:`Federated SVM with Scikit-learn (GitHub) <examples/advanced/sklearn-svm>` - `scikit-learn <https://scikit-learn.org/>`_ と `SVM <https://scikit-learn.org/stable/modules/generated/sklearn.svm.SVC.html>`_ による NVIDIA FLARE の例。
  * :github_nvflare_link:`Federated XGBoost (GitHub) <examples/advanced/xgboost>` - ヒストグラムベースおよびツリーベースのアルゴリズムの例を含みます。ツリーベースのアルゴリズムには、バギングと巡回(cyclic)のアプローチも含まれます。垂直連合 XGBoost の例も含みます。

6. 医用画像解析
==============================

  * :github_nvflare_link:`MONAI Integration (GitHub) <integration/monai>` - 連合平均(FedAvg)と MONAI Bundle `MONAI <https://project-monai.github.io/>`_ を使用して 3D 医用画像解析モデルをトレーニングする NVIDIA FLARE の使用例

7. 連合統計
======================

  * :ref:`連合統計の概要 <federated_statistics>` - 連合統計の機能全体について説明します。
  * :github_nvflare_link:`Federated Statistics for medical imaging (Github) <examples/advanced/federated-statistics/image_stats/README.md>` - ローカルの画像ヒストグラムを収集してグローバルなデータセットのヒストグラムを計算する例。
  * :github_nvflare_link:`Federated Statistics for tabular data with DataFrame (Github) <examples/advanced/federated-statistics/df_stats/README.md>` - Pandas DataFrame からローカルの統計サマリーを収集してグローバルなデータセットの統計を計算する例。
  * :github_nvflare_link:`Federated Statistics with Monai Statistics integration for Spleen CT Image (Github) <integration/monai/examples/README.md>` - Monai 統計の統合と、連合統計のその他いくつかの機能を示す例

  .. toctree::
    :maxdepth: 1
    :hidden:

    examples/federated_statistics_overview

8. 連合サイトポリシー
========================================

  * :github_nvflare_link:`Federated Policies (Github) <examples/advanced/federated-policies/README.rst>` - 認可、リソース、データプライバシー管理のための連合サイトポリシーについて説明します
  * :github_nvflare_link:`Custom Authentication (Github) <examples/advanced/custom_authentication/README.rst>` - カスタム認証ポリシーとセキュアモードを示します。
  * :github_nvflare_link:`Job-Level Authorization (Github) <examples/advanced/job-level-authorization/README.md>` - ジョブレベルの認可ポリシーとセキュアモードを示します。
  * :github_nvflare_link:`KeyCloak Site Authentication Integration (Github) <examples/advanced/keycloak-site-authentication/README.md>` - サイト固有の認証をサポートするための KeyCloak 統合を実演します。

9. 実験トラッキング
====================================

  * :github_nvflare_link:`FL Experiment Tracking with TensorBoard Streaming <examples/advanced/experiment-tracking/tensorboard>` - :ref:`(ドキュメント) <tensorboard_streaming>` - クライアントからサーバーへの TensorBoard ストリーミングを Hello PyTorch に組み込んだ例
  * :github_nvflare_link:`FL Experiment Tracking with MLflow <examples/advanced/experiment-tracking/mlflow>` - :ref:`(ドキュメント) <experiment_tracking_mlflow>`- クライアントからサーバーへのストリーミングとともに Hello PyTorch を MLflow と統合した例
  * :github_nvflare_link:`FL Experiment Tracking with Weights and Biases <examples/advanced/experiment-tracking/wandb>` - クライアントからサーバーへの Weights and Biases のストリーミング機能とともに Hello PyTorch を統合した例。

  .. toctree::
    :maxdepth: 1
    :hidden:

    examples/tensorboard_streaming
    examples/fl_experiment_tracking_mlflow

10.  自然言語処理(NLP)
======================================

  * :github_nvflare_link:`NLP-NER (Github) <examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-8_federated_LLM_training/08.1_fed_bert/federated_nlp_with_bert.ipynb>` - `NCBI disease データセット <https://pubmed.ncbi.nlm.nih.gov/24393765/>`_ を使用した固有表現認識(NER)タスクで、`Hugging Face <https://huggingface.co/>`_ の `BERT <https://github.com/google-research/bert>`_ および `GPT-2 <https://github.com/openai/gpt-2>`_ モデル(`BERT-base-uncased <https://huggingface.co/bert-base-uncased>`_、`GPT-2 <https://huggingface.co/gpt2>`__)の両方を例示します。

11. 連合大規模言語モデル(LLM)
========================================================

  * :github_nvflare_link:`Parameter Efficient Fine Turning <integration/nemo/examples/peft>` - NeMo の PEFT 手法を利用して LLM を下流タスクに適応させる例。
  * :github_nvflare_link:`Supervised Fine Tuning (SFT) <integration/nemo/examples/supervised_fine_tuning>` - 教師ありデータで LLM の全パラメーターをファインチューニングする例。
  * :github_nvflare_link:`LLM Tuning via HuggingFace SFT Trainer <examples/advanced/llm_hf>` - LLM チューニングタスクのために HuggingFace のトレーナーと FLARE を使用する例。


12. グラフニューラルネットワーク(GNN)
==============================================================

  * :github_nvflare_link:`Protein Classification <examples/advanced/gnn>` - GraphSAGE を使用し、`PPI <http://snap.stanford.edu/graphsage/#code>`_ データセットでタンパク質分類を行う GNN の例。
  * :github_nvflare_link:`Financial Transaction Classification <examples/advanced/gnn>` - GraphSAGE を使用し、`Elliptic++ <https://github.com/git-disl/EllipticPlusPlus>`_ データセットで金融取引分類を行う GNN の例。

13. 金融アプリケーション
================================================

  * :github_nvflare_link:`Financial Application with Federated XGBoost Methods <examples/advanced/finance>` 金融データセットで不正検出を行う連合モデルをトレーニングするために、XGBoost をさまざまな方法で使用する例。
  * :github_nvflare_link:`Financial Transaction Classification <examples/advanced/gnn>` - GraphSAGE を使用し、`Elliptic++ <https://github.com/git-disl/EllipticPlusPlus>`_ データセットで金融取引分類を行う GNN の例。


サンプルとノートブックのための仮想環境のセットアップ
==============================================================================================
サンプルの依存関係をインストールする前に、仮想環境をセットアップすることを推奨します。仮想環境用の依存関係を以下でインストールします:

.. code-block:: bash

    python3 -m pip install --user --upgrade pip
    python3 -m pip install --user virtualenv


venv がインストールされたら、以下で仮想環境を作成できます:

.. code-block:: shell

    $ python3 -m venv nvflare_example

これにより、現在の作業ディレクトリに ``nvflare_example`` ディレクトリが(存在しない場合)作成され、その中に Python インタープリターのコピー、標準ライブラリ、各種サポートファイルを含むディレクトリも作成されます。


以下のコマンドを実行して virtualenv を有効化します:

.. code-block:: shell

    $ source nvflare_example/bin/activate

必要なパッケージのインストール
------------------------------------------------------------
各サンプルフォルダで、トレーニングに必要なパッケージをインストールします:

.. code-block:: bash

    pip install --upgrade pip
    pip install -r requirements.txt

(オプション)一部のサンプルには TensorBoard のイベントファイルをプロットするスクリプトが含まれています。必要な場合は、サンプルフォルダ内の追加の依存関係もインストールしてください:

.. code-block:: bash

    pip install -r plot-requirements.txt


ノートブック用の仮想環境での JupyterLab
------------------------------------------------------------------------------
ノートブックを含むサンプルを実行するには、`JupyterLab <https://jupyterlab.readthedocs.io>`_ の使用を推奨します。

仮想環境を有効化した後、JupyterLab をインストールします。

.. code-block:: bash

  pip install jupyterlab

作成した仮想環境を JupyterLab で使用できるように登録する必要がある場合は、以下でカーネルを登録できます:

.. code-block:: bash

  python -m ipykernel install --user --name="nvflare_example"

Jupyter Lab を起動します:

.. code-block:: bash

  jupyter lab .

ノートブックを開いたら、右上のドロップダウンメニューで、登録したカーネル "nvflare_example" を選択します。

サンプルアプリにおけるカスタムコード
========================================================================
NVIDIA FLARE を使用する際に、:ref:`カスタムコード <custom_code>` をクライアントで利用できるようにする方法はいくつかあります。
ほとんどの hello-* サンプルは、FL アプリケーション内の custom フォルダを使用しています。
セキュアプロビジョニングを使用する場合、アプリ内の custom フォルダの使用は :ref:`許可 <troubleshooting_byoc>` されている必要があることに注意してください。
デフォルトでは、このオプションはセキュアモードで無効になっています。一方、POCモードでは、カスタムコードはデフォルトで動作します。

対照的に、:github_nvflare_link:`CIFAR-10 <examples/advanced/cifar10>` のサンプルでは、学習コードがクライアントのシステムにすでにインストールされており、PYTHONPATH で利用可能であることを前提としています。
そのため、アプリのフォルダにはカスタムコードが含まれていません。
PYTHONPATH は、サンプルの ``run_poc.sh`` または ``run_secure.sh`` スクリプトで設定されます。
README の説明に従ってこれらのスクリプトを実行すると、学習コードがクライアントで利用可能になります。
