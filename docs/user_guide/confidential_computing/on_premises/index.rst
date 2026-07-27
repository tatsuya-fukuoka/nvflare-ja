********************************
オンプレミスにおけるIP保護
********************************

以下のドキュメントでは、IP保護のためのFLAREのConfidential Federated AIアーキテクチャについて詳しく説明します。

- :ref:`cc_architecture` - システムアーキテクチャとコンポーネント設計
- :ref:`cc_deployment_guide` - AMD SEV-SNPとNVIDIA GPUを用いたオンプレミスCVMセットアップのデプロイガイド
- :ref:`base_image_build` - Ubuntuベースイメージ、ファームウェア、および必要なバイナリのビルド手順
- :ref:`confidential_computing_attestation` - アテステーションの仕組みと信頼の確立
- :ref:`hashicorp_vault_trustee_deployment` - Trusteeを用いた実運用向けHashiCorpキーボールトのデプロイ

.. toctree::
   :maxdepth: 2

   cc_architecture
   cc_deployment_guide
   base_image_build
   attestation
   hashicorp_vault_trustee_kbs_deployment
