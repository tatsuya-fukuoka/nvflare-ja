.. _hello_xgboost:

#####################
Hello XGBoost
#####################

概要
========

この例では、NVIDIA FLARE を使用してフェデレーテッド XGBoost 学習を実行する方法を示します。
XGBoost は最も広く使われている勾配ブースティングフレームワークの 1 つであり、不正検知、
信用スコアリング、ヘルスケアなどの表形式データのアプリケーションで広く利用されています。

NVIDIA FLARE は複数の XGBoost フェデレーションモードをサポートしています。

- **水平分割（行分割）** -- 各サイトが同じ特徴量を持つ異なるサンプルを保有します
- **垂直分割（列分割）** -- 各サイトが同じサンプルに対する異なる特徴量を保有します
- **ヒストグラムベース** -- 木の構築のためのフェデレーテッドなヒストグラム集約

XGBoost の包括的なガイドについては :doc:`/user_guide/data_scientist_guide/federated_xgboost/federated_xgboost` を参照してください。

サンプル
==========

- `Federated XGBoost (horizontal) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb>`_ -- ヒストグラムベースの集約を用いた標準的なフェデレーテッド XGBoost
- `Secure Federated XGBoost <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/xgboost/fedxgb_secure>`_ -- セキュアな集約のための暗号化を備えた XGBoost
