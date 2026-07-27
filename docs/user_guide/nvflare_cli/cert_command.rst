.. _cert_command:

################
Cert コマンド
################

``nvflare cert`` コマンドファミリーは、分散プロビジョニングにおける証明書関連の資材を管理します。
リクエスターと Project Admin の双方が使用します。

公開されているサブコマンドは次のとおりです。

.. code-block:: none

   usage: nvflare cert [-h] {init,request,approve} ...

   positional arguments:
     {init,request,approve}
       init                Initialize root CA for a distributed provisioning
                           federation (Project Admin only).
       request             Create a distributed provisioning request zip.
       approve             Approve a distributed provisioning request zip.

**********************
ルート CA の初期化
**********************

Project Admin は、まず **プロジェクトプロファイル** ファイルを作成します。このファイルは
Project Admin のマシン上のどこに置いても構いません。決められた配置場所はありません。
そのパスは ``cert init`` と ``cert approve`` のすべてのコマンドに明示的に渡されます。
NVFlare が自動的にこのファイルを探すことはありません。

.. code-block:: yaml

   name: hospital_federation
   scheme: grpc
   connection_security: tls
   server:
     host: server1.hospital-central.org
     fed_learn_port: 8002
     admin_port: 8003  # optional; defaults to fed_learn_port if omitted

このプロファイルには、Project Admin が所有する 3 つの情報が記述されます。

- **name**: プロジェクト名です。すべての参加者定義でこの文字列を正確に一致させる必要が
  あります。
- **scheme** と **connection_security**: トランスポートの設定です ( ``grpc`` /
  ``tls`` がデフォルトです)。
- **server**: クライアントやユーザーが接続するサーバーのエンドポイントです。
  ``cert approve`` はこの値を読み取り、署名済み zip すべてに埋め込みます。そのため、
  リクエスター自身がこの情報を用意する必要はありません。

続いて Project Admin は、フェデレーションごとに一度だけ ``cert init`` を実行します。

.. code-block:: shell

   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00

これにより、次のものが作成されます。

- ``./ca/rootCA.pem``: ルート CA 証明書
- ``./ca/rootCA.key``: ルート CA の秘密鍵です。秘密として厳重に管理してください。
- ``./ca/ca.json``: ``nvflare cert init`` が生成する内部的な CA メタデータです。
  ``nvflare cert approve`` が使用するプロジェクト名、デプロイバージョン、ルート CA の
  フィンガープリントを保持します。手動で編集しないでください。

``init`` の主なオプション:

- ``--profile``: プロジェクトプロファイルの yaml ファイルへのパスです。必須です。ファイルは
  このパスから直接読み込まれ、自動的なファイル探索は行われません。
  ``cert init`` は ``name`` フィールドのみを読み取ります。プロファイル全体は後で
  ``cert approve`` が使用します。
- ``--deploy-version``: ``ca.json`` に記録され、承認済み zip に署名付きで含まれるデプロイ
  世代です。デフォルトは ``00`` です。通常はこのオプションを気にする必要はありません。
  ``00`` は ``prod_00`` に対応します。新しいデプロイ用の CA や世代を意図的に作成する場合に
  のみ ``01`` 、 ``02`` などを使用してください。
- ``--org``: ルート CA 証明書の O フィールドに設定する組織名です (省略可)。
- ``-o, --output-dir``: CA の出力ディレクトリです。必須です。
- ``--valid-days``: ルート CA の有効期間 (日数) です。デフォルトは ``3650`` です。
- ``--force``: 新しいデプロイバージョンを使用する場合に限り、既存の CA ファイルを
  バックアップしたうえで置き換えます。既存のデプロイバージョンを新しいルート CA で
  再利用することは拒否されます。これは、異なる CA から生成されたスタートアップキットが
  同一の ``prod_<NN>`` ディレクトリに混在しないようにするためです。例えば ``00`` は
  ``prod_00`` に対応します。
- ``--schema``: このコマンドの JSON スキーマを出力します。

************************
リクエスト zip の作成
************************

リクエスターは、秘密鍵を保持すべきマシン上で ``cert request`` を実行します。このコマンドは
参加者定義ファイルを 1 つ読み込み、秘密鍵、CSR、メタデータ、およびリクエスト zip を作成します。

クライアントサイトのリクエスト:

.. code-block:: shell

   nvflare cert request --participant hospital-a.yaml

サーバーのリクエスト:

.. code-block:: shell

   nvflare cert request --participant server.yaml

ユーザーのリクエスト:

.. code-block:: shell

   nvflare cert request --participant alice.yaml

参加者定義は、集中型の ``project.yaml`` と同じトップレベル構造を使用します。すなわち、
プロジェクトを表す ``name`` フィールドと ``participants`` リストです。
分散プロビジョニングでは、``participants`` リストには **必ず 1 つのエントリだけ** を
含める必要があります。1 ファイル、1 アイデンティティ、1 リクエストです。キー名が複数形なのは
集中型のフォーマットとの一貫性を保つためですが、リストのエントリが 0 個または 2 個以上の場合、
``cert request`` はそのファイルを拒否します。

**参加者定義のフォーマット:**

*サーバー* — サーバー自身のリッスンポートを含みます。

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: server.example.com
       type: server
       org: nvidia
       fed_learn_port: 8002
       admin_port: 8003  # optional; defaults to fed_learn_port if omitted

.. note::

   ``fed_learn_port`` と ``admin_port`` は、このサーバーがローカルで *リッスンする* ポートです。
   ``admin_port`` は省略可能で、省略した場合は ``fed_learn_port`` がデフォルトとして使われます。

   ``nvflare cert approve`` は、リクエスト zip 内の ``name`` フィールドが ``ca.json`` および
   ``project_profile.yaml`` のプロジェクト名と一致することを検証します。プロジェクト名が
   一致しないリクエストは拒否されます。

*クライアント* — 必要なのは name、type、org のみです。

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: hospital-a
       type: client
       org: hospital_alpha

*ユーザー (FL 管理者)* — ``type: admin`` は FL の管理者ユーザー (org_admin、lead、または member) を示すものであり、CA を管理する Project Admin を指すものではありません。``role`` を追加してください。

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: alice@hospital.org
       type: admin
       org: hospital_alpha
       role: lead

``hospital-b`` のような 2 人目の参加者をリクエストする場合は、単一のエントリだけを持つ別の
``hospital-b.yaml`` ファイルを作成し、``cert request`` を再度実行してください。参加者ごとに
独自の秘密鍵とリクエスト zip が作成されます。

.. note::

   **リクエスターが Project Admin から (帯域外で) 受け取る必要がある情報:**

   - **プロジェクト名** ( ``name:`` ): すべての参加者定義で、Project Admin の
     ``project_profile.yaml`` にあるプロジェクト名を正確に使用する必要があります。

   クライアントおよびユーザーの定義には、サーバーのエンドポイントは含まれません。Project
   Admin は ``cert approve`` の実行時に、``project_profile.yaml`` にある承認済みのサーバー
   ホスト、ポート、スキーム、接続セキュリティの設定をすべての署名済み zip に埋め込みます。
   リクエスターはこれらの情報を署名済み zip の中で受け取るのであって、自身の参加者定義
   ファイルに書き込むわけではありません。

デフォルトでは、``cert request`` は ``./<name>/`` に出力します。``hospital-a`` の場合は
次のようになります。

.. code-block:: text

   hospital-a/
     hospital-a.key
     hospital-a.csr
     site.yaml
     request.json
     hospital-a.request.zip

このディレクトリ内のファイルはすべて ``cert request`` によって自動生成されます。
手動で編集しないでください。``request.json`` にはリクエストのメタデータが記録され、
後で ``nvflare package`` が署名済み zip を検証する際に使用します。

Project Admin に送るのは ``hospital-a.request.zip`` だけです。秘密鍵はローカルに残り、
zip には含まれません。

``request`` の主なオプション:

- ``-p, --participant``: 参加者定義ファイルです。必須です。
- ``--out``: リクエストフォルダーです。デフォルトは ``./<participant-name>`` です。
- ``--force``: 既存のリクエストファイルを上書きします。
- ``--schema``: このコマンドの JSON スキーマを出力します。

**************************
リクエスト zip の承認
**************************

Project Admin は、プロジェクトの CA とプロジェクトプロファイルを指定して ``cert approve`` を
実行します。

.. code-block:: shell

   nvflare cert approve hospital-a/hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml

これにより、リクエスト zip の検証、リクエストのプロジェクトが CA およびプロジェクト
プロファイルと一致することの確認、CSR への署名、``project_profile.yaml`` にある Project Admin
承認済みのサーバーエンドポイントの注入、``signed.json`` への CA メタデータの署名が行われ、
次のものが作成されます。

.. code-block:: text

   hospital-a/hospital-a.signed.zip
     signed.json
     signed.json.sig
     site.yaml
     hospital-a.crt
     rootCA.pem

署名済み zip をリクエスターに返送してください。

``signed.json.sig`` は、``signed.json`` に対する Project Admin の CA 署名です。
``nvflare package`` は、承認済みのサーバーエンドポイント、スキーム、接続セキュリティの各
フィールド、および ``ca_info`` を信頼する前に、この署名を検証します。
``ca_info`` には、``nvflare package`` が使用する署名済みの CA メタデータが含まれます。

署名済み zip にはすでに ``rootCA.pem`` が含まれています。リクエスターは ``nvflare package``
を実行する前に、別途 ``rootCA.pem`` ファイルを受け取ったり配置したりする必要はありません。

コマンドの出力には ``rootca_fingerprint_sha256`` が含まれます。リクエスターが
``nvflare package`` の実行時に署名済み zip のルート CA を検証できるよう、このフィンガー
プリントの値のみを信頼できる帯域外のチャネルで共有してください。

署名済み zip の出力先を指定するには ``--out`` を使用します。

.. code-block:: shell

   nvflare cert approve hospital-a/hospital-a.request.zip \
       --ca-dir ./ca \
       --profile project_profile.yaml \
       --out ./signed/hospital-a.signed.zip

``approve`` の主なオプション:

- ``request_zip``: ``nvflare cert request`` が生成したリクエスト zip です。必須です。
- ``-c, --ca-dir``: ``rootCA.pem`` 、 ``rootCA.key`` 、 ``ca.json`` を含むディレクトリです。
  必須です。
- ``--profile``: ``name`` 、 ``scheme`` 、 ``connection_security`` 、および ``server``
  エンドポイントを含む Project Admin の ``project_profile.yaml`` です。
  必須です。
- ``--out``: 署名済み zip の出力パスです。デフォルトは、リクエスト zip と同じ場所にある
  ``<name>.signed.zip`` です。
- ``--valid-days``: 参加者証明書の有効期間 (日数) です。デフォルトは ``1095`` です。
- ``--force``: 既存の署名済み zip を上書きします。
- ``--schema``: このコマンドの JSON スキーマを出力します。

****************************
エンドツーエンドのフロー
****************************

リクエスター:

.. code-block:: shell

   nvflare cert request --participant hospital-a.yaml

Project Admin:

.. code-block:: shell

   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00
   nvflare cert approve hospital-a/hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml

リクエスター:

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>

参加者定義の例や成果物のレイアウトを含むワークフロー全体については、
:ref:`distributed_provisioning` を参照してください。

.. note::

   **互換性について:** 本リリースより前に生成されたリクエスト zip には
   ``site_yaml_sha256`` の整合性フィールドが含まれていないため、
   ``nvflare cert approve`` によって拒否されます。approve を実行する前に、
   ``nvflare cert request`` でリクエスト zip を再生成してください。
