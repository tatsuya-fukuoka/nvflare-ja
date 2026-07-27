.. _flare_mobile:

########################
FLARE モバイル開発
########################

FLARE 2.7 では、Android と iOS の両プラットフォームに対する包括的なモバイル開発サポートが導入され、エッジデバイス上で直接連合学習を実行できるようになりました。本ガイドでは、モバイル SDK の統合、API の使い方、およびモバイルプラットフォーム上で FL アプリケーションを開発する際のベストプラクティスについて説明します。

.. note::
   本ガイドは、:ref:`エッジ開発の概念 <flare_edge>` と :ref:`階層型アーキテクチャ <flare_hierarchical_architecture>` を理解していることを前提としています。エッジシステム全体を把握するために、まずは :ref:`エッジ開発ガイド <flare_edge>` に目を通してください。

概要
========

FLARE モバイル SDK は、Android (Kotlin/Java) および iOS (Swift/Objective-C) 向けのネイティブライブラリを提供し、次のことを可能にします。

* **オンデバイストレーニング**: モバイル向けに最適化されたモデル実行のために ExecuTorch を使用します
* **連合学習との統合**: NVIDIA FLARE の階層型エッジシステムと連携します
* **リアルタイム通信**: HTTP/HTTPS 経由で FLARE サーバーと通信します
* **モデル管理**: モデルの読み込み、トレーニング、更新を行います
* **データ処理**: 柔軟なデータセットインターフェースを利用できます
* **エラー処理とリカバリ**: モバイル特有のシナリオに対応します

.. tip::
   モバイル開発をすぐに始めるには、`エッジのサンプル <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ にある完全なサンプルを参照してください。

プラットフォームのサポート
============================

Android
-------
* **最小 SDK**: API レベル 29 (Android 10)
* **ターゲット SDK**: 最新の安定版
* **言語**: Kotlin/Java
* **ビルドシステム**: Gradle
* **依存関係**: ExecuTorch、OkHttp、Gson、Coroutines

iOS
---
* **最小バージョン**: iOS 13.0
* **ターゲットバージョン**: 最新の安定版
* **言語**: Swift/Objective-C
* **ビルドシステム**: Xcode
* **依存関係**: ExecuTorch、Foundation、UIKit

アーキテクチャ
================

モバイル SDK のアーキテクチャは、``FlareRunner`` 、 ``Connection`` 、 ``DataSource`` 、 ``ETTrainer`` 、 ``Dataset`` といったモジュール化されたコンポーネントで構成されています。各コンポーネントは、オーケストレーション、通信、データ処理、モデルトレーニングなど、モバイルデバイス上の連合学習における特定の側面を担当します。詳細は以下のコンポーネント説明を参照してください。

コアコンポーネント
--------------------

**FlareRunner** (Android: ``AndroidFlareRunner`` 、 iOS: ``NVFlareRunner`` )
    ジョブの取得、タスクの実行、結果の報告を担う中心的なオーケストレーターです。

**Connection** (Android: ``Connection`` 、 iOS: ``NVFlareConnection`` )
    FLARE サーバーとの HTTP/HTTPS 通信を管理します。

**DataSource** (Android: ``DataSource`` 、 iOS: ``NVFlareDataSource`` )
    FL システムにトレーニングデータを提供するためのインターフェースです。

**ETTrainer** (Android: ``ETTrainer`` 、 iOS: ``ETTrainer`` )
    オンデバイスでのモデルトレーニングを行う ExecuTorch ベースのトレーナーです。

**Dataset** (Android: ``Dataset`` 、 iOS: ``NVFlareDataset`` )
    トレーナーにトレーニング用のサンプルを供給するデータインターフェースです。

はじめに
===============

前提条件
-------------

モバイル開発を始める前に、以下が用意されていることを確認してください。

1. **NVIDIA FLARE サーバー**: 階層型エッジ構成で稼働している FLARE サーバー ( :ref:`階層型アーキテクチャ <flare_hierarchical_architecture>` を参照)
2. **ExecuTorch**: モバイル向けに最適化された PyTorch ランタイム ( `ExecuTorch のドキュメント <https://pytorch.org/executorch/>`_ )
3. **開発環境**:
   * Android Studio (Android) - `ダウンロード <https://developer.android.com/studio>`_
   * Xcode (iOS) - Mac App Store から入手できます
4. **モデル**: ExecuTorch 形式に変換された PyTorch モデル
5. **エッジのサンプル**: ``examples/advanced/edge/`` にある動作するサンプル

.. warning::
   ExecuTorch は、モバイルプラットフォーム向けに固有のビルド構成を必要とします。対象プラットフォームについて、公式の ExecuTorch セットアップガイドに必ず従ってください。

Android のセットアップ
========================

インストール
--------------

``examples/advanced/edge/mobile/android`` 配下の Android サンプルのソースには、Gradle のビルドファイルが含まれていません。Android Studio で新しい Kotlin プロジェクトを作成するか、既存の Android アプリを使用し、そのプロジェクトに NVFlare Android SDK およびソースファイルを追加してください。

1. アプリモジュールの ``build.gradle.kts`` に **依存関係を追加** します。

.. code-block:: kotlin

   dependencies {
       // ExecuTorch dependencies
       implementation(fileTree(mapOf("dir" to "libs", "include" to listOf("*.jar", "*.aar"))))
       implementation("com.facebook.soloader:nativeloader:0.10.5")
       implementation("com.facebook.fbjni:fbjni:0.5.1")

       // Network dependencies
       implementation("com.squareup.okhttp3:okhttp:4.12.0")
       implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")

       // JSON parsing
       implementation("com.google.code.gson:gson:2.10.1")

       // Coroutines for async operations
       implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3")
   }

2. プロジェクトに **SDK をコピー** します。

.. code-block:: bash

   cp -r examples/advanced/edge/mobile/android/sdk \
         app/src/main/java/com/nvidia/nvflare/

3. ``app/libs/`` ディレクトリに **ExecuTorch のライブラリを追加** します。

基本的な使い方
----------------

.. code-block:: kotlin

   import com.nvidia.nvflare.sdk.core.AndroidFlareRunner
   import com.nvidia.nvflare.sdk.core.Connection
   import com.nvidia.nvflare.sdk.core.DataSource

   class MainActivity : AppCompatActivity() {
       private lateinit var flareRunner: AndroidFlareRunner

       override fun onCreate(savedInstanceState: Bundle?) {
           super.onCreate(savedInstanceState)

           // Create connection
           val connection = Connection(
               serverURL = "",  // Replace with your actual server URL
               allowSelfSignedCerts = true
           )

           // Create data source
           val dataSource = MyDataSource()

           // Create FlareRunner
           flareRunner = AndroidFlareRunner(
               context = this,
               connection = connection,
               jobName = "my_fl_job",
               dataSource = dataSource,
               deviceInfo = mapOf(
                   "device_id" to getDeviceId(),
                   "platform" to "android",
                   "app_version" to getAppVersion()
               ),
               userInfo = mapOf("user_id" to getUserId()),
               jobTimeout = 30.0f
           )

           // Start federated learning
           lifecycleScope.launch {
               flareRunner.run()
           }
       }
   }

iOS のセットアップ
====================

インストール
--------------

1. Xcode プロジェクトに **ExecuTorch フレームワークを追加** します。
2. プロジェクトに **NVFlareSDK をコピー** します。

.. code-block:: bash

   cp -r examples/advanced/edge/mobile/ios/NVFlareSDK YourProject/

3. Xcode プロジェクトのターゲットに **フレームワークを追加** します。

基本的な使い方
----------------

.. code-block:: swift

   import NVFlareSDK
   import UIKit

   class ViewController: UIViewController {
       private var flareRunner: NVFlareRunner?

       override func viewDidLoad() {
           super.viewDidLoad()

           // Create data source
           let dataSource = MyDataSource()

           // Create FlareRunner
           flareRunner = try? NVFlareRunner(
               jobName: "my_fl_job",
               dataSource: dataSource,
               deviceInfo: [
                   "device_id": UIDevice.current.identifierForVendor?.uuidString ?? "unknown",
                   "platform": "ios",
                   "app_version": Bundle.main.infoDictionary?["CFBundleShortVersionString"] as? String ?? "unknown"
               ],
               userInfo: [:],
               jobTimeout: 30.0,
               serverURL: "",  // Replace with your actual server URL
               allowSelfSignedCerts: true
           )

           // Start federated learning
           Task {
               await flareRunner?.run()
           }
       }
   }

API リファレンス
==================

AndroidFlareRunner
------------------

Android における連合学習の中心的なオーケストレーターです。

**コンストラクター**

.. code-block:: kotlin

   AndroidFlareRunner(
       context: AndroidContext,
       connection: Connection,
       jobName: String,
       dataSource: DataSource,
       deviceInfo: Map<String, String>,
       userInfo: Map<String, String>,
       jobTimeout: Float,
       inFilters: List<Filter>? = null,
       outFilters: List<Filter>? = null,
       resolverRegistry: Map<String, Class<*>>? = null
   )

**パラメータ**

- ``context``: Android のアプリケーションコンテキストです。
- ``connection``: サーバー通信のための Connection インスタンスです。
- ``jobName``: 参加する FL ジョブの名前です。
- ``dataSource``: トレーニングデータを提供するデータソースです。
- ``deviceInfo``: デバイスのメタデータです ( ``device_id`` 、 ``platform`` など)。
- ``userInfo``: ユーザーのメタデータです ( ``user_id`` など)。
- ``jobTimeout``: ジョブ操作のタイムアウト (秒) です。
- ``inFilters``: データ処理用の入力フィルターです (省略可)。
- ``outFilters``: 結果処理用の出力フィルターです (省略可)。
- ``resolverRegistry``: コンポーネントリゾルバーのレジストリです (省略可)。

**メソッド**

.. code-block:: kotlin

   // Start federated learning
   suspend fun run()

   // Stop federated learning
   fun stop()

   // Get current status
   fun getStatus(): String

Android SDK の API の詳細については、:ref:`mobile_android_api` を参照してください。


NVFlareRunner (iOS)
-------------------

iOS における連合学習の中心的なオーケストレーターです。

**イニシャライザー**

.. code-block:: swift

   init(
       jobName: String,
       dataSource: NVFlareDataSource,
       deviceInfo: [String: String],
       userInfo: [String: String],
       jobTimeout: TimeInterval,
       serverURL: String,
       allowSelfSignedCerts: Bool = false,
       inFilters: [NVFlareFilter]? = nil,
       outFilters: [NVFlareFilter]? = nil,
       resolverRegistry: [String: ComponentCreator.Type]? = nil
   ) throws

**パラメータ**

- ``jobName``: 参加する FL ジョブの名前です。
- ``dataSource``: トレーニングデータを提供するデータソースです。
- ``deviceInfo``: デバイスのメタデータです ( ``device_id`` 、 ``platform`` など)。
- ``userInfo``: ユーザーのメタデータです ( ``user_id`` など)。
- ``jobTimeout``: ジョブ操作のタイムアウト (秒) です。
- ``serverURL``: FLARE サーバーの URL です。
- ``allowSelfSignedCerts``: 自己署名証明書を許可するかどうかを指定します。
- ``inFilters``: データ処理用の入力フィルターです (省略可)。
- ``outFilters``: 結果処理用の出力フィルターです (省略可)。
- ``resolverRegistry``: コンポーネントリゾルバーのレジストリです (省略可)。

**メソッド**

.. code-block:: swift

   // Start federated learning
   func run() async

   // Stop federated learning
   func stop()

   // Get current status
   var status: NVFlareStatus { get }

データソース
==============

データソースの実装
-------------------------

いずれのプラットフォームでも、トレーニングデータを提供するためにデータソースのインターフェースを実装する必要があります。

**Android の DataSource インターフェース**

.. code-block:: kotlin

   interface DataSource {
       fun getDataset(jobName: String, context: Context): Dataset
   }

**iOS の NVFlareDataSource プロトコル**

.. code-block:: swift

   protocol NVFlareDataSource {
       func getDataset(for jobName: String, context: NVFlareContext) throws -> NVFlareDataset
   }

**実装例**

.. code-block:: kotlin

   class MyDataSource : DataSource {
       override fun getDataset(jobName: String, context: Context): Dataset {
           return MyDataset()
       }
   }

.. code-block:: swift

   class MyDataSource: NVFlareDataSource {
       func getDataset(for jobName: String, context: NVFlareContext) throws -> NVFlareDataset {
           return MyDataset()
       }
   }

モデル開発
=================

ExecuTorch の統合
----------------------

モバイルでの FL トレーニングでは、最適化されたモデル実行のために ExecuTorch を使用します。モデルは PyTorch から ExecuTorch 形式へ変換する必要があります。

**モデルの変換**

.. code-block:: python

   import torch
   from executorch.exir import to_edge_transform_and_lower

   # Load your PyTorch model
   model = YourPyTorchModel()
   model.eval()

   # Prepare example input
   example_input = torch.randn(1, 3, 224, 224)

   # Export the model using torch.export
   exported_program = torch.export.export(model, (example_input,))

   # Convert to ExecuTorch format using public API
   edge_program = to_edge_transform_and_lower(exported_program)

**モデルの要件**

- モデルは ExecuTorch がサポートする演算に対応している必要があります。
- 入出力の形状は変換時に固定されている必要があります。
- カスタム演算には ExecuTorch の拡張が必要になる場合があります。
- モデルの変換には公式の ExecuTorch エクスポート API を使用してください。

ベストプラクティス
====================

パフォーマンスの最適化
------------------------

1. **モデルサイズ**: モバイルの制約に合わせてモデルを軽量に保ちます。
2. **バッチサイズ**: デバイスのメモリに適したバッチサイズを使用します。
3. **トレーニング頻度**: トレーニングの頻度とバッテリー寿命のバランスを取ります。
4. **データのキャッシュ**: 頻繁に使用するデータはローカルにキャッシュします。

エラー処理
--------------

1. **ネットワークエラー**: ネットワーク障害に備えてリトライロジックを実装します。
2. **モデルエラー**: モデルの読み込みやトレーニングのエラーを適切に処理します。
3. **データエラー**: トレーニングの前にデータを検証します。
4. **タイムアウトの扱い**: 適切なタイムアウトを実装します。

セキュリティ上の考慮事項
--------------------------

1. **証明書の検証**: 本番環境では適切な証明書検証を行います。
2. **データプライバシー**: 機微なデータが安全に扱われるようにします。
3. **モデルの保護**: 機微なアプリケーションではモデルの暗号化を検討します。
4. **ネットワークセキュリティ**: サーバーとの通信にはすべて HTTPS を使用します。

トラブルシューティング
========================

よくある問題
--------------

**ビルドエラー**
* すべての依存関係が正しくリンクされていることを確認してください。
* ExecuTorch ライブラリの互換性を確認してください。
* SDK のファイルが正しくコピーされていることを確認してください。

**実行時エラー**
* ネットワーク接続を確認してください。
* サーバーの構成を確認してください。
* 具体的なエラーメッセージについてデバイスのログを確認してください。

**パフォーマンスの問題**
* トレーニング中のメモリ使用量を監視してください。
* モデルアーキテクチャを最適化してください。
* バッチサイズやトレーニングパラメータを調整してください。

サンプルとチュートリアル
==========================

完全に動作するサンプルは、NVIDIA FLARE のリポジトリで入手できます。

* **iOS のサンプルアプリ**: `iOS Example Project <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge/mobile/ios/ExampleProject>`_
* **Android のサンプルアプリ**: `Android Example Project <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge/mobile/android>`_
* **NVIDIA FLARE をエッジで実行する方法**: `Edge Examples <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ - シミュレーションと実機の両方を扱っています

.. tip::
   独自のアプリケーションを構築する前に、まずサンプルから始めて統合フロー全体を理解してください。

ヘルプの入手
==============

* **ドキュメント**: :ref:`FLARE のドキュメント <user_guide>` を参照してください。
* **サンプル**: ``examples/advanced/edge/`` にあるサンプルを確認してください。
* **問題の報告**: `NVIDIA FLARE の GitHub リポジトリ <https://github.com/NVIDIA/NVFlare>`_ で問題を報告してください。
* **コミュニティ**: NVIDIA FLARE のコミュニティディスカッションに参加してください。
* **ExecuTorch のサポート**: モバイル固有の問題については `ExecuTorch のドキュメント <https://pytorch.org/executorch/>`_ を参照してください。
