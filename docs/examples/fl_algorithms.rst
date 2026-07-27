.. _fl_algorithms:

********************
FLアルゴリズム
********************

連合平均化(Federated Averaging)
------------------------------------
NVIDIA FLARE では、FedAvg は :ref:`scatter_and_gather_workflow` を通じて実装されています。連合平均化ワークフローでは、
初期重みのセットがクライアントワーカーに配布され、クライアントワーカーがローカルトレーニングを実行します。ローカルトレーニングの後、クライアントは
ローカルの重みを Shareable として返し、それらが集約(平均化)されます。この新しいグローバル平均重みのセットが
再びクライアントに配布され、指定されたラウンド数だけこのプロセスが繰り返されます。

FedProx
-------
`FedProx <https://arxiv.org/abs/1812.06127>`_ は、グローバルモデルからの乖離に基づいてクライアントのローカル重みにペナルティを課す
:class:`Loss function <nvflare.app_common.pt.pt_fedproxloss.PTFedProxLoss>` を実装しています。設定例は
:github_nvflare_link:`CIFAR-10 example <examples/advanced/cifar10>` の cifar10_fedprox にあります。

FedOpt
------
`FedOpt <https://arxiv.org/abs/2003.00295>`_ は、グローバルモデルの更新時に指定した Optimizer と Learning Rate Scheduler を使用できる
:class:`ShareableGenerator <nvflare.app_common.pt.pt_fedopt.PTFedOptModelShareableGenerator>` を実装しています。設定例は
:github_nvflare_link:`CIFAR-10 example <examples/advanced/cifar10>` の cifar10_fedopt にあります。

SCAFFOLD
--------
`SCAFFOLD <https://arxiv.org/abs/1910.06378>`_ は、CIFAR-10 Learner 実装をわずかに変更したバージョン、
すなわち `CIFAR10ScaffoldLearner` を使用します。これは、`Li et al. <https://arxiv.org/abs/2102.02079>`_ に記載されている
`implementation <https://github.com/Xtra-Computing/NIID-Bench>`_ に従い、ローカルトレーニング中に補正項を追加するものです。設定例は :github_nvflare_link:`CIFAR-10 example <examples/advanced/cifar10>` の cifar10_scaffold にあります。

連合XGBoost
-----------------

NVFlare は、人気のある勾配ブースティングライブラリ XGBoost を使用した連合学習をサポートしています。
学習には、federated プラグイン付きの XGBoost ライブラリ(xgboost バージョン >= 1.7.0rc1)を使用します。

連合XGBoost を直接実行する場合と比較して、NVFlare で XGBoost を使用すると次の利点があります。

* XGBoost インスタンスのライフサイクルが NVFlare によって管理されます。XGBoost のクライアントとサーバーの両方が
  NVFlare のワークフローによって自動的に起動/停止されます。
* ヒストグラムベースの XGBoost では、federated サーバーを自動割り当てのポート番号で自動的に設定できます。
* 相互 TLS を使用する場合、証明書は既存のプロビジョニングプロセスを使用して NVFlare によって
  管理されます。
* 各インスタンスを手動で設定する必要はありません。code:`rank` のようなインスタンス固有のパラメータは、
  NVFlare のコントローラーによって自動的に割り当てられます。

* :github_nvflare_link:`Federated Horizontal XGBoost (GitHub) <examples/advanced/xgboost>` - ヒストグラムベースおよびツリーベースのアルゴリズムの例が含まれます。ツリーベースのアルゴリズムには、バギングとサイクリックのアプローチも含まれます。
* :github_nvflare_link:`Federated Vertical XGBoost (GitHub) <examples/advanced/vertical_xgboost>` - Private Set Intersection と XGBoost を垂直分割された HIGGS データに使用する例です。

連合分析
-------------------

* :github_nvflare_link:`Federated Statistics for medical imaging (Github) <examples/advanced/federated-statistics/image_stats/README.md>` - ローカル画像ヒストグラムを収集してグローバルデータセットのヒストグラムを計算する例です。
* :github_nvflare_link:`Federated Statistics for tabular data with DataFrame (Github) <examples/advanced/federated-statistics/df_stats/README.md>` - Pandas DataFrame からローカル統計サマリーを収集してグローバルデータセットの統計を計算する例です。
* :github_nvflare_link:`Federated Statistics with Monai Statistics integration for Spleen CT Image (Github) <integration/monai/examples/spleen_ct_segmentation_local/README.md>` - Monai 統計インテグレーションと、連合統計のその他いくつかの機能を示す例です。
