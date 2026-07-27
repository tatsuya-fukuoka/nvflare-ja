**********************************
FLARE v2.3.0 の新機能
**********************************

クラウドデプロイメントのサポート
========================================
Dashboard UI と FL エンティティは、Azure と AWS の両方における :ref:`cloud_deployment` のサポートを拡張しました。
インフラの作成、デプロイ、そして Dashboard UI、FL サーバー、FL クライアントの起動を行うシンプルな CLI コマンドが
利用できるようになりました。

.. code-block:: bash

    nvflare dashboard --cloud azure | aws
    <server-startup-kit>/start.sh --cloud azure | aws
    <client-startup-kit>/start.sh --cloud azure | aws

これらの起動スクリプトは、必要なリソース、VM、ネットワーク、セキュリティグループを自動的に作成し、新しく作成された
インフラに FLARE をデプロイして FLARE システムを起動できます。

Python バージョンのサポート
----------------------------------
FLARE は Python 3.9 および Python 3.10 でサポートされるようになったため、FLARE 2.3.0 は Python バージョン 3.8、3.9、3.10 をサポートします。
Python 3.7 は、積極的なサポートおよびテストの対象ではなくなりました。

ユーザー体験を向上させる新しい FLARE API
------------------------------------------------
新しい FLARE API は、FLAdminAPI をより使いやすく改良したバージョンです。FLARE API は現在、一部のコマンドをサポートしています。
新しい FLARE API への移行の詳細については、:ref:`FLARE API への移行 <migrating_to_flare_api>` を参照してください。当面の間、FLAdminAPI も引き続き機能します。
FLARE API の詳細については、こちらのノートブックをご覧ください: https://github.com/NVIDIA/NVFlare/blob/2.3/examples/tutorials/flare_api.ipynb

セキュリティ向上のためのジョブ署名
------------------------------------------
ジョブがサーバーにサブミットされる前に、サブミッターの秘密鍵を使って各ファイルのダイジェストに署名し、カスタムコードが署名されていることを保証します。
各フォルダーには 1 つの署名ファイルがあり、ファイル名とそのフォルダー内のすべてのファイルの署名を対応付けます。署名検証のために、署名者の証明書も
含まれます。ジョブがデプロイされるまでクライアントはジョブを受け取らないため、検証はサブミット時ではなくデプロイ時に実行されます。

クライアント側でのモデル初期化
--------------------------------------
FLARE 2.3.0 より前は、モデルの初期化はサーバー側で行われていました。
モデルは、モデルファイルから初期化されるか、カスタムのモデル初期化コードによって初期化されていました。モデルファイルを事前定義するには、モデルファイルを
事前に生成して保存し、それをサーバーへ送信するという余分な手順が必要でした。また、サーバー上でカスタムのモデル初期化コードを実行することはセキュリティリスクになり得ました。

FLARE 2.3.0 では、クライアント側でモデルを初期化する別の方法を導入します。FL サーバーは、ユーザーが選択したストラテジーに基づいて
初期モデルを選択できます。クライアント側モデル初期化を使用した例はこちらです: https://github.com/NVIDIA/NVFlare/tree/2.3/examples/hello-world/hello-pt
この機能の詳細については、:ref:`initialize_global_weights_workflow` をご覧ください。

従来型の機械学習の例
-----------------------------
フェデレーテッドラーニングで従来型の機械学習アルゴリズムを利用できるよう、いくつかの新しい例が追加されました。
   - scikit-learn ライブラリを用いた :github_nvflare_link:`線形モデル <examples/advanced/sklearn-linear>`（
     `反復的な SGD 学習 <https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.SGDClassifier.html>`_ を使用）。
     この反復的な例に従い、異なる損失関数を採用することで、線形回帰やロジスティック回帰を実装できます。
   - scikit-learn ライブラリを用いた :github_nvflare_link:`SVM <examples/advanced/sklearn-svm>`。この 2 段階のプロセスでは、サーバーがクライアントから収集したサポートベクターに対して、さらに 1 ラウンドの SVM を実行します。
   - scikit-learn ライブラリを用いた :github_nvflare_link:`K-Means <examples/advanced/sklearn-kmeans>`（
     `ミニバッチ K-Means 手法 <https://scikit-learn.org/stable/modules/generated/sklearn.cluster.MiniBatchKMeans.html>`_ を使用）。
     この反復的なプロセスでは、各クライアントがミニバッチ K-Means を実行し、サーバーがグローバルモデルに向けて更新を同期します。
   - XGBoost ライブラリの `ランダムフォレスト機能 <https://xgboost.readthedocs.io/en/stable/tutorials/rf.html>`_ を用いた
     :github_nvflare_link:`ランダムフォレスト <examples/advanced/random_forest>`。この 2 段階のプロセスでは、クライアントが
     ローカルデータ上でサブフォレストを構築し、サーバーが収集したすべてのサブフォレストをアンサンブルしてグローバルなランダムフォレストを生成します。

垂直学習
-----------------

Federated Private Set Intersection (PSI)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
セキュアなユーザー ID マッチングや特徴量の重複発見といった垂直学習のユースケースをサポートするために、私たちは
データの交差集合を安全に発見できるマルチパーティの Private Set Intersection（PSI）オペレーターを開発しました。
このアプローチでは、ECDH と Bloom Filter に基づく OpenMined の 2 者間
`Private Set Intersection Cardinality プロトコル <https://github.com/OpenMined/PSI>`_ を活用し、このプロトコルを
マルチパーティで利用できるようにしました。私たちのアプローチと PSI オペレーターの使い方の詳細については、
:github_nvflare_link:`PSI の例 <examples/advanced/psi/README.md>` を参照してください。

なお、PSI は Split Learning の例において前処理ステップとして使用されている点に注目してください。その例はこちらの
:github_nvflare_link:`ノートブック <examples/advanced/vertical_federated_learning/cifar10-splitnn/README.md>` にあります。

Split Learning
~~~~~~~~~~~~~~
Split Learning は、垂直に分割されたデータ上でディープニューラルネットワークを学習させることを可能にします。本リリースでは、一方のクライアントが画像を保持し、
もう一方のクライアントが損失と精度メトリクスを計算するためのラベルを保持していると仮定して、CIFAR-10 データセットを用いて
`split learning <https://arxiv.org/abs/1810.06060>`_ を実行する方法を示す `例 <https://github.com/NVIDIA/NVFlare/blob/2.3/examples/advanced/vertical_federated_learning/cifar10-splitnn/README.md>`_ を含めています。

活性化値とそれに対応する勾配は、FLARE の新しい通信 API を用いてクライアント間で交換されます。

NLP の新しい例
-----------------------
新しい :github_nvflare_link:`NLP-NER の例 <examples/advanced/nlp-ner/README.md>` では、`Hugging Face <https://huggingface.co/>`_ の
`BERT <https://github.com/google-research/bert>`_ と `GPT-2 <https://github.com/openai/gpt-2>`__ の両モデル（`BERT-base-uncased <https://huggingface.co/bert-base-uncased>`_、`GPT-2 <https://huggingface.co/gpt2>`__）を、
`NCBI disease データセット <https://pubmed.ncbi.nlm.nih.gov/24393765/>`_ を用いた固有表現抽出（NER）タスクで紹介します。

研究領域
--------------

FedSM
~~~~~
:github_nvflare_link:`FedSM の例 <research/fed-sm/README.md>` では、CVPR 2022 に採択されたパーソナライズドフェデレーテッドラーニングアルゴリズム
`FedSM <https://arxiv.org/abs/2203.10144>`_ を紹介します。FedSM は SoftPull メカニズムを通じてクライアント間で異なるデータ分布を橋渡しし、
Super Model を活用します。モデルセレクターは、特定のサンプルがどのクライアントのパーソナライズドモデル、あるいはグローバルモデルに属するかを
予測するように学習されます。このモデルの学習はまた、極端なラベル不均衡を伴う困難なフェデレーテッドラーニングのシナリオも示しています。そこでは、
各ローカル学習が単一のラベルのみに基づいて行われる一方で、クライアント数と同じ数のクラスの分類に向けて最適化が行われます。この場合、
Adam オプティマイザーの高次モーメントもモデル更新とともに平均化・同期されます。

Auto-FedRL
~~~~~~~~~~
:github_nvflare_link:`Auto-FedRL の例 <research/auto-fed-rl/README.md>` は、ECCV 2022 に採択された
`Auto-FedRL: Federated Hyperparameter Optimization for Multi-institutional Medical Image Segmentation <https://arxiv.org/abs/2203.06338>`_ に記載された自動機械学習ソリューションを実装しています。
従来のハイパーパラメーター最適化アルゴリズムは、多数の学習トライアルを伴うため、実世界の FL アプリケーションではしばしば非現実的です。
限られた計算予算では、そうしたトライアルを賄えないことが多いためです。
Auto-FedRL は、効率的な強化学習（RL）ベースのフェデレーテッドハイパーパラメーター最適化アルゴリズムを提案します。
このアルゴリズムでは、オンラインの RL エージェントが現在の学習の進捗に基づいて各クライアントのハイパーパラメーターを動的に調整できます。

フェデレーテッドラーニングにおけるデータ漏洩の定量化
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~
この研究 :github_nvflare_link:`例 <research/quantifying-data-leakage/README.md>` には、IEEE Transactions on Medical Imaging に採択された
`Do Gradient Inversion Attacks Make Federated Learning Unsafe? <https://arxiv.org/abs/2202.06924>`_ で説明されている胸部 X 線実験を再現するために必要なツールが含まれています。
この例では、各クライアントのデータ漏洩を定量化し、それを FL 学習ラウンドの関数として可視化できる新しい FLARE フィルターを用いて、
FL における潜在的なデータ漏洩を測定・可視化する新しい方法を示します。
FL におけるデータ漏洩を定量化することで、差分プライバシーのようなプライバシー保護技術とモデル精度との間の最適なトレードオフを、定量可能な指標に基づいて判断する助けになります。

通信フレームワークのアップグレード
------------------------------------------
エンドユーザーにとって、設定や利用パターンに目に見える変更はないはずですが、基盤となる通信レイヤーは
より高い柔軟性と性能を実現するために改善されました。これらの新しい通信機能は、次のリリースで一般提供される予定です。

**********************************
2.3.0 への移行: 注意点とヒント
**********************************
2.3.0 では、いくつかの API と挙動の変更が導入されています。この移行ガイドは、以前の NVFLARE バージョンから現在のバージョンへ移行する際に役立ちます。

1. FLARE API
------------
FLARE API は、バージョン 2.3 においてより良いユーザー体験のために再設計された FLAdminAPI です。FLARE API の使い方、FLAdmin API との関係、
および移行手順を理解するには、:ref:`FLARE API への移行 <migrating_to_flare_api>` を参照してください。

2. ``list_jobs`` コマンドの機能強化
--------------------------------------------
``list_jobs`` コマンドに、サブミット時刻の新しい順（逆時系列順）で結果を表示する ``-r`` オプションが追加されました。また、返されるジョブの
最大数を制限する ``-m`` オプションも追加されました。

3. 通信レイヤーの再設計
----------------------------------
NVFLARE 2.3.0 には新しい通信レイヤーが搭載されています。本格的な機能が一般提供されるのは次のリリースになりますが、
基盤となる通信エンジンはすでに置き換えられており、ログ出力に変化が見られる場合があります。

そのため、:class:`ClientEngineExecutorSpec<nvflare.private.fed.client.client_engine_executor_spec.ClientEngineExecutorSpec>` において、通信関連の API をいくつか変更する必要がありました。


FLARE 2.2.x

.. code-block:: python

    @abstractmethod
    def send_aux_request(self, topic: str, request: Shareable, timeout: float, fl_ctx: FLContext) -> Shareable:
      """Send a request to Server via the aux channel.

      Implementation: simply calls the ClientAuxRunner's send_aux_request method.

      Args:
          topic: topic of the request
          request: request to be sent
          timeout: number of secs to wait for replies. 0 means fire-and-forget.
          fl_ctx: FL context

      Returns: a reply Shareable

      """
      pass

FLARE 2.3.0

.. code-block:: python

    @abstractmethod
    def send_aux_request(
      self,
      targets: Union[None, str, List[str]],
      topic: str,
      request: Shareable,
      timeout: float,
      fl_ctx: FLContext,
      optional=False,
    ) -> dict:
      """Send a request to Server via the aux channel.

      Implementation: simply calls the ClientAuxRunner's send_aux_request method.

      Args:
          targets: aux messages targets. None or empty list means the server.
          topic: topic of the request
          request: request to be sent
          timeout: number of secs to wait for replies. 0 means fire-and-forget.
          fl_ctx: FL context
          optional: whether the request is optional

      Returns:
          a dict of reply Shareable in the format of:
              { site_name: reply_shareable }

      """

4. Controller の挙動変更
------------------------------
:class:`ControllerSpec<nvflare.apis.controller_spec.ControllerSpec>` の内部で、``wait_time_after_min_received`` の挙動が変更され、
すべてのレスポンスを受信した場合には待機しないようになりました。

.. code-block:: python

    class ControllerSpec(ABC):

        def broadcast(
          self,
          task: Task,
          fl_ctx: FLContext,
          targets: Union[List[Client], List[str], None] = None,
          min_responses: int = 0,
          wait_time_after_min_received: int = 0,
        ):

リリース 2.3.0 より前:

Wait_time_after_min_received: min_response を受信した後、wait_time_after_min_received の時間だけ待機することを意味します。

リリース 2.3.0 では:

Wait_time_after_min_received: min_response を受信したものの、すべてのレスポンスを受信していない場合に、wait_time_after_min_received の時間だけ待機します。
すべてのレスポンスを受信した場合は、待機しません。

5. POC ``–stop`` の挙動変更
------------------------------------
2.2.x のバージョンでは、POC の stop はシステムの状態にかかわらず、プロセスを直接強制終了しようとしていました。

2.3.0 のバージョンでは、stop コマンドは次の手順を試みます。

  #. サーバーに接続します
  #. サーバーに接続できた場合は、アクティブなジョブを一覧表示します
  #. すべてのアクティブなジョブを中止します
  #. システムのシャットダウンを呼び出し、システムが段階的にシャットダウンするのを待ちます
  #. 最大 30 秒のタイムアウトでシステムのシャットダウンを待ちます
  #. その後、プロセスの強制終了を試みます（これが 2.2.x の挙動のすべてでした）

6. Scatter and Gather Controller の API 変更
--------------------------------------------------
:class:`ScatterAndGather<nvflare.app_common.workflows.scatter_and_gather.ScatterAndGather>` に新しい引数が追加されました。``allow_empty_global_weights`` は、
空のグローバルウェイトを許可するかどうかを決めるオプションのブール値で、デフォルトは False です。

パイプラインによっては、最初のラウンドでグローバルウェイトが空になり、クライアントがグローバル情報なしでゼロから学習を開始する場合があります。

7. Job Scheduler の設定に関する更新
---------------------------------------------
Job Scheduler をさまざまな引数で設定する方法については、:ref:`job_scheduler_configuration` を参照してください。
