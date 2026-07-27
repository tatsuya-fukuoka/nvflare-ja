:orphan:

##############################################
NVFlare による連合 XGBoost
##############################################

XGBoost (https://github.com/dmlc/xgboost) は、勾配ブースティング(Gradient Boosting)
フレームワークに基づく機械学習アルゴリズムを実装するオープンソースプロジェクトです。
高い効率性、柔軟性、可搬性を備えるように設計された、最適化された分散勾配ブースティング
ライブラリです。
この実装では、クライアントの通信と同期に MPI(メッセージパッシングインターフェース)を
使用します。

MPI は、基盤となる通信ネットワークが完全であることを要求します。メッセージが 1 つでも
失われると、トレーニングは失敗します。

これは通常、NCCL のような高信頼の専用ネットワークによって実現されます。

オープンソースの XGBoost は連合パラダイムをサポートしており、クライアントが異なる場所に
存在し、インターネット接続上の gRPC で相互に通信します。

より信頼性の高い連合セットアップのために、NVFlare による連合 XGBoost を紹介します。

.. toctree::
   :maxdepth: 1

   secure_xgboost_user_guide
   reliable_xgboost_design
   reliable_xgboost_timeout
   secure_xgboost_design

