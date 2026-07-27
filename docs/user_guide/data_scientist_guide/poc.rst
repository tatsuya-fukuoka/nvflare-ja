.. _poc:

POC: 概念実証: 本番デプロイメントをローカルでシミュレートする
==============================================================


.. _setting_up_poc:

POC モードでのアプリケーション環境のセットアップ
--------------------------------------------------

:ref:`installation` の後に概念実証 (POC) のセットアップを始めるには、次のコマンドを実行して、
1 台のサーバー、2 つのクライアント、1 つの管理クライアントを含む poc フォルダを生成します。

.. code-block:: shell

    $ nvflare poc prepare -n 2

詳細は :ref:`poc_command` を参照してください。

.. _starting_poc:

POC モードでのアプリケーション環境の起動
--------------------------------------------------

FL システムを起動する準備ができたら、次のコマンドを実行して、サーバーとクライアントのシステム、
および管理コンソールを起動できます。

.. code-block::

  nvflare poc start

管理コンソールなしでサーバーとクライアントのシステムを起動するには、次のようにします。

.. code-block::

  nvflare poc start -ex admin@nvidia.com

:ref:`job_cli` を使うと、POC システムにジョブを簡単に送信できます。(注: シミュレータで実行したのと同じジョブを POC モードで実行できます。 :ref:`fed_job_api` を使用している場合は、 ``job.export_job()`` でジョブ設定をエクスポートするだけです。)

.. code-block::

  nvflare job submit -j NVFlare/examples/hello-world/hello-numpy

.. code-block::

  nvflare poc stop

.. code-block::

  nvflare poc clean

詳細は :ref:`poc_command` を参照してください。

`POC チュートリアル: <https://github.com/NVIDIA/NVFlare/tree/main/examples/tutorials/setup_poc.ipynb>`_ もご覧ください。
