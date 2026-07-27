.. _flare_api:

FLARE API
=========

:mod:`FLARE API<nvflare.fuel.flare_api.flare_api>` は、バージョン 2.3 でより良いユーザー体験のために再設計された FLAdminAPI です。
FLAdminAPI と同様に FL サーバーに発行できる admin コマンドのラッパーであり、プロビジョニングされた admin
クライアントの証明書と鍵を使用して :class:`Session<nvflare.fuel.flare_api.flare_api.Session>` を初期化し、この API のコマンドを使用できます。

.. _flare_api_initialization:

初期化と使い方
------------------------------
:func:`new_secure_session<nvflare.fuel.flare_api.flare_api.new_secure_session>` に、ユーザー名と、admin クライアントの
証明書と鍵を含む startup フォルダを持つプロビジョニング済みユーザーのスタートアップキットフォルダへのパスを指定して、
FLARE API を初期化します:

.. code-block:: python

    from nvflare.fuel.flare_api.flare_api import new_secure_session

    sess = new_secure_session(
        "super@nvidia.com",
        "/workspace/example_project/prod_00/super@nvidia.com"
    )

セッションを特定のスタディにスコープするには、``study`` パラメータを渡します:

.. code-block:: python

    sess = new_secure_session(
        "super@nvidia.com",
        "/workspace/example_project/prod_00/super@nvidia.com",
        study="cancer-research"
    )

``study`` を省略した場合、セッションは ``"default"`` スタディを使用します。セッションを通じて発行される
スタディ対応コマンド(``submit_job``、``list_jobs``、``get_job_meta``、``clone_job`` など)は、
アクティブなスタディにスコープされます。名前付きスタディにはマルチスタディ構成のデプロイメントが必要です。
詳細は :ref:`multi_study_guide` を参照してください。

ログインは自動的に処理され、返されたセッションオブジェクト(前のコードブロックの ``sess``)でコマンドを実行できます。

FLARE API の使い方は以前の FLAdminAPI に似ていますが、よりシンプルです。戻り値の構造は、ステータスと詳細の辞書を持つ
オブジェクトではなくなり、コマンドによっては文字列であったり、何も返さなかったりします。エラーを含むステータスを返す代わりに、
FLARE API は例外を送出するようになり、これらの例外の処理は FLARE API を使用するコード側の責任となります。

返されるものがはるかにシンプルになったため、コマンドをラップする必要はなくなったはずです。たとえば ``submit_job`` は、
ジョブがシステムに受理されるとジョブ ID を文字列として返すようになりました。各コマンドが返す内容の詳細は、
:mod:`FLARE API<nvflare.fuel.flare_api.flare_api>` の docstring を参照してください。

.. _flare_api_implementation_notes:

実装に関する注記
--------------------------------
以前の FLAdminAPI と同様に、サーバーへの接続とコマンドの送信には :class:`AdminAPI<nvflare.fuel.hci.client.api.AdminAPI>` が使用されます。

``logout()`` はなくなりました。代わりに ``close()`` を使用してセッションを終了します。よくある使用パターンの 1 つは、
セッションを使用するコードで try ブロック内でコマンドを実行し、finally 句でセッションを閉じることです:

.. code-block:: python

    try:
        print(sess.get_system_info())
        job_id = sess.submit_job("/workspace/location_of_jobs/job1")
        print(job_id + " was submitted")
        # monitor_job() waits until the job is done, see the section about it below for details
        sess.monitor_job(job_id)
        print("job done!")
    finally:
        sess.close()


.. note::

    注記: ``close()`` を呼び出した際、セッションモニターが閉じるまでに少し時間がかかる場合があります。

追加コマンドと複雑なコマンド
--------------------------------------------------------
たとえば、上のコードブロックの submit_job の後に得られる ``job_id`` を使って、実行できる他のコマンドの例を
いくつか示します:

.. code-block:: python

    # get job meta dictionary with job info
    job_meta_dict = sess.get_job_meta(job_id)

    # submit a copy of an existing job with clone_job
    new_job_id = sess.clone_job(job_id)
    print(new_job_id + " was submitted as a clone of " + job_id)

.. _flare_api_monitor_job:

ジョブの監視
^^^^^^^^^^^^^^^^^^^^^^^^
デフォルトでは、上記 :ref:`flare_api_implementation_notes` の最も基本的な使い方のように、``monitor_job()`` は
第 1 引数で指定されたジョブが終了するまで待機しますが、追加の引数を指定することで、より柔軟な使い方ができます。
たとえば、ステータスのポーリングごとに呼び出されるカスタムコードを含む独自のコールバックを渡せます。以下は
monitor_job の API 仕様です:

.. code-block:: python

    def monitor_job(
        self, job_id: str, timeout: int = 0, poll_interval: float = 2.0, cb=None, *cb_args, **cb_kwargs
    ) -> MonitorReturnCode:
        """Monitor the job progress until one of the conditions occurs:
         - job is done
         - timeout
         - the status_cb returns False

        Args:
            job_id: the job to be monitored
            timeout: how long to monitor. If 0, never time out.
            poll_interval: how often to poll job status
            cb: if provided, callback to be called after each poll

        Returns: a MonitorReturnCode

        Every time the cb is called, it must return a bool indicating whether the monitor
        should continue. If False, this method ends.

        """

必須なのは第 1 引数のみですが、追加の引数を使うことで、``monitor_job()`` をほぼ望みどおりに
カスタマイズできます。以下は sample_cb と cb_kwargs の使い方が分かる例です。
このコールバックは常に True を返すため、第 1 引数で指定されたジョブが終了するまで待機するという
``monitor_job()`` のデフォルトの動作を維持しますが、望みの動作になるようにカスタマイズできます。

.. code-block:: python

    def sample_cb(
        session: Session, job_id: str, job_meta, *cb_args, **cb_kwargs
    ) -> bool:
        if job_meta["status"] == "RUNNING":
            if cb_kwargs["cb_run_counter"]["count"] < 3:
                print(job_meta)
                print(cb_kwargs["cb_run_counter"])
            else:
                print(".", end="")
        else:
            print("\n" + str(job_meta))
        
        cb_kwargs["cb_run_counter"]["count"] += 1
        return True

    # Calling monitor_job with the sample_cb above and a cb_kwarg
    sess.monitor_job(job_id, cb=sample_cb, cb_run_counter={"count":0})
