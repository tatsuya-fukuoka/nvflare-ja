.. _system_command:

#########################
System コマンド
#########################

``nvflare system`` コマンドグループは、admin API を介して稼働中の FL システムを管理します。

***********************
コマンドの使い方
***********************

.. code-block:: none

   nvflare system -h

   usage: nvflare system [-h]  ...

   system subcommands:
     status         show server and client status
     resources      show server and client resource usage
     shutdown       shut down server, clients, or all
     restart        restart server, clients, or all
     disable-client disable a client from reconnecting
     enable-client  enable a disabled client to reconnect
     version        show NVFlare version on each remote site
     log-config     change logging level on server or client sites

*********************
よく使う例
*********************

システム全体のステータスを表示します。

.. code-block:: shell

   nvflare system status

サーバーのみのステータスを表示します。

.. code-block:: shell

   nvflare system status server

報告されているすべてのリソースを表示します。

.. code-block:: shell

   nvflare system resources

クライアントのリソースのみを表示します。

.. code-block:: shell

   nvflare system resources client

サーバーを再起動します。

.. code-block:: shell

   nvflare system restart server --force

サーバーをシャットダウンします。

.. code-block:: shell

   nvflare system shutdown server --force

特定のクライアントをシャットダウンします。

.. code-block:: shell

   nvflare system shutdown client site-1 site-2 --force

クライアントの再接続を無効化します。

.. code-block:: shell

   nvflare system disable-client site-1 --force

無効化されたクライアントを有効化します。

.. code-block:: shell

   nvflare system enable-client site-1 --force

デプロイされている NVFlare のバージョンを表示します。

.. code-block:: shell

   nvflare system version
   nvflare system version --site server

実行時のロギングを変更します。

.. code-block:: shell

   nvflare system log-config concise
   nvflare system log-config --site server DEBUG
   nvflare system log-config --site site-1 msg_only

.. note::

   サーバーに接続するすべての ``nvflare system`` コマンドは、次の順序でスタートアップキットを
   解決します。``--kit-id <id>`` 、``--startup-kit <path>`` 、``NVFLARE_STARTUP_KIT_DIR`` 、
   そして ``~/.nvflare/config.conf`` の ``startup_kits.active`` です。``--kit-id`` と
   ``--startup-kit`` は、コマンドごとの任意の上書き指定です。指定した場合は、その実行に限り
   アクティブなスタートアップキットよりも優先され、``startup_kits.active`` は変更されません。
   アクティブなスタートアップキットの管理には ``nvflare config add`` と ``nvflare config use``
   を使用します。:ref:`config_command` を参照してください。

**************************
ステータスとリソース
**************************

``nvflare system status`` は、サーバーとクライアントの接続状況を報告します。

位置引数 ``target`` は ``server`` または ``client`` を意味します。これによって NVFlare に
何を照会するかを指示します。

status の引数:

- 位置引数 ``target``: 任意。``server`` または ``client`` 。
- 位置引数 ``client_names``: クライアントを対象とする場合の、任意のクライアント名のリスト。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare system status
   nvflare system status server
   nvflare system status client site-1 site-2

``nvflare system status client site-1`` における ``client site-1`` は、クライアント ``site-1`` を
照会することを意味します。

``nvflare system resources`` は、サーバーとクライアントのリソース使用状況を報告します。

位置引数 ``target`` は ``server`` または ``client`` を意味します。これによって NVFlare に
何を照会するかを指示します。

resources の引数:

- 位置引数 ``target``: 任意。``server`` または ``client`` 。
- 位置引数 ``client_names``: クライアントを対象とする場合の、任意のクライアント名のリスト。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare system resources
   nvflare system resources client
   nvflare system resources client site-1

``nvflare system resources client site-1`` における ``client site-1`` は、クライアント ``site-1`` を
照会することを意味します。

******************************
シャットダウンと再起動
******************************

admin チャネルを介してサーバーやクライアントのプロセスを制御するには、``shutdown`` と ``restart``
を使用します。

サポートされている対象:

- ``server`` — FL サーバーをシャットダウンまたは再起動します (admin セッションは閉じられます)。
- ``client`` — 1 つ以上のクライアントをシャットダウンまたは再起動します。
- ``all`` — サーバーとすべてのクライアントをシャットダウンまたは再起動します (admin セッションは閉じられます)。

制御用の引数:

- 位置引数 ``target``: 必須。``server`` 、``client`` 、``all`` のいずれか。
- 位置引数 ``client_names``: 任意。1 つ以上のクライアント名。``target`` が ``client`` の場合にのみ意味を持ちます。
- ``--force``: 確認プロンプトをスキップします。
- ``--no-wait``: シャットダウンまたは再起動を要求した後、完了を待たずに戻ります。
- ``--timeout SECONDS``: シャットダウンまたは再起動の完了を待つ最大秒数 (正の値)。デフォルト: ``30`` 。
  投げっぱなしの動作にしたい場合は ``--timeout 0`` ではなく ``--no-wait`` を使用してください。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare system shutdown server --force
   nvflare system shutdown client site-1 site-2 --force
   nvflare system shutdown all --force
   nvflare system shutdown all --force --no-wait
   nvflare system shutdown all --force --timeout 120

   nvflare system restart server --force
   nvflare system restart client site-1 --force
   nvflare system restart all --force
   nvflare system restart server --force --no-wait
   nvflare system restart all --force --timeout 120

非対話的なコンテキストでは ``--force`` が必須です。

デフォルトでは、shutdown は対象が停止するまで待ってから戻り、restart は対象が再び到達可能になるまで
待ってから戻ります。``restart all`` の場合、これにはサーバーの再起動と、それまで接続していた
クライアントの再接続を待つことが含まれます。``--no-wait`` を指定すると、コマンドは開始済みの
ステータスを返して直ちに戻ります。``target`` が ``server`` または ``all`` の場合、シャットダウンまたは
再起動の要求を送信した後、admin セッションは自動的に閉じられます。
待機時間が ``--timeout`` を超えた場合、コマンドは接続失敗を報告するのではなく、``TIMEOUT`` と
終了コード ``3`` を返します。

******************************
クライアントのアクセス制御
******************************

稼働中のフェデレーションへの参加を、あるクライアントアイデンティティに対して恒久的にブロックするには
``disable-client`` を使用します。サーバーはそのクライアントのアクティブなレジストリエントリを削除し、
クライアントが有効化されるまで、以降の登録やハートビートの試行を拒否します。これはクライアントの証明書を
失効させたり、そのスタートアップキットを削除したりするものではありません。JSON 出力には
``already_disabled`` が含まれるため、呼び出し側は状態変更と冪等な no-op を区別できます。

無効化フラグを解除するには ``enable-client`` を使用します。クライアントは次回の登録またはハートビートで
再参加できます。

無効化クライアントのポリシーは、サーバー上の ``<server_workspace>/disabled_clients.json`` に保存され、
サーバー起動時に読み込まれます。更新と永続化の書き込みはサーバーのクライアントマネージャーのロックに
よって直列化され、一時ファイルへの書き込みとアトミックな置換によって行われるため、ポリシーは
部分的に書き込まれたファイルを残すことなくサーバーの再起動を越えて維持されます。ファイルが存在するのに
読み込めない場合、サーバーは以前に無効化されたクライアントを受け入れるのではなく、起動時に
フェイルクローズします。

クライアントアクセス用の引数:

- 位置引数 ``client_name``: 必須。無効化または有効化するクライアントの名前。
- ``--force``: 確認プロンプトをスキップします。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare system disable-client site-1 --force
   nvflare system enable-client site-1 --force

****************
バージョン
****************

リモートサイトが報告する NVFlare のバージョンを照会するには ``nvflare system version`` を使用します。

version の引数:

- ``--site``: ``server`` 、クライアント名、または ``all`` 。デフォルト: ``all`` 。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

例:

.. code-block:: shell

   nvflare system version
   nvflare system version --site server
   nvflare system version --site site-1

このコマンドは、各サイトのバージョン、それらがサーバーのバージョンと互換性があるかどうか、および
どのサイトが不一致であるかを報告します。

************************
実行時のロギング
************************

サーバーまたはクライアントサイトのロギングを変更するには ``nvflare system log-config`` を使用します。

.. code-block:: shell

   nvflare system log-config DEBUG
   nvflare system log-config --site server verbose
   nvflare system log-config --site site-1 msg_only

ロギングの引数:

- 位置引数 ``level``: 実行時に必須のログレベル、または組み込みのログモード。省略すると CLI エラーが返されます。
- ``--site``: ``server`` 、クライアント名、または ``all`` 。デフォルト: ``all`` 。
- ``--schema``: コマンドスキーマを JSON として出力して終了します。

位置引数 ``level`` でサポートされている組み込みの値:

- ``DEBUG``
- ``INFO``
- ``WARNING``
- ``ERROR``
- ``CRITICAL``
- ``concise``
- ``msg_only``
- ``full``
- ``verbose``
- ``reload``

``level`` は実行時に必須です。省略しても argparse のパースは失敗しませんが、コマンドはエラーを返します。

*************************
JSON 出力とヘルプ
*************************

機械可読な出力を得るには、サブコマンドの後に ``--format json`` を追加します。

.. code-block:: shell

   nvflare system status --format json
   nvflare system version --site server --format json

標準出力には単一の JSON エンベロープが含まれ、人間向けの進行状況や診断情報は標準エラー出力に
送られます。

機械可読なコマンド探索には ``--schema`` を使用します。``--schema`` は常に JSON を返すため、
``--format json`` を併用する必要はありません。

.. code-block:: shell

   nvflare system status --schema
   nvflare system shutdown server --schema

人間向けの引数エラーでは、まずコマンドのヘルプが出力され、続いて具体的なエラーとヒントが
表示されます。JSON モードでは JSON のエラーエンベロープのみが出力されます。
