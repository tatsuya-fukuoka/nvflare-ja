.. _data_preparation:

############################################
データ準備とデータの異質性
############################################

概要
========

連合学習(Federated Learning)におけるデータ準備は、以下の理由で集中型 ML とは異なります:

- 各サイトは、中央から検査できない独自のローカルデータセットを持つ
- サイト間のデータ分布は多くの場合 **non-IID**\ (同一分布でない)である
- 生データを共有せずに特徴量スキーマを揃える必要がある
- データ品質がサイトごとに異なる

このガイドでは、これらの課題に対する実践的なアプローチを扱います。

データの異質性(Non-IID データ)
==================================================================

連合学習では、サイト間のデータは多くの場合 **non-IID**\ (独立同一分布でない)です。
サイトごとにラベル分布、特徴量分布、データセットサイズが異なる可能性があります。
この異質性は、集中型トレーニングと比較して、収束を遅くし、モデル精度を低下させることがあります。

**緩和戦略とサンプル:**

- `FedProx <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_fedprox>`_ -- 近接正則化項を追加し、ローカルトレーニング中にクライアントモデルがグローバルモデルから離れすぎるのを防ぎます
- `SCAFFOLD <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/cifar10/pt/cifar10-sim/cifar10_scaffold>`_ -- 制御変量(control variates)を使用してクライアントドリフトを補正し、異質性の高いデータでの収束を大幅に改善します

連合データ探索
============================

トレーニングの前に、**Federated Statistics** を使用して、生データを共有することなく
サイト間のデータ分布を把握します:

- :doc:`Hello Tabular Statistics </hello-world/hello-tabular-stats/index>` -- 連合テーブルデータ全体で統計量(平均、標準偏差、ヒストグラム)を計算します
- `Federated Image Statistics <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/federated-statistics/image_stats>`_ -- サイト間で画像のヒストグラム統計を計算します

垂直連合学習のためのユーザーアラインメント
==========================================================================

垂直連合学習では、重複するユーザーについて、異なるサイトが異なる特徴量を保持します。
トレーニングの前に、各サイトは自身の完全なデータセットを明かすことなく、共通のユーザーを特定する必要があります。

- `Private Set Intersection (PSI) <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/psi>`_ -- 垂直連合学習のためのユーザーアラインメント。プライベートデータを公開することなく、サイト間で共通のユーザー/エンティティを見つけます
