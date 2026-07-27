.. _site_config:

サイト設定メタデータ
==========================

クライアントサイトは、登録時にサーバーへローカルのメタデータをアドバタイズし、
``local/resources.json`` を通じてサイトレベルのポリシーを少数ながら制御できます。
メタデータは自動的に配信されるため、追加の通信チャネルは不要です。

このページでは、関連する 2 つの制御について説明します。

- :ref:`site_config_metadata` — カスタムのサイトメタデータ (ラベル、
  ケイパビリティ、リソースのヒントなど) をサーバーにアドバタイズします。
- :ref:`allow_log_streaming` — ライブログストリーミングをオプトアウトします。

.. _site_config_metadata:

サイトメタデータのアドバタイズ
--------------------------------------

概要
~~~~~~~~~~

クライアントがサーバーに登録する際、FLARE は既存の登録メッセージに乗せて、
厳選されたサイトメタデータの dict (*site_config*) を転送します。
サーバーはそれを検証し、登録済みの ``Client`` オブジェクトに保存します。保存されたメタデータは
Controller から利用できるようになり、ジョブメタデータにも含まれます。

ユースケースには次のようなものがあります。

- 環境 / リージョンのラベルでサイトにタグ付けする (``"region": "us-east"``)。
- サイトのケイパビリティをアドバタイズする (``"capabilities": ["he", "psi"]``)。
- ハードウェアのヒントを Controller に報告する (``"site_resources":
  {"memory_gb": 128, "gpu_count": 4}``)。

サイトがメタデータを提供する方法
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

サイトの ``local/resources.json`` にカスタムのトップレベルキーを追加します。

.. code-block:: json

    {
      "format_version": 2,
      "client": { "retry_timeout": 30 },
      "components": [ ... ],

      "labels": { "region": "us-east", "tier": "research" },
      "capabilities": ["he", "psi"],
      "site_resources": { "memory_gb": 128, "cpu_cores": 32 }
    }

FLARE はこのファイルを自動的に site_config へ射影します。すべてのトップレベルキーを
ディープコピーし、他のマシンでは意味をなさない構造的 / ローカル専用のキーを除外します。
以下のキーは常に **除外** されます。

- ``format_version``
- ``client``
- ``servers``
- ``components``
- ``handlers``
- ``snapshot_persistor``
- ``admin``
- ``relay_config``
- ``overseer_agent``

サイトが設定内で ``client.site_config`` を明示的に指定している場合は、その値がそのまま
尊重されます (自動射影はスキップされます)。

同じ射影処理は POC / 本番の起動時 (``FLClientStarterConfiger``) とシミュレータ
(``SimulatorDeployer``) の両方で実行されるため、両モードで同一のペイロードが生成されます。

サーバー側の検証
~~~~~~~~~~~~~~~~~~~~~~

サーバーは、登録時に site_config を受け取ると次の 3 つのチェックを適用します。

1. 値は ``dict`` でなければなりません (そうでない場合は警告とともに破棄されます)。
2. 値は JSON シリアライズ可能でなければなりません。
3. シリアライズされたペイロードは 64 KB を超えてはなりません。

また、site_config は通常のクライアント以外 (リレーなど) からは破棄されます。失敗した場合は
ソフトに破棄されます。つまり登録自体は成功し、メタデータが付かないだけです。

サーバー側での site_config へのアクセス
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

サーバー側のコード (ワークフローやウィジェット) の内部では、登録済みのクライアントを検索して
そのメタデータを読み取ります。

.. code-block:: python

    client = engine.get_client_from_name(client_name)
    site_config = client.get_site_config() or {}
    region = site_config.get("labels", {}).get("region")

同じ dict はクライアントのシリアライズ形式 (``Client.to_dict()``) にも現れるため、
ジョブメタデータからも参照できます。

.. note::

   サイトが ``site_config`` に追加した内容はすべて、サーバーおよびジョブメタデータを参照する
   任意の Controller から観測可能です。**ここに機密情報を置かないでください。**

.. _allow_log_streaming:

ライブログストリーミングの制御
--------------------------------------

クライアントによっては、ジョブログをリアルタイムでサーバーへストリーミングしたくない場合があります。
``resources.json`` の ``allow_log_streaming`` ブール値は、ライブログストリーミングに対する
サイトレベルのキルスイッチです。機能そのものの概要については :ref:`live_log_streaming` を
参照してください。

デフォルトは ``true`` であり、ストリーミングは有効です。あるサイトでストリーミングを無効にするには、
次のように設定してオプトアウトします。

.. code-block:: json

    {
      "format_version": 2,
      "client": { ... },
      "components": [ ... ],

      "allow_log_streaming": false
    }

``allow_log_streaming`` が ``false`` に設定されている場合:

- :class:`~nvflare.app_common.logging.job_log_streamer.JobLogStreamer` はジョブ開始時に
  警告をログ出力し、何も行いません。ライブストリームは開かれません。
- :class:`~nvflare.app_common.logging.site_log_streamer.SiteLogStreamer` は、
  デプロイされたジョブ設定からあらかじめ宣言された ``JobLogStreamer`` を取り除き、
  自身の自動インジェクションをスキップし、完了後のエラーログのアップロードもスキップします。
- サーバー側の
  :class:`~nvflare.app_common.logging.job_log_receiver.JobLogReceiver` は、
  ストリーミングを明示的に無効化しているサイトから何らかの理由でチャンクが届いた場合に
  エラーをログ出力します (``(client, job_id)`` ごとに 1 回)。エラーメッセージには
  問題のクライアントとジョブが示されます。

この値は site_config の一部として転送されるため (除外リストに含まれていないため)、
サーバーはチャンク受信時に各クライアントのポリシーを独立して検証できます。

デフォルトの挙動
~~~~~~~~~~~~~~~~~~~~~~

``resources.json`` にこのフィールドが存在しない場合、FLARE はそのサイトがストリーミングを
許可しているものとして扱います。同様に、``resources.json`` を読み取れない場合や、送信元に
対応する ``Client`` がまだ登録されていない場合も、受信側は拒否ではなく許可にフォールバックします。
ストリームを無効にできるのは、明示的な ``false`` のみです。

.. note::

   **アーリーアダプター向けの移行に関する注意。** 開発期間中、関連するプロビジョニング
   プロパティが一時的に ``allow_error_sending`` という名前 (デフォルトは ``False`` で、
   明示的に有効化しない限りストリーミングは無効) になっていたことがあります。
   この名前が公開リリースされたことはありません。現在のプロパティは
   ``allow_log_streaming`` であり、デフォルトはその逆の ``True`` です。プレリリース版の
   ビルドに対してセットアップしたワークスペースを再プロビジョニングする場合は、
   ストリーミングが現在 **デフォルトで有効** であることに注意してください。従来の
   「デフォルトで無効」という状態を維持するには、``project.yml`` の participant (または
   生成された ``resources.json.default``) に ``allow_log_streaming: false`` を設定してください。
