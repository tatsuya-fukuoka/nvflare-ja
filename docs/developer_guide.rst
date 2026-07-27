.. _developer_guide:
.. _programming_guide:

##############################################################
アーキテクチャと開発者ガイド
##############################################################

このガイドは、FLARE の内部を理解し、カスタムワークフローを構築し、プラットフォームを拡張する必要のある開発者向けです。より高レベルな使い方については、:ref:`ユーザーガイド <user_guide>` を参照してください。

システムアーキテクチャ
============================================

- :doc:`システムアーキテクチャ概要 <programming_guide/system_architecture>`
- :doc:`FLARE システムアーキテクチャ <flare_system_architecture>`
- :doc:`CellNet アーキテクチャ <system_architecture/cellnet_architecture>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/system_architecture
   flare_system_architecture
   system_architecture/cellnet_architecture

コアコンセプト
============================

- :doc:`ジョブ <user_guide/core_concepts/job>`
- :doc:`ワークスペース <user_guide/core_concepts/workspace>`
- :doc:`アプリケーション <user_guide/core_concepts/application>`
- :doc:`FLModel <programming_guide/fl_model>`
- :doc:`FLContext <programming_guide/fl_context>`
- :doc:`FLComponent <programming_guide/fl_component>`
- :doc:`イベントシステム <programming_guide/event_system>`
- :doc:`FedJob API <programming_guide/fed_job_api>`
- :doc:`FL シミュレーター <user_guide/nvflare_cli/fl_simulator>`
- :doc:`POC <user_guide/data_scientist_guide/poc>`

.. toctree::
   :maxdepth: 1
   :hidden:

   user_guide/core_concepts/job
   user_guide/core_concepts/workspace
   user_guide/core_concepts/application
   programming_guide/fl_model
   programming_guide/fl_context
   programming_guide/fl_component
   programming_guide/event_system
   programming_guide/fed_job_api
   user_guide/nvflare_cli/fl_simulator
   user_guide/data_scientist_guide/poc

ワークフローとコントローラー
========================================================

- :doc:`ワークフローとコントローラー <programming_guide/workflows_and_controllers>`
- :doc:`Model Controller <programming_guide/controllers/model_controller>`
- :doc:`Scatter and Gather <programming_guide/controllers/scatter_and_gather_workflow>`
- :doc:`Cyclic ワークフロー <programming_guide/controllers/cyclic_workflow>`
- :doc:`クライアント制御ワークフロー <programming_guide/controllers/client_controlled_workflows>`
- :doc:`クロスサイトモデル評価 <programming_guide/controllers/cross_site_model_evaluation>`
- :doc:`グローバル重みの初期化 <programming_guide/controllers/initialize_global_weights>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/workflows_and_controllers
   programming_guide/controllers/model_controller
   programming_guide/controllers/scatter_and_gather_workflow
   programming_guide/controllers/cyclic_workflow
   programming_guide/controllers/client_controlled_workflows
   programming_guide/controllers/cross_site_model_evaluation
   programming_guide/controllers/initialize_global_weights

高度なトピック
============================

- :doc:`フィルター <programming_guide/filters>`
- :doc:`コンポーネント設定 <programming_guide/component_configuration>`
- :doc:`Resource Manager と Consumer <programming_guide/resource_manager_and_consumer>`
- :doc:`グローバルモデルの初期化 <programming_guide/global_model_initialization>`
- :doc:`タイムアウトリファレンス <programming_guide/timeouts>`
- :doc:`Dashboard API <programming_guide/dashboard_api>`
- :doc:`安全でないコンポーネントの検出 <user_guide/admin_guide/security/unsafe_component_detection>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/filters
   programming_guide/component_configuration
   programming_guide/resource_manager_and_consumer
   programming_guide/global_model_initialization
   programming_guide/timeouts
   programming_guide/dashboard_api
   user_guide/admin_guide/security/unsafe_component_detection

大規模モデルと LLM
====================================

LLM を含む大規模モデルの連合学習トレーニングとファインチューニングのための技術です。

**デプロイメントと最適化:**

- :doc:`大規模モデルに関する注意事項 <user_guide/admin_guide/deployment/notes_on_large_models>` -- 大規模モデルトレーニングにおけるデプロイメント上の考慮事項
- :doc:`メッセージ量子化 <programming_guide/message_quantization>` -- 量子化によるメッセージサイズの削減
- :doc:`ファイルストリーミング <programming_guide/file_streaming>` -- 参加者間での大きなファイルのストリーミング
- :doc:`Tensor Downloader <programming_guide/tensor_downloader>` -- 効率的なモデルパラメーターの転送
- :doc:`メモリ管理 <programming_guide/memory_management>` -- トレーニング中のメモリ使用量の制御
- :doc:`大きなオブジェクト向けの Decomposer <programming_guide/decomposer_for_large_object>` -- 大きなオブジェクトの効率的なシリアライズ

**LLM ファインチューニング:**

- `Federated SFT (Supervised Fine-Tuning) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/llm_hf>`_ -- HuggingFace による連合 SFT
- `Federated PEFT (Parameter-Efficient Fine-Tuning) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/llm_hf>`_ -- LoRA およびその他の PEFT 手法
- `NeMo SFT Integration <https://github.com/NVIDIA/NVFlare/tree/main/integration/nemo/examples/supervised_fine_tuning>`_ -- NeMo による連合 SFT
- `NeMo PEFT Integration <https://github.com/NVIDIA/NVFlare/tree/main/integration/nemo/examples/peft>`_ -- NeMo による連合 PEFT

.. toctree::
   :maxdepth: 1
   :hidden:

   user_guide/admin_guide/deployment/notes_on_large_models
   programming_guide/message_quantization
   programming_guide/memory_management
   programming_guide/tensor_downloader
   programming_guide/file_streaming
   programming_guide/decomposer_for_large_object

階層型アーキテクチャ
========================================

- :doc:`階層型アーキテクチャ <programming_guide/hierarchical_architecture>`
- :doc:`階層型通信 <programming_guide/hierarchical_communication>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/hierarchical_architecture
   programming_guide/hierarchical_communication

サードパーティ統合
====================================

- :doc:`サードパーティ統合 <programming_guide/execution_api_type/3rd_party_integration>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/execution_api_type/3rd_party_integration

低レベル API
========================

これらは、より高レベルな抽象化(Client API、FLARE API)の基盤となる API です。ほとんどのユーザーがこれらを直接使う必要はありませんが、高度なカスタマイズのために利用できます。

- :doc:`Executor <programming_guide/execution_api_type/executor>`
- :doc:`Shareable <programming_guide/shareable>`
- :doc:`Data Exchange Object <programming_guide/data_exchange_object>`
- :doc:`コントローラー <programming_guide/controllers/controllers>`
- :doc:`実行 API タイプ <programming_guide/execution_api_type>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/execution_api_type/executor
   programming_guide/shareable
   programming_guide/data_exchange_object
   programming_guide/controllers/controllers
   programming_guide/execution_api_type

テスト
============

- :doc:`開発者向けテスト <programming_guide/developer_testing>`
- :doc:`ノートブックのテスト <programming_guide/notebook_testing>`

.. toctree::
   :maxdepth: 1
   :hidden:

   programming_guide/developer_testing
   programming_guide/notebook_testing

トラブルシューティング
============================================

- :doc:`タイムアウトのトラブルシューティング <user_guide/timeout_troubleshooting>`
- :doc:`FAQ <faq>`

.. toctree::
   :maxdepth: 1
   :hidden:

   user_guide/timeout_troubleshooting
   faq
