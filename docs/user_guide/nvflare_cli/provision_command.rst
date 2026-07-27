*****************************
プロビジョニングコマンド
*****************************

``nvflare provision -h`` を実行すると、利用可能なすべてのオプションが表示されます。

.. code-block:: shell

    usage: nvflare provision [-h] [-p PROJECT_FILE] [-g] [-e] [-w WORKSPACE]
                             [-c CUSTOM_FOLDER] [--add_user ADD_USER]
                             [--add_client ADD_CLIENT] [-s] [--force] [--schema]

    options:
    -h, --help                                               show this help message and exit
    -p PROJECT_FILE, --project_file PROJECT_FILE                 file to describe FL project
    -g, --generate                                             generate a sample project.yml and exit
    -e, --gen_edge                                             generate a sample edge project.yml and exit
    -w WORKSPACE, --workspace WORKSPACE                          directory used by provision
    -c CUSTOM_FOLDER, --custom_folder CUSTOM_FOLDER    additional folder to load python code
    --add_user ADD_USER                                             yaml file for added user
    --add_client ADD_CLIENT                                       yaml file for added client
    -s, --gen_scripts                                            generate helper scripts such as start_all.sh
    --force                                                      skip Y/N confirmation prompts
    --schema                                                     print command schema as JSON and exit

オプションを何も指定せず、かつカレントワーキングディレクトリに project.yml ファイルが存在しない状態で
``provision`` を実行すると、デフォルトの project.yml をカレントワーキングディレクトリにコピーするかどうかを
確認するプロンプトが表示されます。

JSON モード
===========

``nvflare provision --format json`` は、標準出力に JSON エンベロープのみを返します。
コマンドがサンプルのプロジェクトファイルを生成する場合、JSON の ``data`` セクションには
次のような構造化されたガイダンスが含まれます。

- ``message``
- ``next_step``
- ``suggested_command``

これにより、JSON 出力を機械可読なまま保ちながら、後続の手順も併せて伝えることができます。

ルート CA の有効期間
====================

集中型のプロビジョニングでは、デフォルトで 360 日間有効な自己署名のプロジェクトルート CA が
作成されます。初期の有効期間を変更するには、``project.yml`` の ``CertBuilder`` に正の整数の
``root_valid_days`` を設定します。

.. code-block:: yaml

   builders:
     - path: nvflare.lighter.impl.cert.CertBuilder
       args:
         root_valid_days: 3650

この設定は、ワークスペースがルートを作成するときにのみ適用されます。以降の実行では、
ワークスペースの ``state`` ディレクトリにあるルートを再利用し、その実際の ``NotBefore`` および
``NotAfter`` の値を報告します。既存のルートと一致しない有効期間が設定されている場合は、
明確にエラーとなります。参加者の証明書は、ルートがそれより先に失効しない限り、通常どおり
360 日間の有効期間を保ちます。

確立済みのルートを変更するには、別途マルチルートのロールオーバーが必要です。
``root_valid_days`` によって既存のルートが延長されたり置き換えられたりすることは決してありません。

証明書アイデンティティのオーバーライド
======================================

mTLS 構成のデプロイメントでは、各 CellNet エンドポイントはピア証明書のコモンネーム (CN) に対して
認証されます。デフォルトでは、期待される証明書アイデンティティは参加者名または FQCN から導出されます。
参加者が意図的に FLARE のサイト名とは異なる証明書 CN を使用する場合は、プロビジョニングの前に
``project.yml`` で ``auth_identity`` を設定してください。

たとえば、FLARE のサイト名が ``site-1`` で、その証明書 CN が ``server.example.com`` である場合は
次のようになります。

.. code-block:: yaml

   participants:
     - name: site-1
       type: client
       org: nvidia
       auth_identity: server.example.com

プロビジョニングではこの値を使用して、サーバーおよびジョブセルが必要とするピアアイデンティティの
マッピングを含む、対応するスタートアップキットの構成を生成します。``site-1.<job_id>`` のようなジョブセルも、
引き続き親サイトに設定された証明書アイデンティティで認証されます。

.. important::

   エンドポイントと CN のバインディングは、mTLS トランスポートが認証済みのピア CN を公開している場合に
   適用されます。アクティブ側の gRPC 接続の一部は Python ドライバーにピア CN を公開しないため、
   NVFlare はそれらのアクティブな接続を受け入れ、パッシブ側のエンドポイント検証と通常の
   アプリケーション層の認証チェックに依存します。

   Admin コンソールのセルはセッションごとのエンドポイント名を使用し、管理者ユーザーを admin リスナー上で
   証明書／ユーザーアイデンティティによって認証します。admin のアプリケーション層認証は必ず構成し、
   保護された状態を維持してください。

   ``auth_identity`` は FLARE プロセスの起動時に読み込まれます。サイト証明書をローテーションしたり
   証明書 CN を変更したりした後は、必要に応じてスタートアップキットを再生成し、影響を受ける FLARE プロセスを
   再起動して、メモリ上のアイデンティティリゾルバーが新しい証明書アイデンティティを使用するようにしてください。

.. warning::

   署名済みのスタートアップキットに含まれる ``startup/fed_client.json`` や ``startup/fed_server.json`` を
   手作業で編集しないでください。``startup/signature.json`` が存在する場合、スタートアップ構成は署名の対象に
   含まれており、手作業の編集はその署名を無効にします。``project.yml`` を変更してプロビジョニングを再実行し、
   スタートアップ構成と署名が一緒に生成されるようにしてください。

.. _dynamic_provisioning_cli:

動的プロビジョニング
====================

``--add_user`` および ``--add_client`` オプションを使用すると、既存のプロジェクトに追加できます。
どちらのコマンドも、プロビジョニングする追加の参加者を定義する yaml ファイルを受け取ります。

``--add_user`` 用の user.yaml のサンプル:

.. code-block:: yaml

    name: new_user@nvidia.com
    org: nvidia
    role: project_admin


``--add_client`` 用の client.yaml のサンプル:

.. code-block:: yaml

    name: new-site
    org: nvidia
    components:
      resource_manager:    # This id is reserved by system.  Do not change it.
        path: nvflare.app_common.resource_managers.gpu_resource_manager.GPUResourceManager
        args:
          num_of_gpus: 0
          mem_per_gpu_in_GiB: 0
      resource_consumer:    # This id is reserved by system.  Do not change it.
        path: nvflare.app_common.resource_consumers.gpu_resource_consumer.GPUResourceConsumer
        args:


``--add_user`` または ``--add_client`` に yaml ファイル名を続けて ``nvflare provision`` を実行すると
( :mod:`nvflare.lighter.provision` はカレントディレクトリから yaml ファイルを探します)、
新しいユーザーまたはクライアントが prod_NN フォルダに含まれます。

ユーザーやクライアントを恒久的に含めるには、project.yml を更新してください。

.. note::

   マルチスタディ機能を使用するには、``project.yml`` で ``api_version: 4`` を設定し、``studies:``
   セクションを追加してください。詳細は :ref:`multi_study_guide` を参照してください。
