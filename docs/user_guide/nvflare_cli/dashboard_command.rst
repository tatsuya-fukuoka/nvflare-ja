*****************************************
ダッシュボードコマンド
*****************************************

ダッシュボードコマンドの概要
=====================================

ダッシュボードコマンドを使うと、 :ref:`dashboard_api` を起動して、さまざまな組織のクライアントとユーザーの情報を簡単に収集し、ユーザーがダウンロードできるスタートアップキットを生成できます。

構文と使い方
=================

``nvflare dashboard -h`` を実行すると、利用可能なすべてのオプションが表示されます。

.. code-block:: shell

    (nvflare_venv) ~/workspace/repos/flare$ nvflare dashboard -h
    usage: nvflare dashboard [-h] [--cloud CLOUD] [--start] [--stop] [-p PORT]
                              [-f FOLDER] [--passphrase PASSPHRASE] [-e ENV]
                              [--cred CRED] [-i IMAGE] [--local]
                              [--vpc-id VPC_ID] [--subnet-id SUBNET_ID]

    options:
    -h, --help            show this help message and exit
    --cloud CLOUD         launch dashboard on cloud service provider (ex:
                          --cloud azure or --cloud aws)
    --start               start dashboard
    --stop                stop dashboard
    -p PORT, --port PORT  port to listen
    -f FOLDER, --folder FOLDER
                          folder containing necessary info (default: current
                          working directory)
    --passphrase PASSPHRASE
                          Passphrase to encrypt/decrypt root CA private key. !!!
                          Do not share it with others. !!!
    -e ENV, --env ENV     additional environment variables: var1=value1
    --cred CRED           set credential directly in the form of
                          USER_EMAIL:PASSWORD
    -i IMAGE, --image IMAGE
                          set the container image name (required for --start
                          and --cloud)
    --local               start dashboard locally without docker image
    --vpc-id VPC_ID       VPC id for AWS EC2 instance. Applicable to AWS only.
                          Ignored if subnet-id is not specified.
    --subnet-id SUBNET_ID
                          Subnet id for AWS EC2 instance. Applicable to AWS
                          only. Ignored if vpc-id is not specified.

.. note::

    ``-i`` / ``--image`` オプションは、Docker でダッシュボードを起動する場合や、クラウド上でダッシュボードを起動する場合に必須です。 ``--stop`` や ``--local`` では必要ありません。

    AWS クラウドで起動する場合は、 ``--vpc-id`` と ``--subnet-id`` を必ず一緒に指定してください。どちらか一方のオプションしか指定されていない場合、ダッシュボードはそのオプションを無視します。

ダッシュボードを起動するには、 ``nvflare dashboard --start -i nvflare/nvflare:2.7.2`` を実行します。
このイメージは標準的なコンテナイメージ参照であり、ランタイムがプルできる任意のレジストリのものを使用できます。
たとえば ``nvflare/nvflare:2.7.2`` 、 ``nvcr.io/nvidia/nvflare:2.7.2`` 、
``registry.example.com/nvflare/nvflare:2.7.2`` などです。各 Docker ホストまたはクラウド VM がプルアクセスを持っている限り、
デプロイごとに異なるレジストリのイメージ名を使用できます。

ダッシュボードの Docker は、データベースが初期化されているかどうかを検出します。初期化されていない場合は、project_admin のメールアドレスの入力を求め、ランダムなパスワードを生成します:

.. code-block::

    Please provide project admin email address.  This person will be the super user of the dashboard and this project.
    project_admin@admin_organization.com
    generating random password
    Project admin credential is project_admin@admin_organization.com and the password is EXAMPLE1

システムが起動したら、この資格情報でログインして、ダッシュボードでのプロジェクト設定を完了してください。
project_admin はログイン後、ダッシュボードシステム上で自分のパスワードを変更できます。

初回は、次のプロンプトが表示されるとおり、nvflare イメージのダウンロードに時間がかかる場合があることに注意してください:

.. code-block::

    Pulling nvflare/nvflare:2.7.2, may take some time to finish.

イメージのプルが完了すると、次のような出力が表示されます:

.. code-block::

    Launching nvflare/nvflare:2.7.2
    Dashboard will listen to port 443
    /path_to_folder_for_db on host mounted to /var/tmp/nvflare/dashboard in container
    No additional environment variables set to the launched container.
    Dashboard container started
    Container name nvflare-dashboard
    id is 3108eb7be20b92ab3ec3dd7bfa86c2eb83bd441b4da0865d2ebb10bd60612345

ルート CA の秘密鍵を保護するために、 ``--passphrase`` オプションでパスフレーズを設定することをお勧めします。いったん設定すると、同じプロジェクトでダッシュボードを再起動するたびに、同じパスフレーズを指定する必要があります。

新しいプロジェクトを開始したい場合は、現在の作業ディレクトリ（または ``--folder`` オプションで設定したディレクトリ）にある db.sqlite ファイルを削除してください。ダッシュボードは最初から開始され、
プロジェクト管理者のメールアドレスを指定して project_admin の新しいパスワードを取得できます。

ダッシュボードは、現在の作業ディレクトリ（または --folder オプションで指定したディレクトリ）内の cert フォルダも確認し、web.crt と web.key を読み込みます。
これらのファイルが存在する場合、ダッシュボードはそれらを読み込んで HTTPS サーバーとして動作します。両方が見つからない場合は HTTP サーバーとして動作します。いずれの場合も、 ``--port`` オプションで別のポートを指定しない限り、サービスはポート 443 をリッスンします。ダッシュボードは ``0.0.0.0`` で動作するため、デフォルトでは同じマシンから
``localhost:443`` でアクセスできます。ネットワーク外のユーザーが利用できるようにするには、ダッシュボードを実行しているマシンへ安全にトラフィックを転送するために、ポートフォワーディングやその他の設定が必要になる場合があります。

.. note::

    ダッシュボードの実行には Docker が必要です。システムが Docker イメージをプルして実行できることを確認する必要があります。最初の docker pull は、ネットワーク接続によっては時間がかかる場合があります。

実行中のダッシュボードを停止するには、 ``nvflare dashboard --stop`` を実行します。
