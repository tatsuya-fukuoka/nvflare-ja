.. _notebook_testing:

##############################
ノートブックのテスト
##############################

NVIDIA FLARE は Jupyter ノートブックのテストに `nbmake <https://github.com/treebeardtech/nbmake>`__ を使用しています。
これにより、コードベースが進化してもサンプルノートブックが動作し続けることを保証します。

.. note::

   **すべてのノートブックが自動テストに対応しているわけではありません。** 一部のノートブックは外部の
   インフラ(稼働中の FLARE サーバー、プロビジョニング済み環境、特定のデータセット)を必要とするか、CI で実行できない
   対話的な要素を含んでいます。ノートブックのテストカバレッジは今後改善していく予定です。

``runtest.sh`` の一般的な使い方(依存関係のキャッシュ、詳細出力モードなど)については :ref:`developer_testing` を参照してください。

.. contents:: 目次
   :local:
   :depth: 2

クイックスタート
================

ノートブックのテストを実行するには ``runtest.sh`` スクリプトを使用します:

.. code:: bash

   # Test default notebook (flare_simulator.ipynb)
   ./runtest.sh -n

   # Test a specific notebook
   ./runtest.sh -n examples/tutorials/flare_simulator.ipynb

   # Test with verbose output
   ./runtest.sh -n -v examples/tutorials/flare_simulator.ipynb

ノートブック固有のオプション
=============================

以下のオプションはノートブックのテスト(``-n``)に固有のものです:

.. list-table::
   :widths: 25 15 60
   :header-rows: 1

   * - 引数
     - デフォルト
     - 説明
   * - ``--timeout=SECONDS``
     - 1200
     - 各ノートブックの実行に対するタイムアウト(秒)
   * - ``--nb-clean=MODE``
     - on-success
     - 出力をクリアするタイミング: ``always``、``on-success``、``never``
   * - ``--kernel=NAME``
     - python3
     - Jupyter カーネル名(利用可能であればデフォルトは ``python3``)
   * - ``-v`` / ``--verbose``
     - off
     - pytest に ``-v`` を渡して詳細な出力を得ます

例
--------

.. code:: bash

   # Set a shorter timeout (5 minutes)
   ./runtest.sh -n --timeout=300 examples/tutorials/flare_simulator.ipynb

   # Use a specific kernel
   ./runtest.sh -n --kernel=python3 examples/tutorials/flare_simulator.ipynb

   # Always clean outputs regardless of pass/fail
   ./runtest.sh -n --nb-clean=always examples/tutorials/

   # Combine multiple options with verbose output
   ./runtest.sh -n -v --timeout=1800 --kernel=python3 examples/tutorials/

pytest を直接使用する
=======================

nbmake を pytest で直接実行することもできます:

.. code:: bash

   pytest --nbmake --nbmake-timeout=1200 --nbmake-clean=on-success examples/tutorials/

   # With specific kernel
   pytest --nbmake --nbmake-timeout=1200 --kernel=python3 examples/tutorials/

ノートブック内のセルをスキップする
====================================

自動テスト中に特定のセルをスキップするには(例: Colab のセットアップセル、対話的な
可視化、ユーザー入力が必要なセルなど)、セルのメタデータに以下のいずれかのタグを追加します:

- ``skip-execution``
- ``skip``
- ``colab``

Jupyter でタグを追加する
--------------------------

**Jupyter Lab の場合:**

1. スキップしたいセルを選択します
2. 右サイドバーの歯車アイコンをクリックします(または View → Right Sidebar → Show Property Inspector)
3. "Common Tools" → "Cell Tags" で ``skip-execution`` を追加します

**Jupyter Notebook (クラシック) の場合:**

1. セルを選択します
2. View → Cell Toolbar → Tags
3. タグを追加します: ``skip-execution``

**VS Code の場合:**

1. セルをクリックします
2. セルの "..." メニューをクリックします
3. "Add Cell Tag" を選択します
4. ``skip-execution`` を入力します

仕組み
============

テストフレームワーク(``conftest.py`` に実装)は自動的に以下を行います:

1. **テスト前**: 元のノートブックの ``.backup`` を作成します
2. **セルのフィルタリング**: ``skip-execution``、``skip``、``colab`` のタグが付いたセルを除去します
3. **カーネルの更新**: 指定または検出されたカーネルに合わせてカーネルスペックを調整します
4. **実行**: nbmake が Jupyter カーネルを通じてノートブックを実行します
5. **復元**: 元のノートブックをバックアップから復元します
6. **出力のクリア**: ``--nbmake-clean`` の設定に基づいてセルの出力をクリアします

これにより次のことが保証されます:

- テストがコミット済みのノートブックを変更しないこと(元のノートブックはバックアップから復元されます)
- ノートブックに Colab 固有のセルや対話的なセルが含まれていても CI が壊れないこと
- 異なる開発環境間で一貫したカーネルが使用されること

トラブルシューティング
========================

カーネルが見つからない
------------------------

"Kernel not found" エラーが表示される場合:

1. 仮想環境が有効になっていることを確認します
2. ipykernel をインストールします: ``pip install ipykernel``
3. カーネルを登録します: ``python -m ipykernel install --user --name=my_env``
4. あるいは既存のカーネルを指定します: ``./runtest.sh -n --kernel=python3``

タイムアウトエラー
--------------------

実行時間の長いノートブックの場合は、タイムアウトを延ばします:

.. code:: bash

   ./runtest.sh -n --timeout=3600 examples/advanced/

外部インフラを必要とするノートブック
--------------------------------------------

一部のノートブック(例: ``flare_api.ipynb``)は、稼働中の FLARE サーバーやプロビジョニング済みの
環境を必要とします。これらのノートブックは、インフラが用意されていない限り自動テストで失敗します。

そのようなノートブックについては、以下を検討してください:

1. 対話的な環境で手動で実行する
2. 外部サービスを必要とするセルに ``skip-execution`` タグを追加する
3. 自動テスト用に簡略化したバージョンを作成する

ベストプラクティス
====================

1. **セルに適切にタグを付ける**: Colab のセットアップ、対話的なウィジェット、ユーザー入力のセルには ``skip-execution`` を付けます

2. **ノートブックを絞り込んだ内容に保つ**: 小さなノートブックほどテストが速く、デバッグも容易です

3. **妥当なタイムアウトを使用する**: 想定される実行時間にバッファを加えて ``--timeout`` を増やします

4. **プッシュ前にローカルでテストする**: コミット前にノートブックに対して ``./runtest.sh -n`` を実行します

5. **コミット前に出力をクリアする**: ``--nb-clean=always`` を使うか手動で出力をクリアし、git の差分をきれいに保ちます

6. **自己完結したサンプルを使用する**: シミュレータを使うノートブック(``flare_simulator.ipynb`` など)は、外部サーバーを必要とするものよりテストが容易です

