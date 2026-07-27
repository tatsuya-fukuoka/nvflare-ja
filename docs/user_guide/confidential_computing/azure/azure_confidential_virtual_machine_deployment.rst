.. _azure_confidential_virtual_machine_deployment:

#######################################################
Azure 機密仮想マシン (CVM) デプロイガイド
#######################################################

概要
========

Azure の機密仮想マシン (confidential virtual machine, CVM) の作成は、通常の VM の作成とよく似ています。本ガイドでは Azure CLI (az) を使用して 1 台の Azure
CVM を作成し、アテステーション操作を実行して検証します。その後、NVFlare をインストールしてスタートアップキットを転送し、CVM 内で NVFlare を起動できます。

.. note::

   Azure CVM の起動には、Azure アカウントに特定の権限が必要になる場合があります。詳細については、ご自身の Azure アカウントおよび Azure にご確認ください。


Azure CVM を起動する手順
===============================================================

* ログインして 1 つのリソースグループを作成する
* Azure CVM を作成する
* アテステーションレポートを取得する

ログインしてリソースグループを作成する
--------------------------------------------

まず、az cli を使って Azure にログインする必要があります。その後、以降の操作で生成されるすべてのリソースを格納する
リソースグループを 1 つ作成できます。

リソースグループには別の名前や別のロケーションを選ぶこともできます。

.. code-block:: bash

   #!/usr/bin/env bash

   resource_group=cc-cvm-rg
   location=northeurope

   az login

   az group create --name $resource_group --location $location


Azure CVM を作成する
-----------------------------------------

リソースグループを作成したら、そのまま CVM の作成に進むことができます。

.. code-block:: bash

   #!/usr/bin/env bash

   resource_group=cc-cvm-rg
   cvm_name=cc_prep_cvm
   cvm_size=Standard_DC4as_v5
   user_name=azureuser
   user_password=<YOUR_OWN_PASSWORD>
   image_name=Canonical:0001-com-ubuntu-confidential-vm-jammy:22_04-lts-cvm:latest

   az vm create --resource-group $resource_group \
      --name $cvm_name \
      --size $cvm_size \
      --admin-username $user_name \
      --admin-password $user_password \
      --enable-vtpm true \
      --image $image_name \
      --public-ip-sku Standard --security-type ConfidentialVM \
      --os-disk-security-encryption-type VMGuestStateOnly \
      --enable-secure-boot true

この cvm_size は AMD SEV-SNP に基づいています。そのため、次のステップで取得するアテステーショントークンには snp 関連のフィールドが含まれます。
user_password はご自身のパスワードに変更するのを忘れないでください。ご利用のサブスクリプションには、より高いセキュリティを確保するためのポリシーが設定されている場合があります。
コンプライアンスのために、すべてのプロパティ、ネットワークセキュリティ、権限を確認してください。

アテステーションレポートの取得
--------------------------------------

上記の CVM のパブリック IPv4 アドレスが確認できます。上記スクリプトで定義した資格情報を使ってログインしてください。

CVM 内でアテステーションレポートを取得するには、まず環境を準備する必要があります。以下のコマンドを実行して、
必要なツールをインストールし、アテステーションを実行するためのソースコードをダウンロードします。

.. code-block:: bash

   #!/usr/bin/env bash

   sudo apt-get update && \
      sudo apt-get install -y build-essential cmake unzip jq \
      libcurl4-openssl-dev libjsoncpp-dev libboost-all-dev nlohmann-json3-dev

   wget https://packages.microsoft.com/repos/azurecore/pool/main/a/azguestattestation1/azguestattestation1_1.1.2_amd64.deb
   sudo dpkg -i azguestattestation1_1.1.2_amd64.deb

   wget https://github.com/Azure/confidential-computing-cvm-guest-attestation/archive/refs/heads/main.zip
   unzip main.zip

   pushd confidential-computing-cvm-guest-attestation-main/cvm-attestation-sample-app
   cmake . && make

   sudo install -D -m0755 AttestationClient /usr/local/bin

   popd

これでアテステーションツールがビルドされ、インストールされました。アテステーショントークンを取得して内容を確認できます。


.. code-block:: bash

   #!/usr/bin/env bash

   sudo AttestationClient -o token > token.b64
   jwt=$(cat token.b64)
   echo "Showing attestation token in base64-encoded format"
   echo $jwt

   echo "Showing the header of attestation token"
   echo -n $jwt | cut -d "." -f 1 | base64 -d 2>/dev/null | jq .

   echo "Showing the payload of attestation token"
   echo -n $jwt | cut -d "." -f 2 | base64 -d 2>/dev/null | jq .


次のステップ
============

これで、NVFlare をインストールし、スタートアップキットをこの CVM インスタンスに転送して NVFlare を起動できます。

以下は cc_site-1.yml ファイルのサンプルで、cc のプロビジョニングにおいて project.yml とともに使用します。project.yml のサンプルも
以下に示します。なお、この project.yml にはサーバーの cc 設定 yaml ファイルが含まれており、これについては
:ref:`confidential_azure_container_instances_deployment` - Secure Aggregation on FLARE Server with Azure ACI (Azure Container Instance) で説明しています。

AZCVMAuthorizer は、デフォルトの Microsoft Azure Attestation エンドポイントとして sharedeus2.eus2.attest.azure.net を使用します。

.. code-block:: yaml

  compute_env: azure_cvm
  cc_cpu_mechanism: amd_sev_snp
  role: client
  cc_issuers:
    - id: az_cvm_authorizer
      path: nvflare.app_opt.confidential_computing.az_cvm_authorizer.AZCVMAuthorizer
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
