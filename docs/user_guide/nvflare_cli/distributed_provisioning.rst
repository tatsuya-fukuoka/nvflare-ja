.. _distributed_provisioning:

##############################
分散プロビジョニング
##############################

分散プロビジョニングでは、各参加者が自分自身の秘密鍵を作成し、そのまま自分で保持できます。
プロジェクト管理者は証明書要求に署名しますが、参加者の秘密鍵を受け取ることは決してありません。

サイト、サーバー、ユーザーをそれぞれ独立してプロビジョニングしたい場合は、このワークフローを
使用します。公開されているワークフローは 3 つのコマンドを使用します:

.. code-block:: shell

   nvflare cert request --participant <participant.yaml>
   nvflare cert approve <request.zip> --ca-dir <ca-dir> --profile <project_profile.yaml>
   nvflare package <signed.zip> --fingerprint <rootca_fingerprint_sha256>

大まかな流れは次のとおりです:

1. プロジェクト管理者が ``project_profile.yaml`` を作成し、その明示的なプロファイルファイルから
   プロジェクト CA を初期化します。デプロイバージョンのデフォルトは ``00`` です。
2. サーバー管理者がサーバーのホスト名とポートを決定し、それらをプロジェクト管理者に共有します。
3. プロジェクト管理者は、リクエストを承認する前に、承認済みのサーバーエンドポイントを
   ``project_profile.yaml`` に記録します。
4. 要求者は、プロジェクト名を使って参加者定義ファイルを作成します。
5. 要求者は ``nvflare cert request`` を実行し、生成されたリクエスト zip のみをプロジェクト
   管理者に送信します。
6. プロジェクト管理者はリクエスト zip を承認し、承認済みのサーバーエンドポイントを含む署名済み
   zip を返します。
7. プロジェクト管理者は、信頼できる帯域外チャネルを通じて ``rootca_fingerprint_sha256`` を
   共有します。
8. 要求者は、ローカルの秘密鍵を保持しているマシン上で署名済み zip をパッケージ化します。

生成されたスタートアップキットは、中央集権的にプロビジョニングされたスタートアップキットと
同じように使用します。

.. note::

   プロジェクト管理者の承認が対象とするのは、署名済み zip の中に含まれるものだけです。すなわち、
   参加者のアイデンティティ、署名済み証明書、 ``rootCA.pem`` 、フェデレーションの接続パラメータ
   （サーバーエンドポイント、 ``scheme`` 、 ``connection_security`` ）、および署名済みの CA 情報
   です。参加者定義ファイル内の ``builders:`` ブロックは、リクエスト zip にも署名済み zip にも
   **含まれず** 、プロジェクト管理者によって承認されることもなく、要求者のマシン上で適用される
   ローカルなパッケージ時の挙動のままです。すべての参加者にわたって協調したビルダー設定を必要と
   する機能（準同型暗号など）は直接サポートされていません。そのようなデプロイでは、中央集権的な
   ``nvflare provision`` を使用してください。

********************************************
開始前に: 接続情報を記録する
********************************************

承認を開始する前に、プロジェクト名とサーバーエンドポイントが確定している必要があります。
新しいデプロイ CA 世代のための任意のデプロイバージョンもありますが、通常はこれを無視して
デフォルトの ``00`` を使用します。

**1. プロジェクト名**

プロジェクト管理者は、 ``project_profile.yaml`` を記述するときにプロジェクト名を決めます。
すべての参加者定義ファイルは、 ``name:`` フィールドでこのとおりの名前を使用しなければなりません。

**2. サーバーのホストとポート**

サーバー管理者は、プロビジョニングを開始する前に、FL サーバーのホスト名（または DNS 名）、
``fed_learn_port`` 、 ``admin_port`` を決定します。サーバー管理者はこれらの値をプロジェクト管理者
に伝え、プロジェクト管理者がそれらを ``project_profile.yaml`` に記録します。
``nvflare cert approve`` は、このエンドポイントを署名済み zip に署名して埋め込みます。
クライアントおよびユーザーの参加者定義ファイルには、ローカルな ``server:`` ブロックは含まれません。

**任意: デプロイバージョン**

プロジェクト管理者は通常このオプションを無視します。デフォルトは ``00`` です:

.. code-block:: shell

   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00

デプロイバージョンは ``ca.json`` に保存される内部メタデータであり、各承認 zip に署名して
埋め込まれます。 ``nvflare package`` はスタートアップキットを ``prod_<NN>`` に書き出します。
デフォルトの ``00`` の場合、同じ CA によって承認された複数の参加者は、同じ ``prod_00``
ディレクトリにパッケージ化されます。 ``01`` 、 ``02`` などは、新しいデプロイ CA/世代を意図的に
作成する場合にのみ使用してください。

要求者による検証のために帯域外で共有される唯一の値は、承認後の
``rootca_fingerprint_sha256`` です。

******************************
プロジェクトプロファイル
******************************

プロジェクト管理者は、フェデレーションごとに一度だけ ``project_profile.yaml`` を作成します。
これは軽量なプロジェクトプロファイルであり、中央集権的なプロビジョニングで使う完全な
``project.yaml`` ではありません。

.. code-block:: yaml

   name: hospital_federation
   scheme: grpc
   connection_security: tls
   server:
     host: server1.hospital-central.org
     fed_learn_port: 8002
     admin_port: 8003

フィールド:

- ``name``: プロジェクト名です。リクエストはこの値と一致しなければなりません。
- ``scheme``: FLARE の通信ドライバです。 ``grpc`` 、 ``tcp`` 、 ``http`` などを指定します。
- ``connection_security``: プロジェクト既定の接続セキュリティです。 ``tls`` 、 ``mtls`` 、
  ``clear`` などを指定します。
- ``server``: プロジェクト管理者が承認した FL サーバーのエンドポイントです。承認の前にサーバー
  管理者から提供されます。

プロジェクト管理者はこのファイルをローカルに保管します。承認時に、 ``scheme`` 、既定の
``connection_security`` 、および ``server`` エンドポイントが署名済み zip のメタデータに
コピーされるため、参加者はプロファイルファイルを必要としません。

********************************
参加者定義ファイル
********************************

各要求者は、1 つのアイデンティティに対して参加者定義ファイルを作成します。このファイルは
中央集権的な ``project.yaml`` と同じ ``participants`` 構造を使用しますが、ローカルの参加者だけを
含みます。

クライアントサイトの例:

.. code-block:: yaml

   name: hospital_federation
   description: Site A - Hospital Alpha

   participants:
     - name: hospital-a
       type: client
       org: hospital_alpha

参加者の証明書のコモンネーム（CN）を FLARE のサイト名と意図的に異なるものにする場合は、証明書
要求を作成する前に、参加者定義に ``auth_identity`` を含めてください。たとえば、証明書 CN
``hospital-a.example.com`` で認証する ``hospital-a`` という名前のサイトは、次のように要求します:

.. code-block:: yaml

   name: hospital_federation
   description: Site A - Hospital Alpha

   participants:
     - name: hospital-a
       type: client
       org: hospital_alpha
       auth_identity: hospital-a.example.com

これにより、生成されるスタートアップキットはエンドポイント/FQCN を、期待される mTLS 証明書の
アイデンティティにマッピングできます。スタートアップキットに ``startup/signature.json`` が
含まれている場合は、パッケージ化後に生成された ``startup/fed_client.json`` や
``startup/fed_server.json`` を編集しないでください。署名が最終的な構成をカバーするよう、
リクエスト、承認、パッケージを作り直してください。

.. important::

   エンドポイントと CN の紐付けは、mTLS トランスポートが認証済みピアの CN を公開している場合に
   強制されます。一部のアクティブ側の gRPC 接続はピアの CN を Python ドライバに公開しないため、
   NVFlare はそれらのアクティブ接続を受け入れ、パッシブ側のエンドポイント検証と、通常の
   アプリケーション層の認証チェックに依存します。

   管理コンソールのセルはセッションごとのエンドポイント名を使用し、管理用リスナー上で証明書/
   ユーザーアイデンティティによって管理ユーザーを認証します。管理者向けのアプリケーション層認証を
   設定し、保護された状態に保ってください。

   ``auth_identity`` は FLARE プロセスの起動時に読み込まれます。サイト証明書のローテーションや
   証明書 CN の変更を行った後は、必要に応じてスタートアップキットを再生成し、影響を受ける FLARE
   プロセスを再起動して、メモリ上のアイデンティティリゾルバが新しい証明書のアイデンティティを
   使用するようにしてください。

ユーザーの例:

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: alice@hospital-alpha.org
       type: admin
       org: hospital_alpha
       role: lead

サーバーの例:

.. code-block:: yaml

   name: hospital_federation
   description: Central FL server for hospital network

   participants:
     - name: server1.hospital-central.org
       type: server
       org: hospital_central
       host_names:
         - 10.0.1.50
         - fl-server.internal
       connection_security: mtls

クライアントとユーザーでは、サーバーエンドポイントのフィールドは意図的に存在しません。
``nvflare package`` は、 ``nvflare cert approve`` が作成した署名済み zip から承認済みの
エンドポイントを取得します。

サーバーの場合、 ``connection_security`` は任意のローカルな上書き設定です。これは、サーバーの
スタートアップキットをローカルのリクエストフォルダからパッケージ化する場合にのみ使用されます。
プロジェクト管理者によって承認されるものではなく、フェデレーションのポリシーとして配布されることも
ありません。サーバー定義でこれを設定しない場合、パッケージ処理は署名済み zip に含まれる
プロジェクト既定値を使用します。

ユーザーのロールは証明書上のロールです。サポートされる値は ``org_admin`` 、 ``lead`` 、
``member`` です。スタディ固有のロールやスタディへの所属は、後からスタディコマンドによって
割り当てられます。

****************************************
クイックスタート: サイトを追加する
****************************************

この例では、プロジェクト ``hospital_federation`` に対して ``hospital-a`` という名前の
クライアントサイトをプロビジョニングします。

プロジェクト管理者が ``project_profile.yaml`` を作成します:

.. code-block:: yaml

   name: hospital_federation
   scheme: grpc
   connection_security: tls
   server:
     host: server1.hospital-central.org
     fed_learn_port: 8002
     admin_port: 8003

プロジェクト管理者は、プロジェクト CA を一度だけ初期化します:

.. code-block:: shell

   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00

.. note::

   サーバーのホストとポートはサーバー管理者が決定します。プロジェクト管理者は、リクエストを
   承認する前にそれらを ``project_profile.yaml`` に記録します。これらの値は承認 zip に署名して
   埋め込まれるため、クライアントやユーザーの参加者定義ファイルにコピーする必要はありません。

サイト管理者は、プロジェクト名と参加者のアイデンティティを記載した ``hospital-a.yaml`` を
作成します:

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: hospital-a
       type: client
       org: hospital_alpha

サイト管理者がリクエストを作成します:

.. code-block:: shell

   nvflare cert request --participant hospital-a.yaml

これにより次が作成されます:

.. code-block:: text

   hospital-a/
     hospital-a.key
     hospital-a.csr
     site.yaml
     request.json
     hospital-a.request.zip

``hospital-a/hospital-a.request.zip`` をプロジェクト管理者に送信します。 ``hospital-a.key`` は
送信しないでください。リクエスト zip に秘密鍵は含まれていません。

プロジェクト管理者がリクエストを承認します:

.. code-block:: shell

   nvflare cert approve hospital-a/hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml

これにより（リクエスト zip の隣に） ``hospital-a/hospital-a.signed.zip`` が作成され、
``rootca_fingerprint_sha256`` が出力されます。署名済み zip にはすでに ``rootCA.pem`` が
含まれています。署名済み zip をサイト管理者に返し、フィンガープリントのみを信頼できる帯域外
チャネルで共有してください。

サイト管理者がスタートアップキットをパッケージ化します:

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>

出力は次の場所に置かれます:

.. code-block:: text

   workspace/hospital_federation/prod_00/hospital-a/

``00`` というディレクトリ名は、CA の ``provision_version`` に由来します。同じ CA/バージョンで
承認された他の参加者も、同じ ``prod_00`` パッケージルート配下に追加されます。

サイトを起動します:

.. code-block:: shell

   cd workspace/hospital_federation/prod_00/hospital-a
   ./startup/start.sh

****************************************
クイックスタート: ユーザーを追加する
****************************************

要求者は、プロジェクト名とユーザーのアイデンティティを記載した ``alice.yaml`` を作成します:

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: alice@hospital-alpha.org
       type: admin
       org: hospital_alpha
       role: lead

要求者がユーザーのリクエストを作成します:

.. code-block:: shell

   nvflare cert request --participant alice.yaml

``alice@hospital-alpha.org/alice@hospital-alpha.org.request.zip`` をプロジェクト管理者に
送信します。

プロジェクト管理者がそれを承認します:

.. code-block:: shell

   nvflare cert approve alice@hospital-alpha.org/alice@hospital-alpha.org.request.zip --ca-dir ./ca --profile project_profile.yaml

要求者は、返された署名済み zip をパッケージ化します:

.. code-block:: shell

   nvflare package alice@hospital-alpha.org/alice@hospital-alpha.org.signed.zip --fingerprint <rootca_fingerprint_sha256>

生成されたユーザー向けスタートアップキットには ``startup/fl_admin.sh`` が含まれます。

.. code-block:: shell

   cd workspace/hospital_federation/prod_00/alice@hospital-alpha.org
   ./startup/fl_admin.sh

****************************************
クイックスタート: サーバーを追加する
****************************************

サーバー管理者はまずホスト名とポートを決定し、それらの値をプロジェクト管理者に伝えます。
プロジェクト管理者は、承認の前にそれらを ``project_profile.yaml`` に記録します。

サーバー管理者が ``server.yaml`` を作成します:

.. code-block:: yaml

   name: hospital_federation

   participants:
     - name: server1.hospital-central.org
       type: server
       org: hospital_central
       host_names:
         - 10.0.1.50
         - fl-server.internal
       connection_security: mtls

サーバー管理者はプロジェクト管理者に次の内容を伝えます:

.. code-block:: text

   Server host:    server1.hospital-central.org
   fed_learn_port: 8002
   admin_port:     8003

プロジェクト管理者は、これらの値を ``project_profile.yaml`` の ``server:`` ブロックに記録します。
``cert approve`` は、そのエンドポイントを承認済みのすべての署名済み zip に署名して埋め込みます。

その後は、同じリクエスト、承認、パッケージのワークフローを使用します:

.. code-block:: shell

   nvflare cert request --participant server.yaml
   nvflare cert approve server1.hospital-central.org/server1.hospital-central.org.request.zip --ca-dir ./ca --profile project_profile.yaml
   nvflare package server1.hospital-central.org/server1.hospital-central.org.signed.zip --fingerprint <rootca_fingerprint_sha256>

サーバー参加者の名前は、中央集権的な ``project.yaml`` のサーバー参加者と同じ検証規則に従います。
本番環境では DNS 名を推奨します。ローカルやデモ用のワークフローでは、引き続き ``localhost`` も
有効です。追加の DNS 名や IP アドレスは ``host_names`` で追加できます。

************************
リモート転送フロー
************************

プロジェクト管理者と要求者が別々のマシンにいる場合、zip ファイルはマシン間でコピーされます。
秘密鍵は要求者のマシンに留まります。

要求者のマシン:

.. code-block:: shell

   nvflare cert request --participant hospital-a.yaml

次のファイルをプロジェクト管理者に転送します（たとえば、プロジェクト管理者の作業ディレクトリに
``hospital-a.request.zip`` としてコピーします）:

.. code-block:: text

   hospital-a/hospital-a.request.zip

プロジェクト管理者のマシン（作業ディレクトリに ``hospital-a.request.zip`` としてファイルを
受け取った状態）:

.. code-block:: shell

   nvflare cert approve hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml

次のファイルを要求者に返送します:

.. code-block:: text

   hospital-a.signed.zip

要求者のマシン（署名済み zip を、元の ``./hospital-a/`` リクエストフォルダとは別の作業
ディレクトリに置いた状態）:

.. code-block:: shell

   nvflare package hospital-a.signed.zip --request-dir ./hospital-a --fingerprint <rootca_fingerprint_sha256>

ここでは署名済み zip がローカルのリクエストフォルダの隣に存在しないため、 ``--request-dir`` が
必要です。

************************************
ルート CA フィンガープリントの確認
************************************

``nvflare cert approve`` は ``rootca_fingerprint_sha256`` を出力します。これは、プロジェクトの
``rootCA.pem`` に対する SHA256 証明書フィンガープリントです。プロジェクト管理者は、この値を
信頼できる帯域外チャネルで要求者に送信してください。

署名済み zip にはすでに ``rootCA.pem`` が含まれています。要求者は ``nvflare package`` を実行する
前に、別途ルート CA ファイルを用意する必要はありません。帯域外で受け取る値は、署名済み zip 内の
ルート CA を検証するためのフィンガープリントだけです。

``nvflare package`` は、署名済み zip から計算したフィンガープリントを常に出力します。パッケージ
コマンドに帯域外フィンガープリントを検証させるには、次のいずれかのオプションを使用します:

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>
   nvflare package hospital-a/hospital-a.signed.zip \
       --fingerprint <rootca_fingerprint_sha256>

期待される帯域外フィンガープリントが手元にある場合は
``--fingerprint <rootca_fingerprint_sha256>`` を使用します。帯域外フィンガープリントの検証を
意図的にスキップする場合は、フィンガープリントのオプションを省略します:

.. code-block:: shell

   nvflare package hospital-a/hospital-a.signed.zip

どちらのオプションも指定しない場合でも、パッケージ処理は署名済み zip、メタデータ、証明書チェーン、
ローカル秘密鍵との一致を検証しますが、プロンプトは表示せず、帯域外フィンガープリントの比較も
行いません。

``signed.json`` には、署名済みの CA メタデータも含まれます。 ``nvflare package`` は、署名済みの
フィンガープリントメタデータが、署名済み zip 内の ``rootCA.pem`` および既存の ``prod_<NN>``
パッケージルートと一致することを確認します。ルート CA の不一致は致命的なエラーです。この内部
チェックは CA の誤った混在を防ぎますが、署名済み zip で配布されるルート CA への信頼を確立するには、
依然として帯域外フィンガープリントが必要です。署名済み CA メタデータを持たない古い署名済み zip は、
デプロイバージョンを既定の ``00`` とみなし、このワークスペース整合性チェックには同梱の
``rootCA.pem`` から計算したフィンガープリントを使用します。

************************
ローカル自動化フロー
************************

ローカルでのテストでは、別のコマンド形態に切り替えるのではなく、同じ zip 成果物を使用します:

.. code-block:: shell

   # create project_profile.yaml and participant definition files
   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00
   nvflare cert request --participant hospital-a.yaml
   nvflare cert approve hospital-a/hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml
   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>

これはリモート承認と同じワークフローです。唯一の違いは、zip ファイルがマシン間でコピーされない
ことだけです。

****************
成果物
****************

リクエストフォルダ:

.. code-block:: text

   hospital-a/
     hospital-a.key          # private key, stays local
     hospital-a.csr          # CSR
     site.yaml               # full local participant definition
     request.json            # request metadata and hashes
     hospital-a.request.zip  # sent to Project Admin

リクエスト zip:

.. code-block:: text

   request.json
   site.yaml
   hospital-a.csr

署名済み zip:

.. code-block:: text

   signed.json
   signed.json.sig
   site.yaml
   hospital-a.crt
   rootCA.pem

``signed.json.sig`` は、 ``signed.json`` に対するプロジェクト管理者 CA の署名です。パッケージ処理
は、承認済みのエンドポイント、スキーム、接続セキュリティの値を信頼する前に、この署名を検証します。
``signed.json`` には、 ``nvflare package`` が使用する署名済みの CA メタデータも含まれます。

ローカルのリクエストフォルダにある ``site.yaml`` は、後で ``nvflare package`` が使用する完全な
参加者定義です。これには、サーバー側の ``connection_security`` の上書きなど、ローカルな
パッケージ時のフィールドを含めることができます。クライアントとユーザーのリクエストフォルダには
サーバーエンドポイントのブロックは含まれません。承認済みのエンドポイントは署名済み zip から
取得されます。

リクエスト zip および署名済み zip の中にある ``site.yaml`` は、サニタイズされた承認用メタデータ
です。承認とパッケージ化に必要なアイデンティティのフィールドを含みますが、秘密鍵は含みません。
サーバーエンドポイントのフィールドは、参加者定義ファイルからではなく、承認時に
``project_profile.yaml`` から注入されます。サーバー側の ``connection_security`` の上書きは、
ローカルなパッケージ時の挙動であるため、このサニタイズされたコピーからは除外されます。

リクエスト zip と署名済み zip に ``*.key`` ファイルが含まれてはいけません。署名済み zip は
スタートアップキットではなく、 ``nvflare package`` が使用する承認レスポンスです。

****************
信頼境界
****************

プロジェクト管理者の署名（ ``signed.json.sig`` ）が対象とするのは、署名済み zip の中に含まれる
ものそのものです:

- 参加者のアイデンティティ: name、org、type、role。
- 署名済みの参加者証明書（ ``*.crt`` ）とルート CA （ ``rootCA.pem`` ）。
- ``nvflare package`` が使用する署名済みの CA 情報。
- フェデレーションの接続パラメータ: ``scheme`` 、 ``connection_security`` 、および
  ``project_profile.yaml`` に記載されたサーバーエンドポイント。

それ以外はすべて **承認チェーンの外側** であり、ローカルなパッケージ時の挙動のままです:

- **カスタムビルダー**: ビルダーは、リクエスト zip、署名済み zip、署名済みメタデータのいずれにも
  含まれず、プロジェクト管理者によって承認されることは決してありません。 ``nvflare package`` は、
  ローカルの参加者定義にある ``builders:`` ブロックを読み取り、要求者のマシン上でパッケージ時に
  それらのビルダーを適用します。これは純粋にローカルな、パッケージ時の挙動です。各サイトが中央での
  調整なしに独自のビルダーを個別に設定するため、すべての参加者にわたって一致したビルダー設定を
  必要とする機能、たとえば準同型暗号（HE）や機密コンピューティング（CC）などは、分散
  プロビジョニングのワークフローでは直接サポートされません。
- **サーバー側の** ``connection_security`` **の上書き**: パッケージ時にローカルのサーバー
  リクエストフォルダから読み取られます。承認されず、配布もされません。
- **秘密鍵**: 要求者のマシンに留まり、どこにも送信されません。
- **ワークスペースのレイアウト**: ``nvflare package`` を実行するときに要求者が選択します。

すべての参加者にわたって中央で調整されたビルダー（HE、CC、その他の拡張）がデプロイに必要な場合は、
代わりに中央集権的な ``nvflare provision`` を使用してください。

****************
コマンド一覧
****************

プロジェクト管理者:

.. code-block:: shell

   # create project_profile.yaml
   nvflare cert init --profile project_profile.yaml -o ./ca --deploy-version 00
   nvflare cert approve hospital-a/hospital-a.request.zip --ca-dir ./ca --profile project_profile.yaml

要求者:

.. code-block:: shell

   nvflare cert request --participant hospital-a.yaml
   nvflare package hospital-a/hospital-a.signed.zip --fingerprint <rootca_fingerprint_sha256>

場所を明示的に指定する場合:

.. code-block:: shell

   nvflare cert request --participant ./defs/hospital-a.yaml --out ./requests/hospital-a
   nvflare cert approve ./requests/hospital-a/hospital-a.request.zip \
       --ca-dir ./ca \
       --profile ./project_profile.yaml \
       --out ./signed/hospital-a.signed.zip
   nvflare package ./signed/hospital-a.signed.zip \
       --request-dir ./requests/hospital-a \
       -w ./workspace \
       --fingerprint <rootca_fingerprint_sha256>

****************
注意事項
****************

- 秘密鍵はリクエストフォルダに留まり、プロジェクト管理者に送信されることはありません。
- クライアントとユーザーの参加者定義ファイルには、サーバーエンドポイントのブロックは含まれません。
  承認済みのエンドポイントは、署名済み zip を経由して ``project_profile.yaml`` から得られます。
- プロジェクト全体の ``scheme`` と既定の ``connection_security`` は、署名済み zip を経由して
  プロジェクト管理者のプロファイルから得られます。
- デプロイバージョンは ``cert init`` の CA メタデータに由来し、 ``prod_<NN>`` 出力ディレクトリを
  制御します。デフォルトは ``00`` で、 ``prod_00`` に対応します。新しいデプロイ CA/世代を意図的に
  作成する場合を除き、通常は無視してかまいません。
- サーバーの ``connection_security`` の上書きは、元のサーバー参加者定義からパッケージ時に
  ローカルで解決されます。
- 帯域外のルート CA フィンガープリントを検証するには
  ``--fingerprint <rootca_fingerprint_sha256>`` を使用します。省略してよいのは、帯域外
  フィンガープリントの検証を意図的にスキップする場合のみです。
- 署名済み zip から生成されたスタートアップキットは、通常の NVFlare ランタイムと互換性があります。
- 標準の分散プロビジョニングは ``signature.json`` を生成しません。信頼は、署名済みの参加者証明書と
  ``rootCA.pem`` を基点としています。カスタムビルダーや別のプロビジョニングモードが
  ``startup/signature.json`` を生成する場合は、パッケージ化後にスタートアップの JSON ファイルを
  変更せず、キットを再生成してください。
- カスタムビルダーは承認チェーンの一部ではありません。ローカルの参加者定義にある ``builders:``
  ブロックは、要求者のマシン上でパッケージ時に適用されます。すべての参加者にわたって協調した
  ビルダー設定を必要とする機能（HE、CC）は直接サポートされていません。そのようなデプロイでは、
  中央集権的な ``nvflare provision`` を使用してください。
