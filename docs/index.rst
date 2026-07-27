.. _user_guide:

############
NVIDIA FLARE
############

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: 概要

   welcome
   roadmap
   release_notes/flare_280
   industry_use_cases

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: はじめに

   installation
   quickstart
   migration_guide

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: ユーザーガイド

   user_guide/data_scientist_guide/client_api_usage
   HuggingFace Client API <user_guide/data_scientist_guide/hf_client_api>
   user_guide/data_scientist_guide/job_recipe
   user_guide/data_scientist_guide/recipe_api
   user_guide/data_scientist_guide/available_recipes
   user_guide/data_scientist_guide/flare_api
   APIの進化と推奨事項 <programming_guide/flare_api_evolution>
   user_guide/data_scientist_guide/flower_integration/flower_integration
   programming_guide/experiment_tracking
   連合XGBoost <user_guide/data_scientist_guide/federated_xgboost/federated_xgboost>
   user_guide/data_scientist_guide/data_preparation
   Agent Skills <user_guide/agent_skills/index>
   CLIツール <user_guide/nvflare_cli/nvflare_cli>

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: サンプルとチュートリアル

   example_applications_algorithms
   tutorials
   self-paced-training/index
   研究論文 <user_guide/researcher_guide/index>

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: 大規模モデルとLLM

   連合LLMファインチューニング <programming_guide/llm_fine_tuning>
   programming_guide/message_quantization
   programming_guide/memory_management
   programming_guide/tensor_downloader
   programming_guide/file_streaming
   programming_guide/decomposer_for_large_object

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: エッジとモバイル

   モバイルトレーニング (iOS / Android) <user_guide/edge_development/mobile_training>
   モバイルSDKリファレンス <user_guide/edge_development/flare_mobile>
   階層型FL <programming_guide/hierarchical_architecture>
   programming_guide/hierarchical_communication

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: デプロイメントと運用

   user_guide/admin_guide/deployment/overview
   programming_guide/provisioning_system
   分散プロビジョニング <user_guide/nvflare_cli/distributed_provisioning>
   user_guide/admin_guide/deployment/dashboard_ui
   user_guide/admin_guide/deployment/cloud_deployment
   Deploy Prepare <user_guide/nvflare_cli/deploy_command>
   DockerでのFLARE実行 <user_guide/admin_guide/deployment/containerized_deployment>
   KubernetesでのFLARE実行 <user_guide/admin_guide/deployment/helm_chart>
   SlurmでのFLARE実行 <user_guide/admin_guide/deployment/slurm_job_launcher>
   OpenShiftへのFLAREデプロイ <user_guide/admin_guide/deployment/openshift>
   Brevスクリプトによるデプロイメントクイックスタート <user_guide/admin_guide/deployment/brev_scripted_deployment>
   Brev Kubernetes Helmデプロイメント <user_guide/admin_guide/deployment/brev_deployment>
   プリフライトチェック <user_guide/nvflare_cli/preflight_check>
   user_guide/admin_guide/deployment/operation
   user_guide/admin_guide/monitoring
   user_guide/admin_guide/configurations/logging_configuration
   ライブログストリーミング <programming_guide/live_log_streaming>
   サイト構成メタデータ <user_guide/admin_guide/configurations/site_config>
   システム構成 <user_guide/admin_guide/configurations/system_configuration>

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: セキュリティとコンプライアンス

   system_architecture/security_overview
   user_guide/admin_guide/security/terminologies_and_roles
   アイデンティティとアクセス制御 <user_guide/admin_guide/security/identity_security>
   user_guide/admin_guide/security/site_policy_management
   ネットワークと通信 <user_guide/admin_guide/security/communication_security>
   データプライバシーとフィルター <user_guide/admin_guide/security/data_privacy_protection>
   差分プライバシー <user_guide/admin_guide/security/differential_privacy>
   user_guide/admin_guide/security/auditing
   コンフィデンシャルコンピューティング <user_guide/confidential_computing/index>
   security_faq

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: 開発者ガイド

   developer_guide

.. toctree::
   :maxdepth: 1
   :hidden:
   :caption: リファレンス

   APIリファレンス <apidocs/modules>
   glossary
   publications_and_talks
   release_notes/previous
   contributing

NVIDIA FLARE (Federated Learning Application Runtime Environment) は、連合学習(Federated Learning)のための
オープンソースSDKです。MLの実務者が既存のトレーニングワークフロー(PyTorch、
TensorFlow、XGBoost、scikit-learn、NeMo)を最小限のコード変更で連合学習の環境に適応させることを支援し、
プラットフォームチームが安全でプライバシーを保護する多者間コラボレーションをデプロイできるようにします。

進め方を選ぶ
========================

FLAREが初めての方(MLの実務者)
--------------------------------------------------------------

既存のトレーニングスクリプトを連合学習化したい場合は、ここから始めてください。

- :doc:`ようこそ <welcome>` -- FLAREとは何か、何をサポートしているか
- :doc:`インストール <installation>` -- FLAREのインストールと環境のセットアップ
- :doc:`クイックスタート <quickstart>` -- Hello Worldサンプルの実行とMLコードの変換
- :ref:`Client API <client_api>` -- 連合学習トレーニングに推奨される高レベルAPI
- :ref:`Job Recipe API <job_recipe>` -- 一般的なFLワークフロー向けの事前構築済みレシピ
- :doc:`移行ガイド <migration_guide>` -- FLAREバージョン間のアップグレード
- :ref:`サンプルとチュートリアル <example_applications>` -- エンドツーエンドのサンプルとチュートリアル

デプロイメントとセキュリティ(本番環境チーム)
--------------------------------------------------------------------------------------------

組織やコンソーシアムでFLAREをデプロイする場合は、ここから始めてください。

- :doc:`デプロイメント概要 <user_guide/admin_guide/deployment/overview>` -- プロビジョニング、Docker/Kubernetes、クラウドデプロイメント、ダッシュボード
- :doc:`Adminコマンド <user_guide/admin_guide/deployment/operation>` -- 稼働中のFLシステムの運用と管理
- :doc:`システム構成 <user_guide/admin_guide/configurations/system_configuration>` -- 構成ファイルと設定
- :doc:`プリフライトチェック <user_guide/nvflare_cli/preflight_check>` -- 起動前の検証
- :doc:`セキュリティ概要 <system_architecture/security_overview>` -- 認証、認可、プライバシー、監査
- :doc:`コンフィデンシャルコンピューティング <user_guide/confidential_computing/index>` -- エンドツーエンドのIP保護のためのハードウェアベースのTEE

開発者(上級者・コントリビューター)
------------------------------------------------------------------------

FLAREを拡張したい、またはカスタムワークフローを構築したい場合は、ここから始めてください。

- :ref:`開発者ガイド <developer_guide>` -- アーキテクチャの詳細解説、コントローラー、フィルター、拡張ポイント
- :doc:`APIリファレンス <apidocs/modules>` -- Python APIの完全なドキュメント
- :doc:`コントリビューション <contributing>` -- NVIDIA FLAREへの貢献方法

ユースケースから探す
======================================

- :doc:`業界ユースケース <industry_use_cases>` -- ヘルスケア、金融、政府機関などにおける実世界のデプロイメント
- :ref:`大規模モデルとLLM <llm_fine_tuning>` -- 大規模モデル向けの連合ファインチューニング、メモリ管理、最適化
- :ref:`エッジとモバイル <mobile_training>` -- モバイルトレーニング(iOS/Android)と大規模デプロイメント向けの階層型FL
