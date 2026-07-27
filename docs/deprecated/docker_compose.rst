:orphan:

.. _docker_compose:

.. deprecated:: 2.7
   Docker Compose によるデプロイメントは非推奨です。現在のコンテナデプロイメントの選択肢については :ref:`containerized_deployment` を参照してください。

######################################################
docker compose による NVIDIA FLARE の起動
######################################################

.. note::
    注記: 非推奨です。これはローカル環境でデプロイメントをシミュレートする
    ための代替手段です。本番環境では使用しないでください。

初めて NVIDIA FLARE を使うユーザーや、求めに応じてデモを行う必要がある
ユーザーなど、できるだけ簡単に NVIDIA FLARE を起動して動かしたい場合は、
この docker compose 機能を利用できます。必要なのは動作する docker 環境
だけです。

NVIDIA FLARE のプロビジョニングツールには ``DockerBuilder`` が含まれて
おり、``compose.yaml`` などの情報を作成できます。
プロビジョニング後、ユーザーは結果フォルダ(通常は
workspace/example_project/prod_NN)に移動し、``docker compose build``
と ``docker compose up`` を入力することで、docker compose 方式でサーバーと
クライアントを起動できます。


プロビジョニング段階
========================
まず、project.yml ファイルに次のセクションが含まれているか確認してください。

.. code-block:: yaml

  - path: nvflare.lighter.impl.docker.DockerBuilder
    args:
      base_image: python:3.8
      requirements_file: docker_compose_requirements.txt


このビルダーは、プロビジョニング時に必要な情報を生成します。

``base_image`` 引数は、docker compose 構成における NVIDIA FLARE のランタイム
docker イメージを作成する際に使用されるベース docker イメージ名です。

``requirements_file`` には、ランタイム docker イメージに nvflare パッケージが
インストールされた後にインストールされる追加の python パッケージを記述できます。
追加の python パッケージをインストールする必要がなければ、空のファイルを指定
できます。


プロビジョニング後の段階
================================

通常どおり、新しい形式の ``nvflare provision`` または単に ``provision`` で
provision コマンドを実行します。

コマンドの実行後、次のような構造のフォルダが作成されているはずです。

.. code-block:: shell

    $ tree -L 1
    .
    ├── admin@nvidia.com
    ├── compose.yaml
    ├── nvflare_compose
    ├── nvflare_hc
    ├── server1
    ├── site-1
    └── site-2

    6 directories, 1 file


``compose.yaml`` は docker compose コマンドの中核となるファイルで、
``nvflare_compose`` フォルダは ``docker compose build`` 段階でランタイム
docker イメージを生成するための compose コンテキストフォルダです。

``nvflare_compose`` の中身は ``Dockerfile`` と ``requirements.txt`` の
2ファイルのみです。必要に応じて変更できます。たとえば、``apt-get install``
で追加のバイナリパッケージをインストールする必要がある場合は、Dockerfile に
追記できます。

``requirements.txt`` は、project.yml ファイルで指定した requirements_file の
コピーです。


docker compose の実行
=======================

prod_NN フォルダ内で、NVIDIA FLARE の docker compose を初めて起動する場合は、
``docker compose build`` を実行してランタイム docker イメージをビルドして
ください。Dockerfile と requirements.txt に変更がなければ、このコマンドを
再度実行する必要はありません。

.. code-block:: shell

    $ docker compose build
    [+] Building 0.1s (10/10) FINISHED                                                                                                                                                                                                       
    => [internal] load build definition from Dockerfile                                                                                                                                                                                0.0s
    => => transferring dockerfile: 177B                                                                                                                                                                                                0.0s
    => [internal] load .dockerignore                                                                                                                                                                                                   0.0s
    => => transferring context: 2B                                                                                                                                                                                                     0.0s
    => [internal] load metadata for docker.io/library/python:3.8                                                                                                                                                                       0.0s
    => [1/5] FROM docker.io/library/python:3.8                                                                                                                                                                                         0.0s
    => [internal] load build context                                                                                                                                                                                                   0.0s
    => => transferring context: 37B                                                                                                                                                                                                    0.0s
    => CACHED [2/5] RUN pip install -U pip                                                                                                                                                                                             0.0s
    => CACHED [3/5] RUN pip install nvflare                                                                                                                                                                                            0.0s
    => CACHED [4/5] COPY requirements.txt requirements.txt                                                                                                                                                                             0.0s
    => CACHED [5/5] RUN pip install -r requirements.txt                                                                                                                                                                                0.0s
    => exporting to image                                                                                                                                                                                                              0.0s
    => => exporting layers                                                                                                                                                                                                             0.0s
    => => writing image sha256:53a1463bd170b8bc213899037bbe4403f2d6f0d553cdd470805855f3968d19d4                                                                                                                                        0.0s
    => => naming to docker.io/library/nvflare-service                                                                                                                                                                                  0.0s

ランタイム docker イメージの準備ができたら、``docker compose up`` を実行する
ことで、1つのサーバーと2つのサイトを一緒に稼働させることができます。サーバー用の
ポートも開かれます。現在の prod_NN フォルダ内のサーバーおよびクライアントの
フォルダは、それぞれ異なる稼働中の docker インスタンスにマウントされます。

.. code-block:: shell

    $ docker compose up
    [+] Running 3/0
    ⠿ Container server1  Recreated
    ⠿ Container site-1   Recreated
    ⠿ Container site-2   Recreated
    Attaching to server1, site-1, site-2
    server1 | 2022-09-23 16:00:59,332 - FederatedServer - INFO - starting secure server at server1:8002
    server1 | deployed FL server trainer.
    server1 | 2022-09-23 16:00:59,346 - nvflare.fuel.hci.server.hci - INFO - Starting Admin Server server1 on Port 8003
    server1 | 2022-09-23 16:00:59,346 - root - INFO - Server started
    site-2  | 2022-09-23 16:00:59,399 - FederatedClient - INFO - Got server address: server1:8002
    site-1  | 2022-09-23 16:00:59,450 - FederatedClient - INFO - Got server address: server1:8002
    server1 | 2022-09-23 16:01:00,393 - ClientManager - INFO - Client: New client site-2@172.18.0.2 joined. Sent token: 3da72f67-3443-47ac-b059-76b0b314dd08.  Total clients: 1
    site-2  | 2022-09-23 16:01:00,394 - FederatedClient - INFO - Successfully registered client:site-2 for project example_project. Token:3da72f67-3443-47ac-b059-76b0b314dd08 SSID:9ba168f0-6cf5-446b-bfd5-a1243dd195f8
    server1 | 2022-09-23 16:01:00,439 - ClientManager - INFO - Client: New client site-1@172.18.0.3 joined. Sent token: 5e0b1012-77e6-41a3-8af0-9fa86df8ef2e.  Total clients: 2
    site-1  | 2022-09-23 16:01:00,440 - FederatedClient - INFO - Successfully registered client:site-1 for project example_project. Token:5e0b1012-77e6-41a3-8af0-9fa86df8ef2e SSID:9ba168f0-6cf5-446b-bfd5-a1243dd195f8

管理コンソールでのログイン
================================
使用しているマシンがサーバーの IP アドレスを解決できるようになれば、管理
コンソールを使って、新しく作成されたこの NVIDIA FLARE システムにログイン
できます。たとえば、IP アドレス 192.168.1.101 のマシン ``desktop1`` で
docker compose を実行しており、マシン ``desktop2`` で管理コンソールを実行
したい場合は、desktop2 の /etc/hosts ファイルを編集して次の行を追加する
必要があります。

.. code-block::

    192.168.1.101 server1

この更新後、管理コンソールは server1 を見つけられるようになります。
project.yml ファイルでサーバーに別の名前(たとえば myserver)を付けている
場合は、その行を次のように変更してください。

.. code-block::

    192.168.1.101 myserver


管理コンソールでのログインは通常どおりです。管理コンソールの startup
フォルダ内の fl_admin.sh を実行するだけです。

.. code-block:: shell
    
    $ ./admin@nvidia.com/startup/fl_admin.sh 
    User Name: admin@nvidia.com
    Trying to obtain server address
    Obtained server address: server1:8003
    Trying to login, please wait ...
    Logged into server at server1:8003
    Type ? to list commands; type "? cmdName" to show usage of a command.
    > check_status server
    Engine status: stopped
    ---------------------
    | JOB_ID | APP NAME |
    ---------------------
    ---------------------
    Registered clients: 2 
    ----------------------------------------------------------------------------
    | CLIENT | TOKEN                                | LAST CONNECT TIME        |
    ----------------------------------------------------------------------------
    | site-2 | 7cfe5dce-00a5-4ffb-a5ad-d31dc050c5dd | Fri Sep 23 16:15:00 2022 |
    | site-1 | 5435ccb6-9240-42b1-a48b-6290cc71d8d0 | Fri Sep 23 16:15:00 2022 |
    ----------------------------------------------------------------------------
    Done [9729 usecs] 2022-09-23 09:15:12.137237

docker compose の終了
==========================

``CTRL-C`` を押すと docker compose を停止できます。
