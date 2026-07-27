.. _preflight_check:

****************************************
NVIDIA FLARE プリフライトチェック
****************************************

NVIDIA FLARE のプリフライトチェックは、ユーザーが自分のマシンで NVFlare のサブシステムを起動する前に
予備的なチェックを実施し、エラーを早期に検出して NVIDIA FLARE のセットアップやジョブ実行にかかる
手間を軽減するためのものです。

一般的な使い方
==============

.. code-block::

    nvflare preflight-check -p PACKAGE_PATH
    nvflare preflight-check --package_path PACKAGE_PATH


このプリフライトチェックスクリプトは、各サイトのマシン上で実行してください。 ``PACKAGE_PATH`` は、
チェック対象のパッケージが格納されているフォルダへのパスです。

スクリプトを実行すると、合格したチェックについては "PASSED" と表示されます。失敗したチェックについては、
問題の内容とその修正方法が報告されます。

終了コード ``0`` は、該当するすべてのチェックに合格したことを意味します。終了コード ``1`` は、
該当するチェックのうち少なくとも 1 つが失敗したことを意味します。
終了コード ``4`` は、パッケージパスまたはパッケージ形式が不正であることを意味します。

以下に、サイトの種類ごとにプリフライトチェックを実行するスクリプトと、報告される可能性のある問題を示します。


サーバーサイトでのプリフライトチェック
--------------------------------------

サーバーパッケージが "/path_to_NVFlare/NVFlare/workspace/example_project/prod_00" にあり、その名前が "server1" である場合、
サーバーサイトでは次のように実行します。

.. code-block::

  nvflare preflight-check -p /path_to_NVFlare/NVFlare/workspace/example_project/prod_00/server1

報告される可能性のある問題は次のとおりです。

.. csv-table::
    :header: チェック項目,報告される問題,対処方法
    :widths: 15, 20, 25

    FL ポートのバインド確認,Can't bind to address ({host}:{port}): {e},DNS とポートを確認してください。
    管理ポートのバインド確認,Can't bind to address ({host}:{port}): {e},DNS とポートを確認してください。
    スナップショットストレージの書き込み可否確認,Can't write to {snapshot_storage_root}: {e}.,ユーザー権限を確認してください。
    ジョブストレージの書き込み可否確認,Can't write to {job_storage_root}: {e}.,ユーザー権限を確認してください。
    dry run の確認,Can't start successfully: {error},dry run のエラーメッセージを確認してください。


クライアントサイトでのプリフライトチェック
------------------------------------------

クライアントをチェックする前に、サーバーが稼働していることを確認してください。

クライアントパッケージが "/path_to_NVFlare/NVFlare/workspace/example_project/prod_00" にあり、その名前が "site-1" である場合、
クライアントサイトでは次のように実行します。

.. code-block::

  nvflare preflight-check -p /path_to_NVFlare/NVFlare/workspace/example_project/prod_00/site-1

報告される可能性のある問題は次のとおりです。

.. csv-table::
    :header: チェック項目,報告される問題,対処方法
    :widths: 15, 20, 25

    サーバーの利用可否確認,Can't connect to {scheme} server ({host}:{port}),サーバーが起動しているか確認してください。
    dry run の確認,Can't start successfully: {error},dry run のエラーメッセージを確認してください。


管理コンソールでのプリフライトチェック
--------------------------------------

FLARE 管理コンソールをチェックする前に、サーバーが稼働していることを確認してください。

FLARE コンソールのパッケージが "/path_to_NVFlare/NVFlare/workspace/example_project/prod_00/" にあり、その名前が "admin@nvidia.com" である場合、
次のように実行します。

.. code-block::

  nvflare preflight-check -p /path_to_NVFlare/NVFlare/workspace/example_project/prod_00/admin@nvidia.com

報告される可能性のある問題は次のとおりです。

.. csv-table::
    :header: チェック項目,報告される問題,対処方法
    :widths: 15, 20, 25

    サーバーの利用可否確認,Can't connect to {scheme} server ({host}:{port}),サーバーが起動しているか確認してください。
    dry run の確認,Can't start successfully: {error},dry run のエラーメッセージを確認してください。
