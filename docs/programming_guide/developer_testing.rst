.. _developer_testing:

########################
開発者向けテスト
########################

このガイドでは、NVIDIA FLARE 開発時にテストを実行するための ``runtest.sh`` スクリプトについて説明します。

.. contents:: 目次
   :local:
   :depth: 2

クイックスタート
================

テストスイート全体を実行します:

.. code:: bash

   ./runtest.sh

**デフォルトの動作**\ (引数なし): CI と同等の完全なテストスイートを実行します:

1. ライセンスヘッダーのチェック
2. コードスタイルのチェック(black、isort、flake8)
3. フォーマットの自動修正
4. カバレッジレポート付きのユニットテスト

個別コマンド
------------

特定のテストを個別に実行します:

.. code:: bash

   ./runtest.sh -u    # Unit tests only (faster, no style checks)
   ./runtest.sh -s    # Check code formatting only
   ./runtest.sh -f    # Fix code formatting only
   ./runtest.sh -n    # Notebook tests
   ./runtest.sh -l    # License header check only

.. note::

   注記: ``./runtest.sh`` は\ **完全なスイート**\ (ライセンス + スタイル + ユニットテスト)を実行します。
   ユニットテストだけが必要な場合は、\ **より速いイテレーション**\ のために ``./runtest.sh -u`` を使用してください。

利用可能なコマンド
==================

.. list-table::
   :widths: 20 80
   :header-rows: 1

   * - オプション
     - 説明
   * - ``-u`` / ``--unit-tests``
     - pytest でユニットテストを実行します
   * - ``-s`` / ``--check-format``
     - コードフォーマットをチェックします(black、isort、flake8)
   * - ``-f`` / ``--fix-format``
     - コードフォーマットの問題を自動修正します
   * - ``-n`` / ``--notebook``
     - nbmake を使ってノートブックテストを実行します(:ref:`notebook_testing` を参照)
   * - ``-l`` / ``--check-license``
     - ソースファイルのライセンスヘッダーをチェックします
   * - ``-c`` / ``--coverage``
     - カバレッジレポートを有効にします(``-u`` と併用)
   * - ``-r`` / ``--test-report``
     - JUnit XML テストレポートを生成します(``-u`` と併用)
   * - ``--clean``
     - ビルド成果物をクリーンアップします

共通オプション
==============

これらのオプションは、どのテストコマンドとも組み合わせて使用できます:

.. list-table::
   :widths: 25 15 60
   :header-rows: 1

   * - オプション
     - デフォルト
     - 説明
   * - ``--numprocesses=<N|auto>``
     - 8
     - 並列 pytest ワーカーの数(デフォルト: 8)。利用可能な CPU コア数に合わせるには ``auto`` を使用します。
   * - ``-d`` / ``--dry-run``
     - オフ
     - 実行せずにコマンドを表示します

例
==

ユニットテスト
--------------

.. code:: bash

   # Run all unit tests
   ./runtest.sh -u

   # Run specific test file or directory
   ./runtest.sh -u tests/unit_test/fuel/

   # Run with coverage report
   ./runtest.sh -u -c

   # Run with coverage and test report
   ./runtest.sh -u -c -r

   # Limit parallelism (e.g. for CI)
   ./runtest.sh -u --numprocesses=4

コード品質
----------

.. code:: bash

   # Check formatting (doesn't modify files)
   ./runtest.sh -s

   # Auto-fix formatting issues
   ./runtest.sh -f

   # Check specific directory
   ./runtest.sh -s nvflare/apis/

ノートブックテスト
------------------

ノートブックテストの詳細なオプションについては :ref:`notebook_testing` を参照してください。

.. code:: bash

   # Test default notebook
   ./runtest.sh -n

   # Test specific notebook with verbose output
   ./runtest.sh -n -v examples/tutorials/flare_simulator.ipynb

トラブルシューティング
======================

クリーンな状態に戻す
--------------------

まっさらな状態から始めるには:

.. code:: bash

   ./runtest.sh --clean
   ./runtest.sh -u  # Will reinstall dependencies
