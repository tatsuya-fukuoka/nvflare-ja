.. _self_paced_training:

セルフペース学習チュートリアル
==============================

NVIDIA FLARE によるフェデレーテッドラーニング: ノートブックと動画
------------------------------------------------------------------
NVIDIA FLARE によるフェデレーテッドラーニングの全 5 部構成コースへようこそ。
本コースでは、基礎から高度な応用、システムデプロイ、プライバシー、セキュリティ、そして実世界の業界ユースケースまでを幅広く扱います。
**100 本以上** のノートブックと **80 本** の動画で構成されています。

.. note::

   これらのノートブックは NVFlare 2.6 で開発されました。すべての内容が最新の API を反映しているわけではなく、一部のサンプルは新しいバージョン向けに調整が必要な場合があります。

学べること
----------

- **基礎:**
  フェデレーテッドラーニングと分散型トレーニングの概念を理解します。
- **システムアーキテクチャ:**
  NVIDIA FLARE のシステムアーキテクチャ、デプロイ、ユーザーとのやり取りについて学びます。
- **プライバシーとセキュリティ:**
  プライバシーとセキュリティの課題、その解決策、エンタープライズグレードの保護機能を探ります。
- **高度なトピック:**
  アルゴリズム(FedOpt、FedProx など)、ワークフロー(cyclic、split、swarm)、LLM の学習、XGBoost を掘り下げます。
- **業界での応用:**
  ヘルスケア、ライフサイエンス、金融における実世界での活用を学びます。
- **実践的スキル:**
  標準的な ML コードからフェデレーテッドワークフローへ移行し、クライアント/サーバーのロジック、ジョブ構造、設定をカスタマイズします。
- **充実したリソース:**
  100 本を超えるノートブックと 88 本の動画を活用して、体系的に学習できます。

.. tip::

   各ノートブックは自己完結しており単独で実行できますが、最良の結果を得るには順番に進めて確かな基礎を築くことをおすすめします。

コースの構成
------------

パート 1: フェデレーテッドラーニング入門
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、NVIDIA FLARE を使ったフェデレーテッドラーニングを実践的に紹介します。フェデレーテッドラーニングアプリケーションの実行方法と開発方法を、初心者から経験者まで役立つ実践的なサンプルと明確なガイダンスとともに学びます。

**学べること**

- フェデレーテッドラーニングの基礎とその利点
- NVIDIA FLARE を使ったフェデレーテッドラーニングモデルの学習とデプロイ方法
- 標準的な ML コードからフェデレーテッドラーニングのワークフローへの移行
- NVIDIA FLARE におけるクライアントとサーバーのロジックのカスタマイズ
- ジョブ構造、設定、統計の理解

**第 1 章: フェデレーテッドラーニングアプリケーションの実行**

- NVIDIA FLARE を使い、PyTorch で画像分類モデルを学習する
- 標準的な PyTorch の学習コードをフェデレーテッドラーニングのコードへ変換する
- NVIDIA FLARE におけるクライアントとサーバーのロジックをカスタマイズする
- フェデレーテッドラーニングのジョブ構造と設定を理解する

.. tip::

    `第 1 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-1_federated_learning_introduction/chapter-1_running_federated_learning_applications/01.0_introduction/introduction.ipynb>`_,
    `第 1 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-1_federated_learning_introduction/chapter-1_running_federated_learning_applications/video.md>`_

**第 2 章: フェデレーテッドラーニングアプリケーションの開発**


- 画像データとテーブル形式データの両方でフェデレーテッド統計を実行する
- PyTorch Lightning や従来型の ML コードを NVIDIA FLARE でフェデレーテッドラーニングのワークフローへ変換する
- 高度なカスタマイズのために NVIDIA FLARE Client API を使用する

.. tip::

    `第 2 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-1_federated_learning_introduction/chapter-2_develop_federated_learning_applications/02.0_introduction/introduction.ipynb>`_ ,
    `第 2 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-1_federated_learning_introduction/chapter-2_develop_federated_learning_applications/video.md>`_

パート 2: フェデレーテッドラーニングシステム
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、NVIDIA FLARE を用いたフェデレーテッドコンピューティングシステムのアーキテクチャとデプロイを扱います。フェデレーテッドラーニング環境におけるシステム構築、ユーザーとのやり取り、監視ツールについて実践的な知識を得られます。

**学べること**

- NVIDIA FLARE のシステムアーキテクチャと中核となる概念
- フェデレーテッドコンピューティングシステムのセットアップとシミュレーション方法(ローカルデプロイ)
- ユーザーとのやり取りの手段: 管理コンソール、Python API、CLI
- Prometheus と Grafana によるシステムイベントの監視

**第 3 章: フェデレーテッドコンピューティングプラットフォーム**

- NVIDIA FLARE のフェデレーテッドコンピューティングプラットフォームとその構成要素を理解する
- システムのロール、通信、ワークフローについて学ぶ

.. tip::

    `第 3 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-2_federated_learning_system/chapter-3_federated_computing_platform/03.0_introduction/introduction.ipynb>`_ ,
    `第 3 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-2_federated_learning_system/chapter-3_federated_computing_platform/video.md>`_

**第 4 章: フェデレーテッドコンピューティングシステムのセットアップ**

- NVIDIA FLARE でフェデレーテッドコンピューティングシステムを構築するためのステップバイステップガイド
- デプロイをシミュレートし、さまざまなツールでシステムと対話する

.. tip::

    `第 4 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-2_federated_learning_system/chapter-4_setup_federated_system/04.0_introduction/introduction.ipynb>`_ ,
    `第 4 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-2_federated_learning_system/chapter-4_setup_federated_system/video.md>`_

パート 3: セキュリティとプライバシー
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

フェデレーテッドラーニングは、データのプライバシーを守りながら分散型のモデル学習を可能にするため、ヘルスケアや金融のような機微な領域に適しています。一方で、フェデレーテッドラーニングにはデータ漏洩、敵対的攻撃、モデルの完全性への脅威といったセキュリティおよびプライバシーのリスクも伴います。

**学べること**

- フェデレーテッドラーニングにおけるプライバシーリスクと攻撃経路
- 保護技術: 差分プライバシー、セキュアアグリゲーション、準同型暗号
- セキュリティの課題: 敵対的攻撃、不正アクセス、通信への脅威
- セキュリティの解決策: 認証、RBAC、暗号化通信、信頼の仕組み
- NVIDIA FLARE がフェデレーテッドラーニング向けに堅牢なセキュリティとプライバシーをどのように実装しているか

**第 5 章: フェデレーテッドラーニングにおけるプライバシー**

- フェデレーテッドラーニングにおけるプライバシーリスクと攻撃を理解する
- NVIDIA FLARE によるプライバシー保護技術を探る

.. tip::

    `第 5 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-3_security_and_privacy/chapter-5_Privacy_In_Federated_Learning/05.0_introduction/introduction.ipynb>`_ ,
    `第 5 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-3_security_and_privacy/chapter-5_Privacy_In_Federated_Learning/video.md>`_


**第 6 章: フェデレーテッドコンピューティングシステムにおけるセキュリティ**

- フェデレーテッドラーニングにおけるセキュリティ上の脅威と解決策について学ぶ
- NVIDIA FLARE が安全な通信、認証、アクセス制御をどのように強制しているかを確認する

.. tip::

    `第 6 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-3_security_and_privacy/chapter-6_Security_in_federated_compute_system/06.0_introduction/introduction.ipynb>`_ ,
    `第 6 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-3_security_and_privacy/chapter-6_Security_in_federated_compute_system/video.md>`_

パート 4: フェデレーテッドラーニングの高度なトピック
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、NVIDIA FLARE を用いたフェデレーテッドラーニングの高度なトピックと技術を扱います。最先端のアルゴリズム、ワークフロー、大規模言語モデル(LLM)の学習、セキュアな XGBoost、そして高レベル API と低レベル API の違いについて学びます。

**学べること**

- 高度なフェデレーテッドラーニングアルゴリズム: FedOpt、FedProx など
- ワークフロー: cyclic、split learning、swarm learning
- NVIDIA FLARE による大規模言語モデル(LLM)の学習とファインチューニング
- セキュアなフェデレーテッド XGBoost
- NVIDIA FLARE における高レベル API と低レベル API の比較

**第 7 章: フェデレーテッドラーニングのアルゴリズムとワークフロー**

- NVIDIA FLARE を用いたさまざまなフェデレーテッドラーニングアルゴリズムとワークフロー戦略を探る

.. tip::

    `第 7 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-7_algorithms_and_workflows/07.0_introduction/introduction.ipynb>`_ ,
    `第 7 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-7_algorithms_and_workflows/video.md>`_

**第 8 章: フェデレーテッド LLM の学習**

- NVIDIA FLARE を使い、フェデレーテッド環境で大規模言語モデルを学習・ファインチューニングする方法を学ぶ

.. tip::

    `第 8 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-8_federated_LLM_training/08.0_introduction/introduction.ipynb>`_ ,
    `第 8 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-8_federated_LLM_training/video.md>`_

**第 9 章: NVIDIA FLARE の低レベル API**

- NVIDIA FLARE の低レベル API が持つ強力さと柔軟性を知る

.. tip::

    `第 9 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-9_flare_low_level_apis/09.0_introduction/introduction.ipynb>`_ ,
    `第 9 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-9_flare_low_level_apis/video.md>`_

**第 10 章: フェデレーテッド XGBoost**

- NVIDIA FLARE でセキュアなフェデレーテッド XGBoost を実現するためのステップバイステップガイド

.. tip::

    `第 10 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-10_federated_XGBoost/10.0_introduction/introduction.ipynb>`_ ,
    `第 10 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-4_advanced_federated_learning/chapter-10_federated_XGBoost/video.md>`_

パート 5: 業界におけるフェデレーテッドラーニングの応用
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

このセクションでは、ヘルスケア、ライフサイエンス、金融サービスを中心に、NVIDIA FLARE が実世界の業界でどのように活用されているかを紹介します。フェデレーテッドラーニングが組織をまたいだ協働、プライバシー保護、イノベーションをどのように可能にするかを学びます。

**学べること**

- NVIDIA FLARE がヘルスケアとライフサイエンスにおける協調的な機械学習をどのように支えているか。次のような領域を含みます。

  - 医用画像解析(例: がん検出、放射線医学)
  - 生存時間解析(例: カプラン・マイヤー法)
  - ゲノミクスと複数機関にまたがる研究
  - 創薬

- 金融サービスでの応用。例:

  - 不正検知
  - 取引における異常検知

**第 11 章: ヘルスケアとライフサイエンスにおけるフェデレーテッドラーニング**

- 医学研究、診断、創薬における NVIDIA FLARE のユースケース
- 病院や研究センターをまたいで、堅牢かつプライバシーを保護するモデルを学習する方法

.. tip::

    `第 11 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-5_federated_learning_applications_in_industries/chapter-11_federated_learning_in_healthcare_lifescience/11.0_introduction/introduction.ipynb>`_ ,
    `第 11 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-5_federated_learning_applications_in_industries/chapter-11_federated_learning_in_healthcare_lifescience/video.md>`_


**第 12 章: 金融サービスにおけるフェデレーテッドラーニング**

- 不正検知、信用リスク、規制順守のための協調的なモデル学習

.. tip::

    `第 12 章のノートブック <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-5_federated_learning_applications_in_industries/chapter-12_federated_learning_in_financial_services/12.0_introduction/introduction.ipynb>`_ ,
    `第 12 章の動画 <https://github.com/NVIDIA/NVFlare/blob/main/examples/tutorials/self-paced-training/part-5_federated_learning_applications_in_industries/chapter-12_federated_learning_in_financial_services/video.md>`_


はじめかた
----------

- 興味のあるパートやトピックから始めても、順番に沿って体系的に進めても構いません。
- より深い解説やトラブルシューティングについては、公式の `NVIDIA FLARE ドキュメント <https://nvflare.readthedocs.io/>`_ を参照してください。

学習をお楽しみください。
