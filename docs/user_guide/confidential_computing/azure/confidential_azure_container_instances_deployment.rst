.. _confidential_azure_container_instances_deployment:

##########################################################
機密 Azure Container Instances デプロイガイド
##########################################################

概要
====

このガイドでは、機密 Azure Container Instances (ACI) の完全なデプロイ手順を順を追って説明します。
この種のデプロイは、ログインの無効化、インスタンス起動前のコンテナイメージの検証、その他の機密コンピューティング機能といった
機密 ACI の機能を活用して、NVFlare サーバーがセキュアアグリゲーションを実行できるようにするために使用されます。

.. note::

   機密 ACI を起動するには、お使いの Azure アカウントに特定の権限が必要です。Azure アカウントおよび Azure の情報をご確認ください。


機密 ACI で NVFlare サーバーを起動する手順
================================================

* ログインしてリソースグループを 1 つ作成する
* Azure Container Registry を 1 つ作成する
* コンテナイメージをビルドする
* コンテナイメージを Azure Container Registry に公開する
* 機密 ACI 起動スクリプトを作成して実行する

ログインとリソースグループの作成
------------------------------------------

まず、az cli を使って Azure にログインする必要があります。その後、以降の操作で生成されるすべてのリソースを
格納するためのリソースグループを 1 つ作成できます。

リソースグループには別の名前を、ロケーションには別の場所を選択することもできます。

.. code-block:: bash

   #!/usr/bin/env bash

   resource_group=cc-prep-rsr-grp
   location=eastus

   az login

   az group create --name $resource_group --location $location


Azure Container Registry (ACR) の作成
---------------------------------------------

リソースグループを作成したら、まずビルドしたイメージのプッシュ先およびプル元となる Azure Container Registry (ACR) を 1 つ作成します。


.. code-block:: bash

   #!/usr/bin/env bash

   resource_group=cc-prep-rsr-grp
   location=eastus
   reg_name=ccprepreg
   dnl_scope=Unsecure

   az group create --name $resource_group --location $location

   az acr create --resource-group $resource_group \
   --name $reg_name --sku Standard \
   --dnl-scope $dnl_scope

.. note::

   後の手順では、より高い権限で Azure Container Registry (ACR) を操作する必要があるため、
   ACR に対して少なくとも "Contributor" ロールを持っているかどうかを確認してください。権限の低いロールでは以降の手順が失敗する可能性があります。

Docker コンテナイメージのビルド
----------------------------------------

これからビルドするコンテナイメージには、NVFlare サーバーのスタートアップキットが含まれます。そのため、
そのファイル一式を入手し、スタートアップキットを現在の作業ディレクトリ内の 1 つのフォルダーにコピーしてください。以下の
例では、サーバーのスタートアップキットと Dockerfile が、それぞれ現在の作業フォルダー配下の
nvflserver.eastus.azurecontainer.io フォルダーと docker フォルダーに格納されています(下記参照)。

.. code-block ::

   $ tree -d -L 1
   .
   ├── docker
   └── nvflserver.eastus.azurecontainer.io


.. code-block:: bash

   #!/usr/bin/env bash

   tag=0.0.1
   name=cc_prep
   reg_name=ccprepreg
   registry=${reg_name}.azurecr.io
   nvfl_root=nvflserver.eastus.azurecontainer.io

   docker build --build-arg NVFL_ROOT=$nvfl_root -t $registry/$name:$tag -f docker/Dockerfile .

.. code-block:: dockerfile

   FROM python:3.10
   ARG NVFL_ROOT=nvflserver.eastus.azurecontainer.io
   WORKDIR /workspace
   RUN python3 -m pip install --no-cache nvflare
   COPY $NVFL_ROOT nvflare


以下は cc_server.yml ファイルのサンプルで、cc プロビジョニングのために project.yml と組み合わせて使用します。project.yml ファイルの
サンプルは cc_server.yml ファイルの後に示します。なお、このサンプルの project.yml ファイルには
:ref:`azure_confidential_virtual_machine_deployment` - Azure 機密仮想マシンの作成 で説明されている cc_site-1.yml も含まれています。

.. code-block:: yaml

  compute_env: azure_confidential_container
  cc_cpu_mechanism: amd_sev_snp
  role: server
  cc_issuers:
    - id: aci_authorizer
      path: nvflare.app_opt.confidential_computing.aci_authorizer.ACIAuthorizer
      token_expiration: 100 # seconds, needs to be less than check_frequency

以下は project.yml ファイルのサンプルです。

.. code-block:: yaml

  api_version: 3
  name: example_project
  description: NVIDIA FLARE sample project yaml file
  participants:
    # Change the name of the server (server1) to the Fully Qualified Domain Name
    # (FQDN) of the server, for example: server1.example.com.
    # Ensure that the FQDN is correctly mapped in the /etc/hosts file.
    - name: server1
      type: server
      org: nvidia
      fed_learn_port: 8002
      cc_config: cc_server.yml
    - name: site-1
      type: client
      org: nvidia
      cc_config: cc_site-1.yml
      # Specifying listening_host will enable the creation of one pair of
      # certificate/private key for this client, allowing the client to function
      # as a server for 3rd-party integration.
      # The value must be a hostname that the external trainer can reach via the network.
      # listening_host: site-1-lh
    - name: admin@nvidia.com
      type: admin
      org: nvidia
      role: project_admin
  # The same methods in all builders are called in their order defined in builders section
  builders:
    - path: nvflare.lighter.impl.workspace.WorkspaceBuilder
    - path: nvflare.lighter.impl.static_file.StaticFileBuilder
      args:
        # config_folder can be set to inform NVIDIA FLARE where to get configuration
        config_folder: config
        # scheme for communication driver (currently supporting the default, grpc, only).
        # scheme: grpc

        # app_validator is used to verify if uploaded app has proper structures
        # if not set, no app_validator is included in fed_server.json
        # app_validator: PATH_TO_YOUR_OWN_APP_VALIDATOR
    - path: nvflare.lighter.impl.cert.CertBuilder
    - path: nvflare.lighter.cc_provision.impl.cc.CCBuilder
    - path: nvflare.lighter.impl.signature.SignatureBuilder

コンテナイメージを Azure Container Registry に公開する
--------------------------------------------------------------

上記の手順により、コンテナイメージが 1 つビルドされてローカルに保存されました。ここでは、手順 2 で作成した ACR にそれをプッシュします。この手順では、
ACR からアクセストークンを取得し、それを使って ACR にログインする必要があります。


.. code-block:: bash

   #!/usr/bin/env bash

   reg_name=ccprepreg
   reg_token_file=reg_token.json

   az acr login --name $reg_name --expose-token > $reg_token_file

   echo "ACR reg token saved to $reg_token_file"


トークンファイルが利用可能になれば、ACR にログインできます。

.. code-block:: bash

   #!/usr/bin/env bash

   reg_name=ccprepreg
   registry=${reg_name}.azurecr.io
   reg_token_file=reg_token.json

   reg_token=$(jq -r .accessToken $reg_token_file)

   docker login $registry -u 00000000-0000-0000-0000-000000000000 -p $reg_token

その後、新しくビルドしたコンテナイメージを ACR にプッシュできます。

.. code-block:: bash

   #!/usr/bin/env bash

   tag=0.0.1
   name=cc_prep
   reg_name=ccprepreg
   registry=${reg_name}.azurecr.io

   docker push $registry/$name:$tag
   docker push $registry/skr:2.7

.. note::

   skr:2.7 は、Microsoft のオープンソースプロジェクト https://github.com/microsoft/confidential-sidecar-containers からビルドされています。
   skr イメージのビルド方法と、レジストリ名を付けたリネーム方法については、そのドキュメントを確認してください。


機密 ACI 起動スクリプトの作成と実行
------------------------------------------------------------

この手順には az cli の confcom 拡張機能が必要です。次のコマンドでインストールできます。

.. code-block:: bash

   az extension add --name confcom


以下は、コンテナイメージの認証情報および検証情報を、Azure Resource Manager (ARM) テンプレートファイル
(この例では cce_done_p.json)に適切に注入するためのスクリプトです。


.. code-block:: bash

   #!/usr/bin/env bash

   resource_group=cc-prep-rsr-grp
   base_file=ccprep.json
   reg_token_file=reg_token.json
   export registry_token=$(jq -r .accessToken $reg_token_file)

   tmp=$(jq . $base_file)
   tmp=$(echo $tmp | jq '.resources[0].properties.imageRegistryCredentials[0].password = env.registry_token')
   echo $tmp > tmp.json

   az confcom acipolicygen -a tmp.json --print-policy > cce_token.b64

   export cce_token=$(cat cce_token.b64)
   cce_done=$(echo $tmp | jq '.resources[0].properties.confidentialComputeProperties.ccePolicy = env.cce_token')

   echo $cce_done > cce_done_p.json

   az deployment group create --resource-group $resource_group --template-file cce_done_p.json

このスクリプトには、以下に示すベースとなる ARM テンプレートファイルが必要です。このサンプルファイルではトークンや検証情報はすべて削除されています。ただし、
お使いの機密 ACI と NVFlare サーバーの構成で使用する情報に合わせて編集する必要があります。ポート番号、ロケーション、コンテナイメージ名、および
機密 ACI のリソースは、ご自身のアカウントとこれまでの設定に応じて更新しなければならない点にご注意ください。

権限に関する問題が発生した場合やスクリプトが停止してしまう場合は、ACR におけるご自身のロールを確認してください。

.. literalinclude:: ../../../resources/ccprep.json
   :language: json


このスクリプトの完了には数分かかります。完了後、Azure の Container Instances ページで機密 ACI が起動して稼働していることを確認できます。
