.. _cc_deployment_guide:

################################################
FLARE 機密連合 AI デプロイメントガイド
################################################

概要
====

本ガイドでは、AMD SEV-SNP CPU と NVIDIA GPU を使用して、機密コンピューティング (CC) 機能を備えた NVIDIA FLARE をオンプレミスに展開するための手順を段階的に説明します。

この展開では、FLARE アプリケーションを含む Confidential VM (CVM) イメージを構築し、参加者向けにシステムをプロビジョニングし、各サイトで CVM を起動します。

デプロイメント環境
==================

本ガイドは以下の展開構成を対象としています。

- **プラットフォーム** : NVIDIA GPU を搭載したオンプレミスの AMD CVM
- **CPU** : AMD SEV-SNP (Secure Encrypted Virtualization - Secure Nested Paging)
- **GPU** : 機密コンピューティングをサポートする NVIDIA GPU (任意)
- **ホスト OS** : Ubuntu 25.04

前提条件
========
ハードウェア IT、ホスト OS の管理、VM の管理を網羅した完全かつ詳細なセットアップガイドについては、 `NVIDIA's Deployment Guide for SecureAI <https://docs.nvidia.com/cc-deployment-guide-snp.pdf>`_ を参照してください。

ハードウェア要件
----------------

**CPU 要件**

- SEV-SNP が有効化された AMD CPU
- SEV-SNP をサポートする AMD ファームウェア

**GPU 要件 (任意)**

- 機密コンピューティングをサポートする NVIDIA GPU (H100、Blackwell)

**ホストシステム**

- ホスト OS: Ubuntu 25.04

ソフトウェア要件
----------------

**必要なソフトウェア**

- Ubuntu 25.04
- QEMU (仮想化用)
- Docker

**NVFlare のコンポーネント**

1. **NVFlare ソースコード**

   GitHub からクローンします。

   .. code-block:: bash

      git clone https://github.com/NVIDIA/NVFlare.git

2. **イメージビルダー**

   イメージビルダーは CVM イメージを構築するツールです。NVFlare のソースツリー内の
   ``nvflare/lighter/cc/image_builder/`` 配下に同梱されており、``cvm_build.sh`` スクリプトのほか、
   Ansible playbook やヘルパースクリプトが含まれます。

   - イメージビルダーのコードは NVFlare チームから入手してください
   - 連絡先: federatedlearning@nvidia.com
   - インストール先: ``~/cc/image_builder``

   このディレクトリ内の ``cvm_build.sh`` へのパスは、``project.yml`` の
   ``build_image_cmd`` から参照されます (デプロイメントワークフローのステップ 2.1 を参照)。

3. **ベースイメージ**

   - :ref:`base_image_build` に従って Ubuntu ベースイメージとファームウェアをビルドします
   - コピー先: ``~/cc/image_builder/base_images``

4. **KBS クライアント**

   - :ref:`base_image_build` に従って kbs-client バイナリをビルドします
   - 推奨コミット: ``a2570329cc33daf9ca16370a1948b5379bb17fbe``
   - kbs-client と認証情報のコピー先: ``~/cc/image_builder/binaries``

5. **SNPGuest ツール**

   - :ref:`base_image_build` に従って snpguest バイナリをビルドします
   - snpguest と認証情報のコピー先: ``~/cc/image_builder/binaries``

**AMD ファームウェアのインストール**

``OVMF.amdsev.fd`` ファームウェアの取得およびインストールに関する完全な手順は :ref:`base_image_build` を参照してください。

ファームウェアは ``/usr/share/ovmf/OVMF.amdsev.fd`` にインストールされます。

**ポリシーファイルのセットアップ**

.. note::

   現在の KBS は個々のルールの更新をサポートしていません。新しい CVM を追加する際は、ルールファイル全体を更新する必要があります。

ポリシーディレクトリを作成し、必要なファイルを取得します。

.. code-block:: bash

   mkdir -p /shared/policy

以下のファイルを ``/shared/policy`` に配置します (NVFlare チームから入手してください)。

- ``policy.rego`` - マスターポリシーファイル
- ``set-policy.sh`` - ポリシー更新スクリプト
- ``private.key`` - 認証キー

プロジェクト管理者に必要な作業
------------------------------

プロジェクト管理者として、以下を行う必要があります。

1. **Trustee サービスの理解**

   - `Trustee Service <https://www.redhat.com/en/blog/introducing-confidential-containers-trustee-attestation-services-solution-overview-and-use-cases>`_ について学習します
   - `Trustee documentation <https://github.com/confidential-containers/trustee?tab=readme-ov-file>`_ を確認します

2. **Trustee KBS サーバーの展開**

   :ref:`hashicorp_vault_trustee_deployment` ガイドに従って、HashiCorp Vault を用いた Trustee Key Broker Service を展開します。

デプロイメントワークフロー
==========================

展開は 5 つの主要なステップで構成されます。

1. **Docker イメージのビルド** - アプリケーションコンテナを作成します
2. **プロビジョニング** - CVM イメージとスタートアップキットを生成します
3. **配布** - スタートアップキットを各サイトに送付します
4. **ユーザーデータ** - ユーザーデータを準備します (任意)
5. **起動** - 各サイトで CVM を起動します

ステップ 1: Docker イメージのビルド
-----------------------------------

CC イメージビルダーは、あらゆる汎用ワークロードをサポートします。NVFlare の場合は、アプリケーションを事前インストールした Docker イメージを作成します。

**Dockerfile の例:**

.. code-block:: dockerfile

   ARG BASE_IMAGE=python:3.12

   FROM ${BASE_IMAGE}

   ENV PYTHONDONTWRITEBYTECODE=1
   ENV PIP_NO_CACHE_DIR=1

   RUN pip install -U pip && \
       pip install nvflare~=2.7.0rc

   COPY code/ /local/custom
   COPY requirements.txt .
   RUN pip install -r requirements.txt

   ENTRYPOINT ["/user_config/nvflare/startup/sub_start.sh", "--verify"]

.. note::

   CC ジョブでは、ランタイムでのカスタムコードは許可されません。すべてのアプリケーションコードは Docker イメージに含める必要があります。
   NVFlare はコンポーネントをロードする前に、コンポーネントの許可リストを確認します。CVM イメージには含まれているものの
   まだ許可されていないクラスをジョブで使用する場合は、それらのクラスパスをサイトの ``cc_config`` にある
   ``class_allow_list`` に追加してください。プロビジョナーは、スタートアップキットが署名・パッケージ化される前に、
   その参加者向けに生成された ``local/resources.json.default`` を拡張します。

   ``cc_config.class_allow_list`` の値は追加的に扱われます。CC イメージが必要とする追加のクラスまたは
   パッケージプレフィックスのみを列挙してください。プロビジョナーは ``resources.json.default`` に組み込まれた
   NVFlare の許可リストのエントリを維持したうえで、まだ含まれていない CC 設定のエントリを追記します。

   このリストは、参照する CC 設定ファイルに直接記述できます。

   .. code-block:: yaml

      compute_env: onprem_cvm
      cc_cpu_mechanism: amd_sev_snp
      role: client

      class_allow_list:
        - hello_cyclic.app.custom.trainer.SimpleTrainer
        - hello_cyclic.

   正確なクラスパス、または ``.`` で終わるパッケージプレフィックスを使用してください。たとえば
   ``hello_cyclic.app.custom.trainer.SimpleTrainer`` は 1 つのクラスを許可し、``hello_cyclic.`` はそのパッケージ
   プレフィックス配下のクラスを許可します。

**イメージのビルドと保存:**

.. code-block:: bash

   docker build -t nvflare-site:latest .
   docker save nvflare-site:latest | gzip > nvflare-site.tar.gz

ステップ 2: プロビジョニング
----------------------------

サンプルディレクトリに移動します。

.. code-block:: bash

   cd NVFlare/examples/advanced/cc_provision

**2.1 プロジェクトの設定**

``project.yml`` を編集し、``build_image_cmd`` のパスを更新します。

.. code-block:: yaml

   packager:
     path: nvflare.lighter.cc_provision.impl.onprem_packager.OnPremPackager
     args:
       # Update this path to your image builder location
       build_image_cmd: ~/nvflare-github/nvflare/lighter/cc/image_builder/cvm_build.sh

**2.2 サーバーの設定**

``cc_server1.yml`` を編集し、``docker_archive`` のパスを設定します。

.. code-block:: yaml

   docker_archive: ~/NVFlare/examples/advanced/cc_provision/docker/nvflare-site.tar.gz

**2.3 クライアントの設定**

``cc_site-1.yml`` を編集します。

1. ``docker_archive`` のパスを設定します。

   .. code-block:: yaml

      docker_archive: ~/NVFlare/examples/advanced/cc_provision/docker/nvflare-site.tar.gz

2. サーバー名が公開ドメインでない場合は、ホストエントリを追加します。

   .. code-block:: yaml

      host_entries:
        server1: 10.176.4.244


**2.4 プロビジョニングの実行**

プロビジョニングを実行する前に、ビルダーがディスクイメージにアクセスするために使用する
NBD (Network Block Device) デバイスが利用可能であることを確認してください。

.. code-block:: bash

   sudo modprobe nbd max_part=8

.. code-block:: bash

   nvflare provision -p project.yml

.. note::

   プロビジョニングでは、CVM イメージ 1 つあたりのビルドに約 1000 秒かかります。

**2.5 出力**

スタートアップパッケージは以下に生成されます。

.. code-block:: text

   ./workspace/example_project/prod_00/
      server1/server1.tgz
      site-1/site-1.tgz
      admin@nvidia.com/

ステップ 3: 配布
----------------

生成されたスタートアップキットを各参加者に配布します。

- ``server1.tgz`` をサーバーサイトに送付します
- ``site-1.tgz`` をクライアント site-1 に送付します
- 管理者は admin パッケージをローカルに保持します

CVM スタートアップキットの内容
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

各スタートアップキット (``server1.tgz`` など) には以下が含まれます。

.. code-block:: bash

   $ tar -zxvf server1.tgz
   $ ls server1/cvm_885fe8f608b3/

.. list-table::
   :header-rows: 1
   :widths: 30 70

   * - ファイル
     - 説明
   * - ``applog.qcow2``
     - アプリケーションログの保存領域 (暗号化されておらず、マウントして内容を確認できます)
   * - ``crypt_root.qcow2``
     - 暗号化されたルートファイルシステム (復号鍵が必要です)
   * - ``initrd.img``
     - アテステーション用の InitApp を含む initramfs
   * - ``launch_vm.sh``
     - CVM 起動スクリプト
   * - ``OVMF.amdsev.fd``
     - kernel-hashes をサポートする AMD SEV-SNP ファームウェア
   * - ``README.txt``
     - ドキュメント
   * - ``user_config.qcow2``
     - NVFlare のスタートアップキットを含むユーザー設定の保存領域
   * - ``user_data.qcow2``
     - ユーザーデータの保存領域 (プレースホルダー。拡張可能です)
   * - ``vmlinuz``
     - Linux カーネル

.. note::

   ``user_config.qcow2`` と ``user_data.qcow2`` のドライブは暗号化されていません。プロビジョニングされた NVFlare の
   起動スクリプトが ``/user_config`` から起動されると、モデルチェックポイントなどのランタイム成果物は
   ``/vault/workspace`` 配下の名前空間分離された暗号化ワークスペースに書き込まれます。別の暗号化ワークスペースのパスを
   選択するには、スクリプトを起動する前に ``NVFL_WORKSPACE`` を設定してください。

ステップ 4: ユーザーデータ
--------------------------

このステップは任意です。ここでは、ユーザーデータを準備し、CVM とワークロードコンテナの両方から利用できるようにする方法を説明します。
ワークロードがユーザーデータを必要としない場合は、このステップを省略してかまいません。

**4.1 ユーザーデータドライブ**

CVM の配布パッケージには、user_data.qcow2 という名前のユーザーデータドライブイメージが含まれています。
このイメージには、ドライブ全体を占める単一の ext4 ファイルシステムが含まれます (パーティションはありません)。

このドライブは暗号化されていません。VM に接続するか、qemu-nbd コマンドを使用することでアクセスできます。例:

.. code-block:: bash

    sudo qemu-nbd --connect=/dev/nbd0 user_data.qcow2
    sudo mount /dev/nbd0 /mnt

  .. note::

    使用後は以下を実行してドライブをアンマウントしてください。

    .. code-block:: bash

        sudo umount /mnt
        sudo qemu-nbd --disconnect /dev/nbd0

**4.2 ローカルデータ**

データがローカルホスト上にある場合は、user_data ドライブに直接コピーできます。データは CVM とコンテナの中で
``/user_data`` として利用できます。

例:

.. code-block:: bash

    cp -r /training_data /mnt

パッケージに同梱されている user_data.qcow2 イメージはプレースホルダーであり、容量が非常に小さくなっています (1 GB)。
以下のコマンドでドライブのサイズを変更し、ファイルシステムを拡張できます。

.. code-block:: bash

    # Add 20GB to the drive
    qemu-img resize user_data.qcow2 +20G

    # Extend filesystem
    sudo qemu-nbd --connect=/dev/nbd0 user_data.qcow2
    sudo e2fsck -f /dev/nbd0
    sudo resize2fs /dev/nbd0
    sudo qemu-nbd --disconnect /dev/nbd0

**4.3 NFS を用いたリモートデータ**

データが NFS サーバー上にホストされている場合、user_data.qcow2 ドライブのルートに ``ext_mount.conf`` という名前の
ファイルが存在すれば、CVM がそれを自動的にマウントします。

マウントされたデータは、CVM とコンテナの中で ``/user_data/mnt`` として利用できます。

このファイルには、エクスポートされたサーバーパスを指定する 1 行のみを記述します。例:

.. code-block:: bash

    nfs-server.example.com:/training_data

既定では、CVM はすべてのアウトバウンドネットワークトラフィックをブロックします。NFS および portmapper の通信を許可するには、
CVM のサイト設定で以下のアウトバウンドポートを有効にする必要があります。

.. code-block:: yaml

    allowed_out_ports: [111, 2049]

この設定は、プロビジョニングプロセスのステップ 2.3 で行う必要があります。

CVM は NFS 接続に使用される送信元ポートを制御できないため、secure な NFS エクスポートはサポートされません。
したがって、サーバーのエクスポートは insecure として設定する必要があります。

.. code-block:: bash

    /training_data *(rw,sync,no_subtree_check,insecure)

詳細については `exports man page <https://manpages.ubuntu.com/manpages/jammy/man5/exports.5.html>`_ を参照してください。

ステップ 5: CVM の起動
----------------------

**5.1 サーバーの起動**

サーバーマシン上で実行します。

.. code-block:: bash

   tar -zxvf server1.tgz
   cd server1/cvm_*
   ./launch_vm.sh

**5.2 クライアントの起動**

各クライアントマシン上で実行します。

.. code-block:: bash

   tar -zxvf site-1.tgz
   cd site-1/cvm_*
   ./launch_vm.sh

サーバーとクライアントは、それぞれの CVM 内で NVFlare システムを自動的に開始します。

**5.3 管理コンソールの起動**

管理用マシン上で実行します。

.. code-block:: bash

   cd NVFlare/examples/advanced/cc_provision

   # Copy jobs to admin transfer folder
   cp -r jobs/* ./workspace/example_project/prod_00/admin@nvidia.com/transfer/

.. note::

   サーバー名が公開ドメインでない場合は、管理用マシンの ``/etc/hosts`` にエントリを追加してください。

管理コンソールを起動します。

.. code-block:: bash

   ./workspace/example_project/prod_00/admin@nvidia.com/startup/fl_admin.sh

**5.4 ジョブの投入**

管理コンソールで実行します。

.. code-block:: bash

   submit_job hello-pt_cifar10_fedavg

設定リファレンス
================

CC 設定パラメータ
-----------------

.. list-table::
   :header-rows: 1
   :widths: 25 25 50

   * - パラメータ
     - 設定値の例
     - 説明
   * - ``compute_env``
     - ``onprem_cvm``
     - 計算環境の種別
   * - ``cc_cpu_mechanism``
     - ``amd_sev_snp``
     - CPU の機密コンピューティング機構
   * - ``role``
     - ``server`` / ``client``
     - NVFlare システムにおける役割
   * - ``root_drive_size``
     - ``45`` (GB)
     - ルートファイルシステムドライブのサイズ
   * - ``applog_drive_size``
     - ``1`` (GB)
     - アプリケーションログドライブのサイズ
   * - ``user_config_drive_size``
     - ``1`` (GB)
     - ユーザー設定ドライブのサイズ
   * - ``user_data_drive_size``
     - ``1`` (GB)
     - ユーザーデータドライブのサイズ
   * - ``docker_archive``
     - ``~/path/to/app.tar.gz``
     - Docker イメージアーカイブへのパス (``docker save`` で作成)
   * - ``user_config``
     - キーと値のペア
     - コンテナ内の ``/user_config/[key]`` にマウントされるパス
   * - ``allowed_ports``
     - ポートのリスト
     - ホワイトリストに登録するインバウンドポート
   * - ``allowed_out_ports``
     - ポートのリスト
     - ホワイトリストに登録するアウトバウンドポート
   * - ``cc_issuers``
     - authorizer のリスト
     - CC アテステーショントークンの発行者
   * - ``class_allow_list``
     - クラスパスのリスト
     - 非 BYOC の CC ジョブで許可される追加のコンポーネントクラスまたはパッケージプレフィックスの追加リスト
   * - ``token_expiration``
     - ``100`` (秒)
     - トークンの有効期間 (``check_frequency`` より小さい必要があります)
   * - ``check_frequency``
     - ``120`` (秒)
     - アテステーションのチェック間隔

完全な設定例
------------

**プロジェクト設定 (project.yml)**

.. code-block:: yaml

   api_version: 3
   name: example_project
   description: NVIDIA FLARE sample project yaml file

   participants:
     - name: server1
       type: server
       org: nvidia
       fed_learn_port: 8002
       cc_config: cc_server1.yml
     - name: site-1
       type: client
       org: nvidia
       cc_config: cc_site-1.yml
     - name: admin@nvidia.com
       type: admin
       org: nvidia
       role: project_admin

   builders:
     - path: nvflare.lighter.impl.workspace.WorkspaceBuilder
     - path: nvflare.lighter.impl.static_file.StaticFileBuilder
       args:
         config_folder: config
     - path: nvflare.lighter.impl.cert.CertBuilder
     - path: nvflare.lighter.cc_provision.impl.cc.CCBuilder
     - path: nvflare.lighter.impl.signature.SignatureBuilder

   packager:
     path: nvflare.lighter.cc_provision.impl.onprem_packager.OnPremPackager
     args:
       build_image_cmd: ~/nvflare-github/nvflare/lighter/cc/image_builder/cvm_build.sh

**サーバー設定 (cc_server1.yml)**

.. code-block:: yaml

   compute_env: onprem_cvm
   cc_cpu_mechanism: amd_sev_snp
   role: server

   # All drive sizes are in GB
   root_drive_size: 45
   applog_drive_size: 1
   user_config_drive_size: 1
   user_data_drive_size: 1

   # Docker image archive saved using:
   # docker save <image_name> | gzip > app.tar.gz
   docker_archive: ~/NVFlare/examples/advanced/cc_provision/docker/nvflare-site.tar.gz

   # Will be mounted inside docker at "/user_config/nvflare"
   user_config:
     nvflare: /tmp/startup_kits

   # Inbound ports whitelist
   allowed_ports:
     - 8002

   # Outbound ports whitelist
   allowed_out_ports:
     - 443    # HTTPS
     - 8002   # NVFlare
     - 8999   # Trustee KBS

   cc_issuers:
     - id: snp_authorizer
       path: nvflare.app_opt.confidential_computing.snp_authorizer.SNPAuthorizer
       token_expiration: 100  # seconds, must be < check_frequency

   cc_attestation:
     check_frequency: 120  # seconds

**クライアント設定 (cc_site-1.yml)**

.. code-block:: yaml

   compute_env: onprem_cvm
   cc_cpu_mechanism: amd_sev_snp
   role: client

   class_allow_list:
     - hello_cyclic.app.custom.trainer.SimpleTrainer
     - hello_cyclic.

   # All drive sizes are in GB
   root_drive_size: 45
   applog_drive_size: 1
   user_config_drive_size: 1
   user_data_drive_size: 1

   # Docker image archive
   docker_archive: ~/NVFlare/examples/advanced/cc_provision/docker/nvflare-site.tar.gz

   # For non-public domain server names
   hosts_entries:
     server1: 10.176.200.152

   # Will be mounted inside docker at "/user_config/nvflare"
   user_config:
     nvflare: /tmp/startup_kits

   cc_issuers:
     - id: snp_authorizer
       path: nvflare.app_opt.confidential_computing.snp_authorizer.SNPAuthorizer
       token_expiration: 100  # seconds, must be < check_frequency

   cc_attestation:
     check_frequency: 120  # seconds

トラブルシューティング
======================

よくある問題
------------

**問題: CVM がブートしない**

- ``applog.qcow2`` でブートログを確認します
- ファームウェアが正しくインストールされているか検証します
- ファームウェアで kernel-hashes が有効になっていることを確認します

**問題: アテステーションの失敗**

- Trustee KBS サーバーにアクセスできるか検証します
- アテステーションサービスへのネットワーク接続を確認します
- ``allowed_out_ports`` で正しいポートがホワイトリストに登録されていることを確認します

**問題: サーバーとクライアントの接続に失敗する**

- 公開ドメインを使用していない場合は ``/etc/hosts`` のエントリを検証します
- ファイアウォールのルールを確認します
- サーバーとクライアントの両方で正しいポートが設定されていることを確認します

NVIDIA GPU CC を使用する際の注意点
==================================

1. GPU CC をサポートするサイトでは、NVFLARE の `GPUAuthorizer` を `cc_site.yml` 設定ファイルに追加できます。

.. code-block:: yaml

    cc_issuers:
      ...
      - id: gpu_authorizer
        path: nvflare.app_opt.confidential_computing.gpu_authorizer.GPUAuthorizer
        token_expiration: 100 # seconds, needs to be less than check_frequency

2. NVFlare の `GPUAuthorizer` は NVIDIA の `nv_attestation_sdk` を使用します。
   NVFlare アプリケーションの Docker イメージをビルドする際は、以下のように requirements に含めるようにしてください。

.. code-block:: bash

    torch
    torchvision
    tensorboard
    tensorflow
    safetensors
    nv_attestation_sdk

3. CVM 内で GPU を動作させるには、以下を確認する必要があります。
       - ホストに GPU ドライバがインストールされていないこと。インストールされているとパススルーが失敗します。
       - 以下のコマンドを実行して VFIO を作成する必要があります。

.. code-block:: bash

    NVIDIA_GPU=$(lspci -d 10de: | awk '/NVIDIA/{print $1}')
    NVIDIA_PASSTHROUGH=$(lspci -n -s $NVIDIA_GPU | awk -F: '{print $4}' | awk '{print $1}')
    echo 10de $NVIDIA_PASSTHROUGH > /sys/bus/pci/drivers/vfio-pci/new_id

4. 詳細については `NVIDIA's Deployment Guide for SecureAI <https://docs.nvidia.com/cc-deployment-guide-snp.pdf>`_ を参照してください。

次のステップ
============

システムの展開に成功したら、以下を行ってください。

- セキュリティモデルを理解するために :ref:`NVFlare CC アーキテクチャ <cc_architecture>` を確認します
- アテステーションの詳細については :ref:`confidential_computing_attestation` を参照します
- 個別のユースケースに応じた高度な設定オプションを検討します
