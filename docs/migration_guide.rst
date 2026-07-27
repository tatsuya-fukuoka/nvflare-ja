.. _migration_guide:

######################
移行ガイド
######################

このガイドでは、FLAREのリリース間でアップグレードする際のAPIおよび構成の変更点を説明します。

2.7.2から2.8.0へのアップグレード
==========================================================

Pythonおよび削除されたレガシーAPI
--------------------------------------------------------------

FLARE 2.8.0は、Python 3.10から3.14を対象としています。Python 3.9は
サポート対象の開発ターゲットから外れました。

非推奨だったFLAdminAPIは削除されました。新しい自動化には、FLARE API、Recipe
API、Client API、および ``nvflare`` CLIワークフローを使用してください。

HA/Overseerのコードも2.8ブランチから削除されました。

Client APIサブプロセスのタイムアウト検証
------------------------------------------------------------------------------

サブプロセスモードのClient APIジョブは、ジョブの初期化時に、大規模モデルの安全性に関する
2つの設定を検証するようになりました:

- ``download_complete_timeout`` は ``None`` であってはなりません。サーバーがサブプロセスの
  ``DownloadService`` からテンソルのダウンロードを完了できるように、``send_to_peer()`` のACK後も
  サブプロセスは稼働し続ける必要があります。
- ``ClientAPILauncherExecutor`` を使用する場合、``max_resends`` は ``None`` であってはなりません。
  再送回数が無制限だと、遅延した1回の大規模モデル転送が、際限のない
  代替ダウンロードトランザクションの連鎖に変わってしまう可能性があります。

2.7.xのジョブでいずれかの値を明示的に ``None`` に設定していた場合は、2.8.0で実行する前に
更新してください。レシピベースの外部プロセスジョブでは、エグゼキューターの引数にデフォルトの
``max_resends=3`` がすでにシリアライズされているため、次の設定が必要になるのは、以前の明示的な
``None`` を上書きする場合や、異なるリトライ回数を選択する場合のみです:

.. code-block:: python

   recipe.add_client_config({
       "download_complete_timeout": 1800,
       "max_resends": 3,  # finite non-negative integer; 0 disables retries
   })

大規模なテンソルやNumPyのペイロードでは、関連するストリーミングタイムアウトも
一貫性を保つようにしてください。``tensor_streaming_per_request_timeout`` または
``np_streaming_per_request_timeout`` を明示的に引き上げる場合は、``PEER_READ_TIMEOUT`` と
``download_complete_timeout`` を、構成したストリーミングのリクエストごとのタイムアウト以上の値に設定し、
``tensor_min_download_timeout`` または ``np_min_download_timeout`` も同じ値以上に
保ってください。

.. code-block:: python

   recipe.add_client_config({
       "tensor_streaming_per_request_timeout": 600,
       "tensor_min_download_timeout": 600,
       "PEER_READ_TIMEOUT": 600,
       "download_complete_timeout": 1800,
       "max_resends": 3,
   })

完了済みダウンロード参照に対する遅延リトライの処理
----------------------------------------------------------------------------------------------------

FLARE 2.8.0では、完了済みの ``DownloadService`` 参照が、同一リクエスターからの
リトライに対して安全になりました。クライアントが大規模なダウンロードを完了したものの、最後の
EOF応答が遅延したためにリトライした場合、サーバーは ``INVALID_REQUEST`` / ``no ref found`` ではなく
同じ終了ステータスを返します。これは内部的な信頼性の修正であり、
ジョブコードの変更は不要ですが、非常に大きなモデルに対して上記のサブプロセスの
タイムアウトが一貫して構成されている場合に最も効果を発揮します。

今後のmainブランチの変更点
========================================================

FLARE API互換性に関する注記
--------------------------------------------------------

現在の ``main`` ブランチでは、:class:`NoConnection<nvflare.fuel.flare_api.api_spec.NoConnection>`
は ``Exception`` を直接継承するのではなく、Python組み込みの ``ConnectionError`` を
継承するようになりました。

影響:

- ``ConnectionError`` を捕捉している既存のコードは、``NoConnection`` も
  捕捉するようになります。
- ``NoConnection`` を捕捉している既存のコードは、変更なしで引き続き動作します。

アプリケーションでFLAREの接続失敗を、より広範なOSやネットワークの例外と
区別している場合は、``main`` からビルドされた次のリリースにアップグレードする前に、
広範な ``except ConnectionError:`` ハンドラーを見直してください。

FLARE APIライフサイクルの制限
----------------------------------------------------------

現在の ``main`` ブランチでは、:meth:`Session.shutdown<nvflare.fuel.flare_api.api_spec.SessionSpec.shutdown>`
および :meth:`Session.restart<nvflare.fuel.flare_api.api_spec.SessionSpec.restart>`
は ``TargetType.SERVER`` のみに制限されるようになりました。

影響:

- ``TargetType.ALL`` または ``TargetType.CLIENT`` を渡す既存の呼び出し元は失敗するようになります。
- サーバースコープのライフサイクル制御は、変更なしで引き続き動作します。
- :meth:`Session.shutdown_system<nvflare.fuel.flare_api.api_spec.SessionSpec.shutdown_system>`
  は変更されておらず、引き続きシステム全体のシャットダウンをサポートします。

ローカルPoC全体のライフサイクル制御には、汎用のシステム管理APIではなく、
PoCのstart/stopフローを使用してください。

CLIスタートアップキット解決の変更
------------------------------------------------------------------

現在の ``main`` ブランチでは、サーバーに接続するCLIコマンドは、
``~/.nvflare/config.conf`` にある共有のアクティブスタートアップキットレジストリを使用します。

影響:

- ``nvflare config add <id> <startup-kit-dir>`` と
  ``nvflare config use <id>`` を使用して、スタートアップキットを登録・アクティベートします。
- ``nvflare config -d/--startup_kit_dir`` は2.7.xスクリプトとの互換性のために
  引き続き受け付けられますが、非推奨です。
- ``NVFLARE_STARTUP_KIT_DIR`` は引き続き自動化用のオーバーライドとして機能し、
  設定されている場合はアクティブなレジストリエントリより優先されます。
- ``nvflare config -jt/--job_templates_dir`` は2.7.xスクリプトとの互換性のために
  引き続き受け付けられますが、ジョブテンプレート構成は非推奨です。
- ルートの ``nvflare config`` は、POCワークスペースなどのローカル設定の管理を
  引き続き担います。スタートアップキットのパスは ``nvflare config``
  サブコマンドで管理されます。

``NVFLARE_STARTUP_KIT_DIR`` をエクスポートするシェルプロファイルやCI設定を使用している場合は、
アクティブなレジストリエントリを上書きするため、アップグレード前に見直してください。

CLI構成フラグの互換性
----------------------------------------------

現在の ``main`` ブランチでは、``nvflare config`` は2.7.xのPOC
ワークスペースのフラグ名を維持しています。

影響:

- POCワークスペースの設定には、``-pw`` と ``--poc_workspace_dir`` が
  引き続きサポートされるフラグです。
- 開発中の暫定表記であった ``--poc.workspace`` は、公開互換性契約の
  一部ではありません。

``-pw`` または ``--poc_workspace_dir`` を使用する古いスクリプトがある場合、
それらは引き続き動作します。

クライアント無効化のセマンティクス
--------------------------------------------------------------------

``nvflare system remove-client`` は、サポートされる公開CLIコマンドとしては
公開されていません。レガシーの対話型コンソールコマンド ``remove_client`` は通常のヘルプからは
非表示となり、レジストリのクリーンアップ操作としてのみ残っています: アクティブなトークンを解放し、
クライアントが再登録できるようにします。クライアントプロセスの停止、認証情報の失効、
再接続の防止は行いません。

クライアントを連合から締め出すことを意図する場合は、新しい永続的なアクセス制御コマンドを
使用してください:

- ``nvflare system disable-client <client> --force`` は、無効化フラグを
  サーバーワークスペースに永続化し、アクティブなレジストリエントリを削除して、
  そのクライアントからの以降の登録やハートビートを拒否します。
- ``nvflare system enable-client <client> --force`` は無効化フラグをクリアし、
  次回の登録またはハートビート時にクライアントが再参加できるようにします。

これは運用上の無効化であり、証明書の失効ではありません。

スタディ名検証の緩和
----------------------------------------

現在の ``main`` ブランチでは、スタディ名の内部位置にアンダースコアが
使用できるようになったため、``my_study`` のような名前が有効になります。

影響:

- ``project.yml`` の検証は、内部にアンダースコアを含むスタディ名を受け付けるようになります。
- ログインおよびスタディスコープの認可パスも、同じ名前を受け付けます。

スタディ識別子に関する外部の検証や命名ポリシーを管理している場合は、
アップグレード前に新しいルールに合わせてそれらのチェックを更新してください。

サイトログ構成の制限
----------------------------------------

現在の ``main`` ブランチでは、:meth:`Session.configure_site_log<nvflare.fuel.flare_api.api_spec.SessionSpec.configure_site_log>`
および対応する ``nvflare system log-config`` の経路は、シンプルなログレベルと
組み込みのログモードのみを受け付けるようになりました。

影響:

- サイト全体のログ変更に、JSONの ``dictConfig`` ペイロードは受け付けられなくなりました。
- サイト全体のログ変更に、ファイルパスベースのログ構成は受け付けられなくなりました。
- サポートされる値は、標準のログレベルに加え、``concise``、``msg_only``、``full``、
  ``verbose``、``reload`` などの組み込みモードのままです。

これまで ``configure_site_log`` で高度なJSON/ファイルベースの構成を使用していた場合は、
``main`` からビルドされた次のリリースにアップグレードする前に、サポートされる
レベル/モードの値に切り替えてください。
dictベースまたはファイルパスによるログ設定には、代わりに実行中のジョブに対して ``configure_job_log`` を使用してください。

POC startのデフォルトサービスの明確化
------------------------------------------------------------------------------

現在の ``main`` ブランチでは、``nvflare poc start`` のドキュメント上のデフォルト動作が、
実際のランタイム動作を反映するように明確化されました:
デフォルトで起動されるのはサーバーとクライアントのサービスであり、
ワークスペース配下のすべての参加者ディレクトリではありません。

影響:

- 明示的な ``-p`` / ``--service`` なしで ``nvflare poc start`` を実行すると、
  サーバーとクライアントが起動します。
- 管理コンソールは、明示的に選択しない限り起動しません。

これはドキュメント/ヘルプの明確化であり、ランタイム動作の変更ではありません。

2.7.0/2.7.1から2.7.2へのアップグレード
================================================================

Recipe APIの変更点
------------------------------------

**initial_modelがmodelにリネーム**

わかりやすさのため、すべてのレシピの ``initial_model`` パラメーターは ``model`` にリネームされました:

.. code-block:: python

    # Before (2.7.0/2.7.1)
    recipe = FedAvgRecipe(
        ...
        initial_model=SimpleNetwork(),
    )

    # After (2.7.2)
    recipe = FedAvgRecipe(
        ...
        model=SimpleNetwork(),
    )

``model`` パラメーターは、オプションの事前学習済みチェックポイントを伴うdictベースの構成も受け付けるようになりました:

.. code-block:: python

    recipe = FedAvgRecipe(
        ...
        model={"path": "my_module.MyModel", "args": {"hidden_size": 256}},
        initial_ckpt="pretrained.pt",
    )

**PTFedAvgEarlyStoppingがPTFedAvgに統合**

``PTFedAvgEarlyStopping`` は、InTime集約のサポートとともに ``PTFedAvg`` に統合されました。
後方互換性のためのエイリアスは提供されていますが、新しいコードでは ``PTFedAvg`` を使用してください:

.. code-block:: python

    # Before
    from nvflare.app_opt.pt.fedavg_early_stopping import PTFedAvgEarlyStopping
    controller = PTFedAvgEarlyStopping(...)

    # After
    from nvflare.app_opt.pt.fedavg import PTFedAvg
    controller = PTFedAvg(...)

MONAI統合
--------------------

独立した ``nvflare-monai`` wheelパッケージは非推奨です。MONAI統合にはClient APIを直接
使用してください。``examples/advanced/monai/`` の更新されたサンプルと
`MONAI Migration Guide <https://github.com/NVIDIA/NVFlare/blob/main/integration/monai/MIGRATION.md>`_ を参照してください。

新機能(移行不要)
------------------------------------

以下の2.7.2の機能は、コード変更なしで自動的に動作します:

- **TensorDownloader**: PyTorchモデル重み転送の透過的なメモリ最適化。
  :ref:`tensor_downloader` を参照してください。
- **サーバー側メモリクリーンアップ**: 自動ガベージコレクションとヒープトリミング。
  :doc:`/programming_guide/memory_management` を参照してください。

後方互換性
--------------------

- **Job Config API**: 既存の ``FedJob`` ベースの構成は、新しいRecipe APIと並行して引き続き動作します。
- **構成ベースのジョブ**: JSON/YAML構成ベースのジョブは、これまでどおり引き続き動作します。
- **Executor/ModelLearner API**: 引き続き機能しますが、もはや推奨パターンではありません。新しいプロジェクトではRecipe APIとClient APIを使用してください。

変更点の全リストは、:doc:`2.7.2の新機能 </release_notes/flare_272>` リリースノートを参照してください。

2.5/2.6から2.7へのアップグレード
============================================================

FLARE 2.7.0では、いくつかの大きな変更が導入されました:

- **Job Recipe API** (テクニカルプレビュー): FLジョブを作成するための高レベルAPI。:ref:`job_recipe` を参照してください。
- **Client API** が、すべての新しいFLジョブに推奨されるパターンになりました。
- **階層型FL**: 大規模デプロイメント向けの新しいリレーベースの通信階層。
  :ref:`flare_hierarchical_architecture` を参照してください。
- **エッジとモバイル**: ExecuTorchによるモバイルデバイス(iOS/Android)上での連合学習トレーニング。
  :ref:`mobile_training` を参照してください。
- **ファイルストリーミング**: 大規模モデル転送のためのプル型ファイルダウンロード。
  :ref:`file_streaming` を参照してください。

古いFLAdminAPIからClient APIへの移行については、:doc:`FLARE APIへの移行 </programming_guide/migrating_to_flare_api>` を参照してください。

2.7.0の変更点の全リストは、:doc:`/release_notes/flare_270` を参照してください。
