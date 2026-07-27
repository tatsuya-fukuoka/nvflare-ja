.. _system_configuration:

####################
システム設定
####################

NVIDIA FLARE の動作は設定ファイルによって制御されます。このセクションでは、
システムの設定可能なあらゆる側面について説明します。

.. toctree::
   :maxdepth: 1
   :hidden:

   configurations
   communication_configuration
   variable_resolution
   server_port_consolidation

設定フォーマットとファイル
================================

- :doc:`設定ファイル <configurations>` -- サポートされるフォーマット、検索順序、設定リファレンス

通信とネットワーク
========================

- :doc:`通信設定 <communication_configuration>` -- CellNet、gRPC、および接続に関する設定
- :doc:`シングルポートでのサーバーデプロイ <server_port_consolidation>` -- 簡素化されたシングルポート構成

ジョブ設定
================

- :doc:`変数解決 <variable_resolution>` -- ジョブ設定における変数の置換
