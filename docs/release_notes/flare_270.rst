*****************************
FLARE v2.7.0 の新機能
*****************************

新機能は以下のカテゴリに分類できます。


機密フェデレーテッド AI
=======================

.. sidebar::

   **機密フェデレーテッド AI のアプリケーション:**

   - **共同 R&D と分析**: Confidential Computing のエンクレーブ内で共同のモデルトレーニングやエージェントベースの分析を行い、データと IP を保護します。
   - **モデル保護**: プロプライエタリまたはライセンス供与された基盤モデルを安全に利用・ファインチューニングします。
   - **規制対象および国境をまたぐ AI**: プライバシー、主権、コンプライアンスを維持しながら、金融、ヘルスケア、防衛、グローバルパートナー間での協働を可能にします。
   - **セキュアなデプロイ**: 信頼できない環境における推論時のモデルやデータの漏洩を防ぎます。



本リリースでは、Confidential Computing を用いたフェデレーテッド構成におけるエンドツーエンドの IP 保護ソリューションとして、
この種のものとしては初となる製品を提供します。

- 本ソリューションは、AMD CPU と NVIDIA GPU を用いた Confidential VM によるベアメタル上のオンプレミス配備を対象としています。
- エンドツーエンドの保護: エンドツーエンドの保護とは、実行時に使用される IP (モデルとコード) を保護するだけでなく、デプロイ時の CVM の改ざんに対しても保護することを意味します。
- 本ソリューションは以下を実行できます。

    - モデルを介したプライバシー漏洩を防ぐための、サーバー側での **セキュア集約**
    - 協働中のモデル IP を保護するための、クライアント側での **モデル窃取対策**
    - 事前承認された認証済みコードによる、クライアント側での **データ漏洩防止**

.. admonition:: 機密フェデレーテッド AI

    この機能は **テクニカルプレビュー** です。
    CVM のビルドスクリプトについては NVIDIA FLARE チームまでお問い合わせください: federatedlearning@nvidia.com

    ユーザーによる利用方法の詳細は :ref:`confidential_computing` を参照してください。


FLARE Core
----------

ジョブレシピ (Job Recipe)
~~~~~~~~~~~~~~~~~~~~~~~~~~

.. sidebar::

    以下は FedAvg の Job Recipe の例です

    .. code-block:: python

            n_clients = args.n_clients
            num_rounds = args.num_rounds
            batch_size = args.batch_size

            recipe = FedAvgRecipe(
                name="hello-pt",
                min_clients=n_clients,
                num_rounds=num_rounds,
                model=SimpleNetwork(),
                train_script="client.py",
                train_args=f"--batch_size {batch_size}",
            )
            add_experiment_tracking(recipe, tracking_type="tensorboard")

            env = SimEnv(num_clients=n_clients)
            run = recipe.execute(env)
            print()
            print("Result can be found in :", run.get_result())
            print("Job Status is:", run.get_status())
            print()


新しい Flare Job Recipe を紹介します。これは、クライアントのトレーニングロジックとサーバー側のアルゴリズムを指定するために必要なコードを、軽量な形で記述する方法です。
同じ Job Recipe を SimEnv、PoCEnv、ProdEnv でシームレスに実行でき、ローカルでの実験から本番デプロイまでをカバーします。

Flare Job Recipe により、データサイエンティストにとってのフェデレーテッドラーニングのワークフローが劇的に簡素化されます。
ほとんどの場合、完全なフェデレーテッドラーニングのジョブを構成するのに必要な Python コードはわずか 6 行程度です。
Client API (通常 4 行程度) と組み合わせれば、フェデレーテッドラーニングの実験の構築と実行はほとんど労力を要しないものになります。


.. admonition:: Job Recipe

    この機能は **テクニカルプレビュー** です。すべての例やコードがまだ Job Recipe を使用するように変換されているわけではありません。
    ただし、レシピのチュートリアルノートブック `Job Recipe Tutorials <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/job_recipe.ipynb>`_
    で直接レシピを体験したり、:ref:`job_recipe` を読んだりすることができます。すぐに使えるレシピが半ダース以上提供されています: :ref:`quickstart`


通信の強化: ポートの統合と新しい HTTP ドライバー
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. sidebar::

    **ポートの統合**
    従来、FLARE のサーバーには 2 つの独立したポートが必要でした。1 つは FL クライアント/サーバー間通信用、もう 1 つは
    Admin クライアント/サーバー間通信用です。2.7 では、これらが設定可能な単一のポートに統合され、ネットワーク設定の複雑さが軽減されます。
    より厳格なネットワークポリシーを持つ環境向けに、デュアルポートモードも引き続き利用可能です。

    **新しい HTTPS ドライバー**
    HTTP ドライバーは、従来の性能上の制約に対処するため aiohttp を用いて書き直されました。同一の API、TLS サポート、
    既存デプロイとの後方互換性を維持しつつ、gRPC と同等の性能を実現しています。


- **ポートの統合**: 2 つのポートから単一のポートに削減され、デプロイが簡素化されます。
- **標準ポートとの互換性**: 標準の HTTPS ポート 443 を使用できます - IT 部門に追加のポートを開放してもらう必要はありません
- **高い性能**: 新しい HTTP ドライバーは、速度と信頼性において gRPC と同等です。

.. admonition:: この機能が重要な理由

    **デプロイの高速化**: カスタムポートの開放に伴うネットワーク設定の遅延や IT 部門の承認が不要になります。
    FLARE 2.7.0 はネットワーク要件を簡素化し、標準的なインフラ上で完全にセキュアなデプロイを可能にします。
    :ref:`FL サーバーのポート統合の詳細はこちら <server_port_consolidation>`。

セキュリティの強化
~~~~~~~~~~~~~~~~~~

以下の問題を修正しました。

- 安全でないデシリアライズ - torch.jit.load を safe-tensor ベースの実装に置き換えました
- 安全でないデシリアライズ - 関数呼び出し - FOB の自動登録を削除しました。FOB のホワイトリストが自動登録されます。
- Grep パラメータ経由のコマンドインジェクション - コマンドインジェクションを回避するようにコマンドを再実装しました


.. admonition:: セキュリティの強化

    同様の問題も多数修正されています



FLARE によるエッジアプリケーションの開発
----------------------------------------

.. sidebar::

   .. image:: ../resources/hierarchical_fl.png
        :height: 150px

   .. image:: ../resources/edge_cross_device_fl.png
        :height: 150px

   .. image:: ../resources/edge_simplify_device_programming.png
        :height: 150px



FLARE 2.7 は、エッジ環境固有の課題に直接対応する機能によって、フェデレーテッドラーニングをエッジデバイスへと
拡張します。

**スケーラビリティ**: **階層型フェデレーテッドアーキテクチャ** :ref:`flare_hierarchical_architecture` により、数百万台のエッジデバイスが
それぞれサーバーへ直接接続することなく効率的に参加できます。

**断続的なデバイス参加**: FedBuff に基づく **非同期 FL** :ref:`flare_edge` は、ネットワークや電源の中断によって参加、
離脱、あるいはローカルトレーニング結果の返却に失敗する可能性のあるデバイスに対応します。

**クロスプラットフォームかつデバイスプログラミング不要**: データサイエンティストは、Swift、Objective-C、Java、Kotlin を記述することなく
iOS および Android :ref:`flare_mobile` にモデルをデプロイできます。FLARE が PyTorch → Executorch の変換とデバイス上のトレーニングコードを自動的に処理します。

**シミュレーションツール**: 大規模テスト用のデバイスシミュレータ (:ref:`device_simulation`)


.. admonition:: FLARE Edge

    `エッジの例 <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ に沿って FLARE のエッジ開発をお試しください



自習型トレーニングチュートリアル
--------------------------------

NVIDIA FLARE によるフェデレーテッドラーニングの 5 部構成コースへようこそ。
本コースでは、基礎から高度な応用、システムのデプロイ、プライバシー、セキュリティ、
そして実世界の業界ユースケースまで、あらゆる内容を扱います。

.. admonition:: NVIDIA FLARE によるフェデレーテッドラーニング

    このチュートリアルには **100 本以上のノートブック** と **80 本の動画** が含まれています。
    詳細は :ref:`self_paced_training` を参照してください。


追加機能
--------

バージョン 2.7.0 では、大規模モデルのストリーミングにおける FileDownloader によるメモリ管理の改善など、その他の新機能もリリースされています。詳細は :ref:`extra_270` を参照してください。
