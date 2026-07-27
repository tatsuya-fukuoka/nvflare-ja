:orphan:

.. _edge_mobile_overview:

##########################
エッジとモバイル
##########################

FLARE は、連合学習をデータセンターの外側にあるエッジデバイスやモバイルプラットフォームへと拡張します。

.. toctree::
   :maxdepth: 1
   :hidden:

   mobile_training

モバイル連合トレーニング
==========================

`ExecuTorch <https://github.com/pytorch/executorch>`_ を利用して iOS および Android のデバイス上でトレーニングを行います。
**デバイス側のプログラミングは不要です** -- 標準的な PyTorch でモデルを開発し、
``ETFedBuffRecipe`` を使うだけで、エクスポート、デプロイ、連合学習のオーケストレーションが処理されます。

- :doc:`モバイル連合トレーニング (iOS / Android) <mobile_training>` -- DeviceModel、ETFedBuffRecipe、およびモバイル SDK ガイド

階層型 FLARE
==============

階層的な集約とリレーベースの通信によって数千台規模のデバイスへスケールさせる方法については、
:ref:`階層型 FLARE <flare_hierarchical_architecture>` および
:ref:`階層型通信 <hierarchical_communication>` を参照してください。
