.. _package_command:

##########################
パッケージコマンド
##########################

``nvflare package`` は、プロジェクト管理者から返却された署名済み zip と、要求元のローカル秘密鍵から
スタートアップキットを組み立てます。

分散プロビジョニングにおける公開された形式は次のとおりです。

.. code-block:: none

   usage: nvflare package [-h] [-w WORKSPACE] [--request-dir REQUEST_DIR]
                          [--fingerprint EXPECTED_FINGERPRINT]
                          [--force] [--schema]
                          input

位置引数 ``input`` は、 ``nvflare cert approve`` が生成した ``*.signed.zip`` ファイルです。

******************************
基本的なパッケージ化の流れ
******************************

``./hospital-a`` に作成され、既定の ``--out`` （リクエスト zip の隣に署名済み zip を書き出します）で
承認されたサイトリクエストの場合は次のようにします。

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip

この場合でも、署名済み zip、署名済みメタデータ、証明書チェーン、ローカル秘密鍵の一致は検証されます。
ただし、帯域外でのルート CA フィンガープリントの比較は行われません。

信頼できる帯域外チャネルを通じてプロジェクト管理者から受け取った値と、署名済み zip のルート CA を
照合するには、期待するフィンガープリントを渡します。

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>

より長い綴りの ``--expected-fingerprint`` も受け付けられます。

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip \
       --expected-fingerprint <rootca_fingerprint_sha256>

署名済み zip がリクエストフォルダの隣にない場合（たとえば、リモート転送を経て要求元に戻された場合など）は、
ローカルのリクエストフォルダを明示的に指定します。

.. code-block:: shell

   nvflare package hospital-a.signed.zip \
       --request-dir ./hospital-a \
       --fingerprint <rootca_fingerprint_sha256>

このコマンドは次の点を検証します。

- 署名済み zip に ``signed.json`` 、 ``signed.json.sig`` 、 ``site.yaml`` 、
  1 つの署名済み証明書、および ``rootCA.pem`` が含まれていること
- 署名済みのエンドポイント、スキーム、接続セキュリティ、 ``ca_info`` の各フィールドを信頼する前に、
  ``signed.json.sig`` が ``rootCA.pem`` に対して検証されること
- 署名済み zip に秘密鍵が含まれていないこと
- ローカルの秘密鍵が署名済み証明書と一致すること
- 証明書が ``rootCA.pem`` までチェーンしていること
- 署名済みの CA フィンガープリントメタデータが、署名済み zip 内の ``rootCA.pem`` と一致すること
- ローカルの ``request.json`` のメタデータと署名済みメタデータが一致すること
- ローカルのリクエストフォルダにある ``site.yaml`` の識別子フィールドが署名済み zip と一致すること

このコマンドは、結果に常に ``rootca_fingerprint_sha256`` を出力します。
``--fingerprint <rootca_fingerprint_sha256>`` を指定しない場合、パッケージ化では帯域外の信頼比較は行われません。

署名済みの ``ca_info`` チェックは、パッケージワークスペース内での意図しない CA の混在を防ぎますが、
署名済み zip はそれ自体の ``rootCA.pem`` を含んでいるため、帯域外でのフィンガープリント検証の代わりにはなりません。
署名済み CA メタデータを含まない古い署名済み zip は、デプロイバージョン ``00`` として扱われ、
ワークスペースの一貫性チェックには同梱の ``rootCA.pem`` から計算したフィンガープリントが使用されます。

出力先は次の場所になります。

.. code-block:: text

   <workspace>/<project-name>/prod_<NN>/<identity>/

例を示します。

.. code-block:: text

   workspace/hospital_federation/prod_00/hospital-a/

デプロイバージョンは ``signed.json`` 内の署名済み CA メタデータから決まります。この値は
``nvflare cert init --deploy-version`` で設定され、既定値は ``00`` です。通常は気にする必要はありません。
同じ CA とデプロイバージョンで承認された複数の参加者は、同じ ``prod_00`` ディレクトリ内に並べて
パッケージ化されます。パッケージ化では、参加者ごとにディレクトリのカウンターが増えることはありません。

``prod_<NN>`` が既に存在する場合、 ``nvflare package`` は既存のパッケージルートが同じ ``rootCA.pem``
フィンガープリントを使用しているかを検証します。ルート CA の不一致は致命的なエラーです。デプロイバージョン
``00`` は ``prod_00`` に、デプロイバージョン ``01`` は ``prod_01`` に対応します。 ``--force`` は、
同一のデプロイバージョンおよび CA の下にある既存の参加者を置き換える場合にのみ使用してください。

********************
接続設定
********************

分散プロビジョニングのフローでは、package コマンドにエンドポイント引数を指定する必要はありません。

接続に関する値は次の情報源から解決されます。

- 署名済み zip 内の ``signed.json`` 。これにはプロジェクト管理者が承認した ``scheme`` 、既定の
  ``connection_security`` 、 ``project_profile.yaml`` 由来の ``server`` エンドポイント、および
  ``ca.json`` 由来の署名済み ``ca_info`` が含まれます
- リクエストフォルダ内にある元のローカル参加者定義。これには参加者の識別情報と、パッケージ化時の
  フィールドが含まれます

サーバーのホストおよびポートのフィールドは、署名済み承認メタデータの一部です。
``nvflare package`` は、スタートアップキットの生成に署名済みの ``server`` エンドポイントを使用します。
クライアントおよびユーザーのリクエストフォルダは、ローカルなエンドポイントの上書きを提供しません。
承認後にサーバーのエンドポイントが変更された場合は、 ``project_profile.yaml`` を更新し、影響を受ける
署名済み zip を再生成してください。

カスタムビルダーやサーバー側の ``connection_security`` の上書きなど、意図的に署名済み zip から
除外されているパッケージ化時のローカルフィールドは、引き続きローカルなパッケージ化の入力として残ります。

クライアントおよびユーザーの参加者定義には、サーバーのエンドポイントは含まれません。

.. code-block:: yaml

   participants:
     - name: hospital-a
       type: client
       org: hospital_alpha

ユーザーのスタートアップキットについても、 ``signed.json`` 内の同じ署名済みエンドポイントが使用されます。

.. code-block:: yaml

   participants:
     - name: alice@hospital-alpha.org
       type: admin
       org: hospital_alpha
       role: lead

サーバーキットの場合は、サーバー参加者の定義で ``connection_security`` を設定できます。

.. code-block:: yaml

   participants:
     - name: server1.hospital-central.org
       type: server
       org: hospital_central
       connection_security: mtls

このサーバー側の値は、パッケージ化時のローカルな上書きです。サーバーキットのビルド時にリクエストフォルダから
読み込まれます。これはプロジェクト管理者によって承認されるものではなく、フェデレーションのポリシーとして
配布されることもありません。設定されていない場合、package は署名済み zip 内の既定の
``connection_security`` を使用します。

****************************************
ユーザースタートアップキットのパッケージ化
****************************************

lead ユーザーの場合は次のようにします。

.. code-block:: shell

   nvflare cert request --participant alice.yaml
   nvflare cert approve alice@hospital-alpha.org/alice@hospital-alpha.org.request.zip --ca-dir ./ca --profile project_profile.yaml
   nvflare package alice@hospital-alpha.org/alice@hospital-alpha.org.signed.zip --fingerprint <rootca_fingerprint_sha256>

生成されるスタートアップキットには次のファイルが含まれます。

.. code-block:: text

   startup/fl_admin.sh

次のコマンドで実行します。

.. code-block:: shell

   cd workspace/hospital_federation/prod_00/alice@hospital-alpha.org
   ./startup/fl_admin.sh

****************
主な引数
****************

- ``input``: ``nvflare cert approve`` が返した承認済みの ``*.signed.zip`` です。
- ``-w, --workspace``: ワークスペースのルートディレクトリです。既定値: ``workspace`` 。
- ``--request-dir``: 秘密鍵、 ``request.json`` 、およびローカル参加者定義の全体を含む、ローカルの
  リクエストディレクトリです。署名済み zip がリクエストフォルダの隣になく、ローカルのリクエスト状態を
  利用できない場合に使用します。
- ``--fingerprint``: 署名済み zip 内の ``rootCA.pem`` に期待される SHA256 フィンガープリントです。
  一致しない場合、コマンドは失敗します。
- ``--expected-fingerprint``: ``--fingerprint`` のより長い綴りです。
- ``--force``: 署名済み CA 情報が引き続き一致している場合に、同じ ``prod_<NN>`` ディレクトリ配下の
  既存の参加者パッケージを置き換えることを許可します。ルート CA の不一致チェックを回避するものでは
  ありません。
- ``--schema``: このコマンドの JSON スキーマを出力します。

***************************
JSON 出力とヘルプ
***************************

機械可読なコマンド探索には ``--schema`` を使用します。

.. code-block:: shell

   nvflare package --schema

トップレベルの CLI は JSON 出力モードもサポートしています。

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip \
       --fingerprint <rootca_fingerprint_sha256> \
       --format json

分散プロビジョニングのエンドツーエンドのワークフローについては、 :ref:`distributed_provisioning` を参照してください。
