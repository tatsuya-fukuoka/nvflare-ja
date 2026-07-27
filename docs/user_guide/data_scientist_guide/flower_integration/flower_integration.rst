####################################################
NVIDIA FLARE で Flower を実行する
####################################################

**既存の Flower アプリケーションを、コード変更なしで FLARE で実行できます。**

`Flower <https://flower.ai>`_ は、連合学習(Federated Learning)、分析、評価への
統一的なアプローチを実装するオープンソースプロジェクトです。Flower は FL アプリケーション開発のための
多数の戦略とアルゴリズムを開発しており、活発な FL 研究コミュニティを有しています。

一方、FLARE は、FL アプリケーションのためのエンタープライズ対応で堅牢なランタイム環境の
提供に注力してきました。

Flower と FLARE の統合により、Flower フレームワークで開発されたアプリケーションは、
一切の変更を加えることなく FLARE ランタイムで動作します。ユーザーがすべきことは、
Flower アプリケーションを FLARE ジョブとして構成し、そのジョブを FLARE システムに提出するだけです。


.. toctree::
   :maxdepth: 1

   flower_initial_integration
   flower_job_structure
   flower_run_as_flare_job
   flare_multi_job_architecture
   flower_detailed_design
   flower_reliable_messaging
