.. _hello_cross_val:

Hello Cross-Site Validation
===========================

始める前に
----------------

このガイドに進む前に、`NVIDIA FLARE <https://pypi.org/project/nvflare/>`_ がインストールされた環境が
用意されていることを確認してください。

Python 仮想環境 (推奨環境) のセットアップと NVIDIA FLARE のインストール方法という一般的な概念については、
:ref:`getting_started` を参照してください。

前提条件
-------------

この例は、:class:`ScatterAndGather<nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather>` ワークフローに基づく
:doc:`Hello NumPy <hello_numpy>` の例を土台にしています。

概念が密接に関連しているため、必ず最後まで目を通しておいてください。

はじめに
-------------

このチュートリアルは、実際のディープラーニングの概念を導入することなく、NVIDIA FLARE システムがどのように動作するかを
示すことだけを目的としています。

この演習を通じて、学習後にクロスサイト検証を実行するために NVIDIA FLARE を numpy と組み合わせて使う方法を学びます。

学習プロセスについては :doc:`Hello NumPy <hello_numpy>` の例で説明しています。

簡略化された重みとメトリクスを使うことで、NVIDIA FLARE がわずかな追加作業だけで異なるサイト間の検証を
どのように実行するかを明確に確認できます。

この演習のセットアップは、1つの **サーバー** と2つの **クライアント** で構成されます。
サーバー側のモデルは重み ``[[1, 2, 3], [4, 5, 6], [7, 8, 9]]`` から始まります。

クロスサイト検証は次のステップで構成されます。

    - :class:`ScatterAndGather<nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather>` ワークフローによる
      学習の初期フェーズで、NPTrainer がクライアントのローカルモデルをディスクに保存します。
    - :class:`CrossSiteModelEval<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>` ワークフローが
      ``submit_model`` タスクでクライアントのモデルを取得します。
    - モデルデータを含む model shareable とともに ``validate`` タスクが参加している全クライアントにブロードキャストされ、
      ``validate`` タスクの結果が保存されます。

この演習では、NVIDIA FLARE が上記のほとんどのステップをユーザーのわずかな作業だけで処理してくれることを確認します。
examples フォルダにある ``hello-numpy-cross-val`` アプリケーションを使って作業します。
カスタム FL アプリケーションは次のフォルダを含むことができます。

 #. **custom**: カスタムコンポーネント (``np_trainer.py``、``np_model_persistor.py``、``np_validator.py``、``np_model_locator``、``np_formatter``) を含みます
 #. **config**: クライアントとサーバーの設定 (``config_fed_client.json``、``config_fed_server.json``) を含みます
 #. **resources**: ロガー設定 (``log_config.json``) を含みます

では始めましょう。まだクローンしていない場合は、まずリポジトリをクローンします。

.. code-block:: shell

  $ git clone https://github.com/NVIDIA/NVFlare.git

インストールガイドで作成した NVIDIA FLARE の Python 仮想環境を有効化することを忘れないでください。
numpy がインストールされていることを確認します。

.. code-block:: shell

  (nvflare-env) $ python3 -m pip install numpy

必要な依存関係がすべてインストールできたので、フェデレーテッドラーニングシステムを実装しましょう。


学習
--------------------------------

:doc:`Hello NumPy <hello_numpy>` の例では、``NPTrainer`` オブジェクトを実装しました。
この例では同じ ``NPTrainer`` を使いますが、クライアントのモデルを取得する
:class:`CrossSiteModelEval<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>`
ワークフローと連携できるよう、``submit_model`` タスクを処理するように拡張します。

``np_trainer.py`` のコードは、モデルの学習の各ステップの後にモデルをディスクに保存します。

サーバーもグローバルモデルを生成することに注意してください。
:class:`CrossSiteModelEval<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>`
ワークフローは、クライアントのモデルの後にサーバーのモデルを評価のために提出します。

Validatorの実装
--------------------------

Validator は Executor であり、:class:`CrossSiteModelEval<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>`
ワークフローの間にサーバーから受け取ったモデルを検証するために呼び出されます。

これらのモデルは、他のクライアントのものである場合もあれば、サーバー上で生成されたものである場合もあります。

.. literalinclude:: ../../nvflare/app_common/np/np_validator.py
   :language: python
   :lines: 15-
   :lineno-start: 15
   :linenos:
   :caption: np_validator.py

Validator は Executor であり、Shareable を受け取る **execute** 関数を実装します。

``validate`` タスクを処理する際は、データの合計を最大値で割り、``random_epsilon`` を加える計算を行い、
その結果を DXO とともに Shareable にパッケージして返します。

.. note::

  hello 系の例では、ディープラーニングとは関係のないデータを使ってフェデレーテッドラーニングを示していることに注意してください。
  NVIDIA FLARE は :ref:`Shareable <shareable>` オブジェクト (``dict`` のサブクラス) 内にパッケージされたあらゆるデータで利用でき、
  そのデータを標準的な方法で管理する手段として :ref:`DXO <data_exchange_object>` の使用が推奨されます。

クロスサイト検証！
----------------------

NVFlare シミュレータを使って実行できます。

.. code-block:: bash

  python3 job_train_and_cse.py


第1フェーズでは、モデルが学習されます。

第2フェーズでは、クロスサイト検証が行われます。

この第2フェーズに入ると、クライアント上のワークフローは
:class:`CrossSiteModelEval<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>` に切り替わります。

クロスサイトモデル評価では、すべてのクライアントが他のクライアントのモデルとサーバーのモデル (存在する場合) を検証します。
これにより多くの結果が生成されることがあります。すべての結果は、ジョブが完了した時点でジョブのワークスペースに保存されます。

出力の理解
^^^^^^^^^^^^^^^^^^^^^^^^

実行ログと結果は、シミュレータのワークスペース内で確認できます。

.. code-block:: bash

  ls /tmp/nvflare/jobs/workdir/
  server/  site-1/  site-2/  startup/


クロスサイト検証の結果:

.. code-block:: bash

  cat /tmp/nvflare/jobs/workdir/server/simulate_job/cross_site_val/cross_val_results.json

おめでとうございます！

numpy を使ったフェデレーテッドラーニングシステムをクロスサイト検証付きで実行できました。

この演習の完全なソースコードは
:github_nvflare_link:`examples/hello-world/hello-numpy-cross-val <examples/hello-world/hello-numpy-cross-val/>` にあります。

Hello Cross-Site Validationの過去バージョン
--------------------------------------------------

  - `hello-numpy-cross-val for 2.0 <https://github.com/NVIDIA/NVFlare/tree/2.0/examples/hello-numpy-cross-val>`_
  - `hello-numpy-cross-val for 2.1 <https://github.com/NVIDIA/NVFlare/tree/2.1/examples/hello-numpy-cross-val>`_
  - `hello-numpy-cross-val for 2.2 <https://github.com/NVIDIA/NVFlare/tree/2.2/examples/hello-numpy-cross-val>`_
  - `hello-numpy-cross-val for 2.3 <https://github.com/NVIDIA/NVFlare/tree/2.3/examples/hello-world/hello-numpy-cross-val/>`_
  - `hello-numpy-cross-val for 2.4 <https://github.com/NVIDIA/NVFlare/tree/2.4/examples/hello-world/hello-numpy-cross-val/>`_
