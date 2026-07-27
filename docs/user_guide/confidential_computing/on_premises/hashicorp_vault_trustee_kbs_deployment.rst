.. _hashicorp_vault_trustee_deployment:

#############################################################
HashiCorp Vault と Trustee KBS の統合デプロイガイド
#############################################################

概要
========

本ガイドでは、Confidential Computing環境向けの統合されたシークレット管理システムとして、HashiCorp Vault と Trustee KBS (Key Broker Service) をデプロイするための完全な手順を説明します。

**アーキテクチャ:**

- **HashiCorp Vault** : シークレットを保存するための安全なバックエンド
- **Trustee KBS** : クライアントの身元を検証し、キーを仲介するフロントエンドプロキシ
- **デプロイ順序** : 先にVaultをデプロイし、その後にKBSをデプロイする必要があります

**本ガイドで学べること:**

- デプロイのアーキテクチャと要件の理解
- 適切なTLS設定を伴うHashiCorp Vaultのセットアップ
- Trustee KBSのコンパイルと設定
- クライアント操作によるシステム全体のテスト
- よくある問題のトラブルシューティング

.. note::

   **TEE環境のデプロイ要件**

   デプロイを開始する前に、デプロイアーキテクチャを適切に計画できるよう、各コンポーネントのハードウェア環境要件を理解してください。

アーキテクチャの理解
=========================================

デプロイアーキテクチャ
------------------------------------

::

   ┌─────────────────┐    ┌──────────────────┐    ┌──────────────────┐
   │   TEE Client    │───▶│   Trustee KBS    │───▶│ HashiCorp Vault  │
   │ (TEE Hardware)  │    │                  │    │                  │
   └─────────────────┘    └──────────────────┘    └──────────────────┘
   │                 │    │                  │    │                  │
   │ Hardware:       │    │ Functions:       │    │ Functions:       │
   │ • Intel TDX     │    │ • Attestation    │    │ • Secret         │
   │ • AMD SEV       │    │   Verification   │    │   Storage        │
   │ • ARM TrustZone │    │ • Policy Engine  │    │ • Access         │
   │ • TPM 2.0       │    │ • Key Broker     │    │   Control        │
   │                 │    │ • JWT Auth       │    │ • Audit Logs     │
   │                 │    │                  │    │ • Encrypted      │
   │                 │    │                  │    │   Transport      │
   └─────────────────┘    └──────────────────┘    └──────────────────┘

環境の種類
--------------------

**テスト環境** (本ガイドで扱う対象):

- VaultとKBSは通常のサーバー上にデプロイされます
- クライアントは "sample attester" を使用してTEEのエビデンスをシミュレートします
- 適した用途: 機能検証、開発時のデバッグ、システム結合テスト

**本番環境** :

- VaultとKBSは引き続き安全な環境(データセンター)にデプロイされます
- クライアントは **必ず** 実際のTEEハードウェア上で動作させる必要があります
- クライアントは実際のハードウェアに基づくアテステーションエビデンスを生成します

デプロイのフェーズ
=========================

このデプロイは4つのフェーズで構成されます。

1. **環境の準備** - 必要なツールと依存関係のインストール
2. **HashiCorp Vaultのデプロイ** - 安全なバックエンドストレージのセットアップ
3. **Trustee KBSのデプロイ** - アテステーションとキーブローカーのサービスのセットアップ
4. **クライアント操作** - システム全体のテストと検証

フェーズ1: 環境の準備
=================================

システム要件
--------------------

**オペレーティングシステム:**

- Ubuntu 22.04 または 24.04 (推奨)
- Debian系のディストリビューション

**必要なツール:**

- Git
- Curl
- OpenSSL
- ビルドツール (gcc、clang)
- Protobufコンパイラ
- Rust (KBSのコンパイル用)

インストール手順
------------------------

**1.1 システムの更新**

.. code-block:: bash

   sudo apt-get update
   sudo apt-get upgrade -y

**1.2 基本ツールのインストール**

.. code-block:: bash

   sudo apt-get install -y git curl build-essential clang libtss2-dev openssl pkg-config protobuf-compiler

**1.3 Rustのインストール**

Trustee KBSのコンパイルにはRustが必要です。

.. code-block:: bash

   curl https://sh.rustup.rs -sSf | sh
   source "$HOME/.cargo/env"

インストール中は、デフォルトのオプション (1) を選択してください。

フェーズ2: HashiCorp Vaultのデプロイ
====================================================

Vaultはシークレットを保存するための安全なバックエンドとして機能します。ここではTLS暗号化と適切なアクセス制御を設定します。

Vaultのインストール
--------------------------

.. code-block:: bash

   wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
   echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
   sudo apt update && sudo apt install vault

Vaultの証明書ディレクトリとデータディレクトリの作成
------------------------------------------------------------------

.. code-block:: bash

   sudo mkdir -p /opt/vault/tls
   sudo mkdir -p /opt/vault/data

自己署名TLS証明書の生成 (テスト用)
------------------------------------------------

以下のコマンドを実行して、Vaultが必要とする vaultlocal.key ファイルと vaultlocal.crt ファイルを生成します。

.. code-block:: bash

   sudo openssl req -x509 -newkey rsa:4096 -keyout /opt/vault/tls/vaultlocal.key -out /opt/vault/tls/vaultlocal.crt -sha256 -days 365 -nodes -subj "/CN=localhost"

.. note::
   本番環境では、信頼されたCAによって発行された証明書を使用してください。このコマンドで生成される自己署名証明書はテスト目的専用です。

Vaultの設定 (/etc/vault.d/vault.hcl)
------------------------------------------------

`sudo nano /etc/vault.d/vault.hcl` で設定ファイルを編集し、以下の内容に置き換えてください。

.. code-block::

    {
      "ui": true,
      "api_addr": "https://<your-server-IP-or-hostname>:8200",  // Example URL
      "storage": {
        "file": {
          "path": "/opt/vault/data"
        }
      },
      "listener": {
        "tcp": {
          "address": "<your-server-IP-or-hostname>:8200",  // Example address
          "tls_cert_file": "/opt/vault/tls/vaultlocal.crt",
          "tls_key_file": "/opt/vault/tls/vaultlocal.key"
        }
      }
    }

CA署名済みサーバー証明書の使用 (厳格な検証を行う場合。推奨)
------------------------------------------------------------------------------

クライアント側 (KBSなど) で厳格なTLS検証を有効にする必要がある場合、CA証明書をそのままサーバー証明書として使用してはいけません。以下の手順に従って、ローカルCAによって署名された「サーバー証明書」(SANを含み、CA:FALSEであり、EKUに serverAuth を含む必要があります) を生成し、そのサーバー証明書をVaultで使用してください。

ローカルCAの生成 (最初の一度だけ必要)
--------------------------------------------------

.. code-block:: bash

   sudo openssl genrsa -out /opt/vault/tls/ca.key 4096
   sudo openssl req -x509 -new -key /opt/vault/tls/ca.key -sha256 -days 3650 \
     -subj "/CN=Local Test CA" \
     -addext "basicConstraints=critical,CA:true,pathlen:0" \
     -addext "keyUsage=critical,keyCertSign,cRLSign" \
     -out /opt/vault/tls/ca.crt

サーバー証明書の生成 (SAN付き、CA:FALSE + serverAuth)
------------------------------------------------------------------

.. code-block:: bash

   # Server private key
   sudo openssl genrsa -out /opt/vault/tls/vault.key 2048

   # Server CSR (non-interactive)
   sudo openssl req -new -key /opt/vault/tls/vault.key -subj "/CN=localhost" -out /opt/vault/tls/vault.csr

   # Write SAN configuration (replace IP/DNS with your actual address)
   sudo tee /opt/vault/tls/san.cnf >/dev/null <<'EOF'

   basicConstraints=CA:false
   keyUsage=critical,digitalSignature,keyEncipherment
   extendedKeyUsage=serverAuth
   subjectAltName=DNS:localhost,IP:127.0.0.1,IP:10.176.193.230
   EOF

CAによるサーバー証明書への署名 (注意: 前の手順で生成した ca.crt / ca.key を使用します)
--------------------------------------------------------------------------------------------------------

.. code-block:: bash

   sudo openssl x509 -req -in /opt/vault/tls/vault.csr \
   -CA /opt/vault/tls/ca.crt -CAkey /opt/vault/tls/ca.key -CAcreateserial \
   -out /opt/vault/tls/vault.crt -days 825 -sha256 -extfile /opt/vault/tls/san.cnf

証明書の主要な拡張の簡易検証 (CA:FALSE、serverAuth、SANの一覧が表示されるはずです)
------------------------------------------------------------------------------------------------------

.. code-block:: bash

   sudo openssl x509 -in /opt/vault/tls/vault.crt -noout -text \
   | sed -n '/Subject:/p;/Subject Alternative Name/,+1p;/Extended Key Usage/,+1p;/Basic Constraints/,+1p'

Vaultの証明書ファイルのパーミッションと所有者の修正 (Vaultは vault ユーザーとして動作します)
------------------------------------------------------------------------------------------------------------------

.. code-block:: bash

   # Directory and file ownership
   sudo chown -R vault:vault /opt/vault/tls
   # Directory and file permissions (directory traversable; private key readable only by owner; certificates readable)
   sudo chmod 750 /opt/vault/tls
   sudo chmod 640 /opt/vault/tls/vault.key
   sudo chmod 644 /opt/vault/tls/vault.crt /opt/vault/tls/ca.crt
   # If needed, ensure parent directories are traversable
   sudo chmod 755 /opt /opt/vault

Vault設定の更新と再起動
------------------------------------

/etc/vault.d/vault.hcl 内の証明書のパスを、新しいサーバー証明書に向けます。

.. code-block::

   tls_cert_file=/opt/vault/tls/vault.crt
   tls_key_file=/opt/vault/tls/vault.key

その後、再起動して状態を確認します。

.. code-block:: bash

   sudo systemctl restart vault
   sudo systemctl status vault | cat
   # Verify HTTPS:
   curl --cacert /opt/vault/tls/ca.crt https://<your-server-IP-or-hostname>:8200/v1/sys/health | cat

Vaultサービスの起動
--------------------------

.. code-block:: bash

   sudo systemctl restart vault
   sudo systemctl enable vault # Set to start on boot

Vaultのデプロイ成功の確認
----------------------------------------

次に進む前に、以下の方法でVaultサービスが正常に動作していることを確認してください。

方法1: サービスの状態を確認する

.. code-block:: bash

   sudo systemctl status vault

成功していれば、緑色の "active (running)" という文字が表示されます。

方法2: ネットワークポートを確認する

.. code-block:: bash

   sudo netstat -tuln | grep 8200

成功していれば、システムがポート8200で待ち受けていることが確認できます。

方法3: Web UIにアクセスする (最も直感的)

ブラウザで https://:8200 にアクセスします。Vaultの初期化画面またはログイン画面が表示されれば、デプロイは完全に成功しています。

Vault UIでの初期化と設定
----------------------------------------

a. 初期化: 初めてUIにアクセスすると、初期化画面が表示されます。これはVaultのセキュリティ機構の中核であり、マスターキーを生成するために使用されます。

- **Key shares** : マスターキーが分割される断片の総数です。
- **Key threshold** : Vaultを毎回 "unseal" するために必要なキー断片の最小数です。

本ガイドのテスト環境では、以下の最も単純な設定を使用します。

- **Key shares** : 1
- **Key threshold** : 1
- **Store PGP keys** : チェックを入れないままにします。

"Initialize" ボタンをクリックすると、システムはRoot TokenとRecovery Keyを生成します。これら2つの値は必ず安全にコピーして保存してください。

b. ログイン: 初期化完了後のページで、先ほど保存したRoot Tokenを使用してログインします。

c. KVエンジンの有効化:

- 左側のメニューから "Secrets Engines" を選択します。
- "Enable new engine +" をクリックします。
- "KV" を選択します。
- 設定ページで以下を指定します。

  - **Path** : kv と入力します (これは後続のKBS設定における mount_path と一致している必要があります)。
  - **Version** : 1 を選択します (KBSは現在V1バージョンのみをサポートしています)。

- "Enable Engine" をクリックします。

.. important::
   以前にUIでKV v2を有効化している場合は、以下の手順に従ってv1に変更してください (Web上での操作)。

   - 左側の "Secrets Engines" の一覧を開き、マウントパスが kv のエントリを見つけ、右側の "⋯" メニューをクリックして "Disable" を選択し、確定します。
   - "Enable new engine +" をクリックし、 "KV" を選択します。設定ページでPathには kv を入力し、Versionには1を選択して、 "Enable" をクリックします。
   - エンジンのページに入ると、右上に "Version: 1" と表示されているはずです。まだv2のままであれば、上記の手順を繰り返してください。

フェーズ3: Trustee KBS (Key Broker Service) のデプロイ
================================================================

Vaultの準備が整ったら、クライアントとVaultをつなぐ中核のプロキシとしてKBSをデプロイします。必要に応じて、
Dockerイメージをビルドして直接実行することもできます。Dockerイメージをビルドする場合は、付録に従ってください。

コードのクローンと特定バージョンのチェックアウト
------------------------------------------------------------------

.. code-block:: bash

   git clone https://github.com/confidential-containers/trustee.git
   cd trustee/kbs
   git checkout a2570329cc33daf9ca16370a1948b5379bb17fbe

KBSのコンパイル (重要!)
--------------------------------

KBSがVaultと通信できるようにするには、コンパイル時に vault フィーチャーを有効にする必要があります。

KBSサービスのコンパイルとインストール

.. code-block:: bash

   sudo cargo install --path . --features="vault"

KBSクライアントツールのコンパイル (非TEE環境でのテストに対応)

.. note::
   非TEE環境では、sample attesterをサポートするために sample_only フィーチャーを有効にする必要があります

.. code-block:: bash

   make cli CLI_FEATURES=sample_only
   sudo make install-cli

トラブルシューティング: コンパイルエラーと実行時エラーの修正
------------------------------------------------------------------------------

問題1: コンパイルエラー "error[E0277]: can't compare"

これは、kbsの依存ライブラリである verifier の内部コードにおける型の不一致が原因です。この依存ライブラリのソースファイルを手動で修正することで解決する必要があります。

a. ファイルの特定: trustee ディレクトリ内で、deps/verifier/src/az_snp_vtpm/mod.rs というファイルを見つけて開きます。

b. コードの修正: 225行目付近の、次のようなコードを見つけます。

.. code-block:: rust

   // Original code
   && get_oid_octets::<64>(&parsed_endorsement_key, HW_ID_OID)? != report.chip_id

コンパイラのヒントに従い、report.chip_id の前にデリファレンス用のアスタリスク * を追加し、以下のように修正します。

.. code-block:: rust

   // Modified code
   && get_oid_octets::<64>(&parsed_endorsement_key, HW_ID_OID)? != *report.chip_id

c. ファイルの保存と再コンパイル: ファイルの修正を保存したら、trustee/kbs ディレクトリに戻り、コンパイルコマンドを再実行します。

.. code-block:: bash

   sudo cargo install --path . --features="vault"

問題2: 再コンパイル後もKBSの起動時に "unknown variant 'Vault'" というエラーが出る

原因: これは通常、cargoでインストールした新しいバージョンではなく、システム上の古いバージョンのkbsプログラムが実行されていることを意味します。

診断と解決策:

a. 現在のユーザーにおけるkbsの正しいパスを確認します。

.. code-block:: bash

   which kbs

このコマンドは、新しくコンパイルされたkbsの絶対パス (例: /home/user/.cargo/bin/kbs) を表示します。

b. 絶対パスで起動する (推奨): sudo kbs ... を直接実行するのではなく、前の手順で得た絶対パスを使って新しいプログラムを起動します。

以下のパスは、前の手順で得た実際のパスに置き換えてください。

.. code-block:: bash

   sudo /home/user/.cargo/bin/kbs --config-file ./kbs-config.toml

c. 恒久的な修正 (任意): 今後 sudo kbs ... を直接使えるようにしたい場合は、シンボリックリンクを作成できます。

以下のリンク元パスは、手順aで見つけた実際のパスに置き換えてください。

.. code-block:: bash

   sudo ln -sf /home/user/.cargo/bin/kbs /usr/local/bin/kbs

KBSが必要とする各種キーファイルの生成 (新規)
----------------------------------------------------------

KBSを起動する前に、KBS用のHTTPS証明書と管理者認証キーを生成する必要があります。trustee/kbs ディレクトリ内で実行してください。

キーを格納するディレクトリを作成します。

.. code-block:: bash

   mkdir -p keys wkdir admin

1. KBSのHTTPS証明書構成の生成 (CA署名モードを推奨)

1.1) KBSローカルCAの生成 (サーバー証明書への署名用)

.. code-block:: bash

   openssl genrsa -out keys/kbs-ca.key 4096
   openssl req -x509 -new -key keys/kbs-ca.key -sha256 -days 3650 \
   -subj "/CN=KBS Local CA" \
   -addext "basicConstraints=critical,CA:true,pathlen:0" \
   -addext "keyUsage=critical,keyCertSign,cRLSign" \
   -out keys/kbs-ca.crt

1.2) KBSサーバー証明書要求の生成

.. code-block:: bash

   openssl genrsa -out keys/key.pem 2048
   openssl req -new -key keys/key.pem -subj "/CN=localhost" -out keys/kbs.csr

1.3) サーバー証明書の拡張設定の作成

.. code-block:: bash

   tee keys/kbs-san.cnf >/dev/null <<'EOF'
   basicConstraints=CA:false
   keyUsage=critical,digitalSignature,keyEncipherment
   extendedKeyUsage=serverAuth
   subjectAltName=DNS:localhost,IP:127.0.0.1
   EOF

1.4) KBSのCAによるサーバー証明書への署名

.. code-block:: bash

   openssl x509 -req -in keys/kbs.csr \
   -CA keys/kbs-ca.crt -CAkey keys/kbs-ca.key -CAcreateserial \
   -out keys/cert.pem -days 825 -sha256 -extfile keys/kbs-san.cnf

1.5) 生成された証明書の検証

.. code-block:: bash

   openssl x509 -in keys/cert.pem -noout -text | \
   sed -n '/Subject:/p;/Subject Alternative Name/,+1p;/Extended Key Usage/,+1p;/Basic Constraints/,+1p'

1.6) クライアント側の信頼設定 (非常に重要)

kbs-ca.crt (手順1.1で生成) は、KBSサーバー証明書に署名するCAルートです。
クライアントがHTTPSでKBSに接続するには、このCAを **必ず** 信頼する必要があります。

オプションA: kbs-client に明示的に渡す

.. code-block:: bash

   --cert-file ./keys/kbs-ca.crt

オプションB (サービス向けに推奨): システムのCAストアにインストールする (Ubuntu/Debian)

.. code-block:: bash

   sudo cp ./keys/kbs-ca.crt /usr/local/share/ca-certificates/kbs-ca.crt
   sudo update-ca-certificates

オプションC (コンテナ): ファイルをマウントし、環境変数 SSL_CERT_FILE=/etc/ssl/certs/kbs-ca.crt を設定する

2. 管理者認証用キーペアの生成 (Ed25519)

.. note::
   KBSの管理APIは、JWT署名の検証にEd25519公開鍵のみを受け付けます

.. code-block:: bash

   openssl genpkey -algorithm Ed25519 -out admin/admin.key
   openssl pkey -in admin/admin.key -pubout -out admin/admin.pub

.. note::
   上記で生成したEd25519アルゴリズムのキーペアを使用してください。RSA公開鍵を使用すると、KBSが "Invalid public key" というエラーを報告します。

KBS設定ファイルの準備 (kbs-config.toml)
--------------------------------------------------------

kbs ディレクトリ内に kbs-config.toml という名前のファイルを作成し、以下の内容を記述します。

.. code-block::

   [http_server]
   sockets = ["0.0.0.0:8999"]
   insecure_http = false
   private_key = "./keys/key.pem"
   certificate = "./keys/cert.pem"

   [admin]
   auth_public_key = "./admin/admin.pub"
   ... (other attestation_service, policy_engine configurations remain unchanged) ...

   [attestation_token]
   insecure_key = true

   [attestation_service]
   type = "coco_as_builtin"
   work_dir = "./wkdir/attestation-service"
   policy_engine = "opa"

   [attestation_service.attestation_token_broker]
   type = "Ear"
   duration_min = 5

   [attestation_service.rvps_config]
   type = "BuiltIn"

   [attestation_service.rvps_config.storage]
   type = "LocalJson"

   [policy_engine]
   policy_path = "./wkdir/policy.rego"

   [[plugins]]
   name = "resource"
   type = "Vault"
   Fill in your deployed Vault address
   vault_url = "https://:8200"
   Fill in the root token you obtained during Vault initialization
   token = "hvs.xxxxnnnnxxxxnnnn"
   Must match the path configured in Vault
   mount_path = "kv"

.. note::
   このパスはKV v1エンジンとしてマウントされている必要があります。KBSは現在kv1 APIを使用しています

   Vaultが自己署名証明書を使用している場合は、これを false に設定します

   verify_ssl = false

   verify_ssl が true で自己署名証明書を使用している場合は、コメントを外してCA証明書のパスを指定します

   ca_certs = ["./wkdir/local-ca.pem"]

.. note::
   vault_url と token は、実際の情報に置き換えてください。

   "Permission denied" エラーが発生する場合は、 [attestation_service.rvps_config.storage] セクションに以下を追加してください。

   file_path = "./wkdir/attestation-service/reference_values.json"

KBSサービスの起動
--------------------------

正しいバージョンが実行されるようにするため、絶対パスで起動することを推奨します。

.. code-block:: bash

   sudo /home/user/.cargo/bin/kbs --config-file ./kbs-config.toml

ターミナルにエラーが表示されず、サービスがポート8999で待ち受けていることが表示されれば、KBSは正常に起動しています。

アテステーションポリシーの設定 (非TEE環境では必須)
------------------------------------------------------------------

非TEE環境でテストする場合は、sample attesterが検証を通過できるように、寛容なアテステーションポリシーを設定する必要があります。

方法1: ポリシーファイルを直接置き換える (推奨)

.. code-block:: bash

   cp ./sample_policies/allow_all.rego ./wkdir/policy.rego

方法2: 管理APIを介して設定する (任意)

.. code-block:: bash

   /path/to/target/release/kbs-client --url https://<trustee-service-host>:8999 \
   --cert-file ./keys/kbs-ca.crt \
   config --auth-private-key ./admin/admin.key \
   set-attestation-policy --policy-file ./sample_policies/allow_all.rego

.. note::
   本番環境では、実際のTEEエビデンスを検証するための厳格なアテステーションポリシーを使用してください。寛容なポリシーはテスト環境や開発環境にのみ適しています。

フェーズ4: クライアント操作と検証
=====================================================

これでシステム全体の準備が整いました。kbs-client を使用して、シークレットの保存と取得をテストできます。

.. note::
   コンパイル済みの kbs-client は trustee/target/release/kbs-client にあります。プロジェクトが /home/user/trustee ディレクトリにある場合、フルパスは /home/user/trustee/target/release/kbs-client となります。

シークレットの保存
------------------------

まず、テスト用のファイル (例: test.txt) を作成します。

.. code-block:: bash

   echo "this is a test file." > test.txt

以下のコマンドを実行して、ファイルの内容をVaultに保存します (管理者操作)。

.. code-block:: bash

   /path/to/target/release/kbs-client --url https://<trustee-service-host>:8999 \
   --cert-file ./keys/kbs-ca.crt \
   config --auth-private-key ./admin/admin.key \
   set-resource --path mysecrets/database/password \
   --resource-file test.txt

シークレットの取得 (リモートアテステーション操作)
------------------------------------------------------------------

まず、TEE秘密鍵を生成します (クライアントのシミュレーション用)。

.. code-block:: bash

   openssl ecparam -name prime256v1 -genkey -noout | \
   openssl pkcs8 -topk8 -nocrypt -out tee_ec.key

シークレットを取得します (クライアントが自動的にアテステーション処理を実行します)。

.. code-block:: bash

   /path/to/target/release/kbs-client --url https://<trustee-server-host>:8999 \
   --cert-file ./keys/kbs-ca.crt \
   get-resource --path mysecrets/database/password \
   --tee-key-file ./tee_ec.key

.. note::
   コンパイル済みの kbs-client を使用してください: /path/to/target/release/kbs-client (実際のパスに置き換えてください)

   非TEE環境では "Sample Attester will be used" という警告が表示されますが、これは正常です

   成功すると、コマンドはbase64エンコードされた内容を出力します。echo "result" | base64 -d でデコードしてください

おめでとうございます。HashiCorp Vault と Trustee KBS から成るシークレット管理システムのデプロイとテストに成功しました。

トラブルシューティング
=========================

問題1: get-resource が "illegal token format" というエラーで失敗する

症状: クライアントで get-resource を実行すると、次のエラーが報告されます。

.. code-block::

   Error: read token
   Caused by: illegal token format

根本原因: 非TEE環境において、kbs-client で sample_only フィーチャーが有効化されておらず、有効なアテステーショントークンを生成できません。

解決策:

sample_only フィーチャーを有効にして kbs-client を再コンパイルします。

.. code-block:: bash

   make -C trustee/kbs cli CLI_FEATURES=sample_only

新しくコンパイルしたクライアントを使用します。

.. code-block:: bash

   /path/to/trustee/target/release/kbs-client [other parameters...]

問題2: アテステーションが "Access denied by policy" というエラーで失敗する

症状: クライアントが次のエラーを報告します。

.. code-block::

   Error: request unauthorized
   ...ErrorInformation { error_type: "PolicyDeny", detail: "Access denied by policy" }

根本原因: KBSのデフォルトポリシーはsampleのエビデンスを拒否し、実際のTEEエビデンスのみを受け入れます。

解決策:

ポリシーファイルを寛容なポリシーに更新します。

.. code-block:: bash

   cp ./sample_policies/allow_all.rego ./wkdir/policy.rego

または、管理APIを介して設定します。

.. code-block:: bash

   kbs-client --url https://<trustee-service-host>:8999 \
     --cert-file ./keys/kbs-ca.crt \
     config --auth-private-key ./admin/admin.key \
     set-attestation-policy --policy-file ./sample_policies/allow_all.rego

問題3: VaultのTLS証明書エラー

症状: KBSの起動時に "CaUsedAsEndEntity" というエラーが報告される、またはVaultへの接続に失敗します。

根本原因: Vaultが規格に適合しない証明書を使用しています (CA証明書がサーバー証明書として使用されている)。

解決策: 本ドキュメントのフェーズ2の手順3を参照して、正しいサーバー証明書を生成してください。

問題4: KVエンジンのバージョンの不一致

症状: set-resource が "Invalid path for a versioned K/V secrets engine" というエラーを報告します。

根本原因: VaultがKV v2エンジンをマウントしているのに対し、KBSはkv1 APIを使用しています。

解決策: Vault UIで既存のKVエンジンを無効化し、v1バージョンとして有効化し直してください。

問題5: RVPSストレージのパーミッションエラー

症状: KBSの起動時に "Permission denied (os error 13)" というエラーが報告され、通常は /opt/confidential-containers/attestation-service/ というパスが関係しています。

根本原因: 組み込みのRVPSはLocalJsonストレージを使用しており、デフォルトでは一般ユーザーに書き込み権限のないシステムディレクトリに書き込もうとします。

解決策: kbs-config.toml の [attestation_service.rvps_config.storage] セクションに、書き込み可能なパスを追加します。

.. code-block:: toml

   [attestation_service.rvps_config.storage]
   type = "LocalJson"
   file_path = "./wkdir/attestation-service/reference_values.json"

問題6: テスト環境における正常な警告メッセージ

症状: 非TEE環境でテストすると、クライアントが以下の警告メッセージを出力します。

.. code-block::

   [WARN] No TEE platform detected. Sample Attester will be used.
   [WARN] Authenticating with KBS failed. Perform a new RCAR handshake: TokenNotFound

説明: これらはエラーではなく、正常な警告メッセージです。

- "No TEE platform detected":

  - 通常のサーバー上でテストする場合に想定される動作です
  - システムは自動的にsample attesterに切り替えて、TEEのエビデンスをシミュレートします
  - これはテスト環境でまさに期待される動作です

- "TokenNotFound" / "Perform a new RCAR handshake":

  - 初回アクセス時の正常な認証フローです
  - クライアントにキャッシュされたアテステーショントークンがありません
  - システムは自動的に新しいRCAR (Relying Party Attestation Capabilities and Resource) ハンドシェイクを実行します

正常に動作したことを確認する方法:

- 最終出力を確認します。base64エンコードされたシークレットの内容が表示されていれば、操作は成功しています
- echo "base64content" | base64 -d を使ってデコードし、内容が正しいことを検証します
- テスト環境では、これらの警告メッセージはまったく正常であり、想定されたものです

付録
========

KBSのDockerイメージのビルド
------------------------------------------


kbs/docker フォルダ内のDockerfileを基に、kbs用のDockerイメージをビルドできます。
ただし、コミットID a2570329cc33daf9ca16370a1948b5379bb17fbe 時点の現在の trustee リポジトリにあるそのファイルは、
ビルドに失敗するか、依存関係が欠落したDockerイメージを生成します。
以下のdiffでそのファイルにパッチを当てることができます。

.. code-block:: diff

   $ git diff
   diff --git a/kbs/docker/Dockerfile b/kbs/docker/Dockerfile
   index e529716..45b9271 100644
   --- a/kbs/docker/Dockerfile
   +++ b/kbs/docker/Dockerfile
   @@ -39,17 +39,17 @@ RUN if [ "${ARCH}" = "x86_64" ]; then curl -fsSL https://download.01.org/intel-s
   WORKDIR /usr/src/trustee
   COPY . .

   -RUN cd kbs && make AS_FEATURE=coco-as-builtin ALIYUN=${ALIYUN} ARCH=${ARCH} && \
   +RUN cd kbs && make VAULT=true AS_FEATURE=coco-as-builtin ALIYUN=${ALIYUN} ARCH=${ARCH} background-check-kbs && \
      make ARCH=${ARCH} install-kbs

   -FROM ubuntu:22.04
   +FROM ubuntu:24.04
   ARG ARCH=x86_64

   WORKDIR /tmp

   RUN apt-get update && \
      apt-get install -y \
   -    curl \
   +    curl gpg \
      gnupg-agent && \
      if [ "${ARCH}" = "x86_64" ]; then curl -fsSL https://download.01.org/intel-sgx/sgx_repo/ubuntu/intel-sgx-deb.key | \
      gpg --dearmor --output /usr/share/keyrings/intel-sgx.gpg && \

kbsのDockerイメージをビルドするには、trustee フォルダ内で以下を実行します。

.. code-block:: bash

   docker build -f kbs/docker/Dockerfile .

-p オプションでポートを公開して、Dockerコンテナ内でKBSを実行できます。例:

.. code-block:: bash

   docker run -p 8080:8080 <image_name>
