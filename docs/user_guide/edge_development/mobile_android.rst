.. _mobile_android_api:

##############################
Android SDK API リファレンス
##############################

本ドキュメントでは、ExecuTorch を用いて Android デバイス上で連合学習を実行するための NVIDIA FLARE Android SDK について、包括的な API リファレンスを提供します。

.. note::
   本 API リファレンスは、:ref:`モバイル開発ガイド <flare_mobile>` および Android 開発の基本的な概念を理解していることを前提としています。

概要
========

Android SDK は、Android デバイス上で連合学習を実装するためのネイティブな Kotlin/Java ライブラリを提供します。この SDK は、FLARE サーバーとの通信、ExecuTorch を用いたモデルトレーニング、およびデータ管理を担当します。

主要なコンポーネント
======================

* **AndroidFlareRunner**: 連合学習の中心的なオーケストレーターです。
* **Connection**: FLARE サーバーとの HTTP/HTTPS 通信を行います。
* **ETTrainer**: ExecuTorch ベースのモデルトレーニングを行います。
* **DataSource**: トレーニングデータを提供するためのインターフェースです。
* **Dataset**: トレーニング用サンプルのデータインターフェースです。

AndroidFlareRunner
==================

Android デバイス上での連合学習における中心的なオーケストレーターです。ジョブの取得、タスクの実行、結果の報告、コンポーネントの解決、フィルタリング、イベント処理を担当します。

コンストラクター
------------------

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

パラメータ
~~~~~~~~~~~~

* ``context``: Android のアプリケーションコンテキストです。
* ``connection``: サーバー通信のための Connection インスタンスです。
* ``jobName``: 参加する FL ジョブの名前です。
* ``dataSource``: トレーニングデータを提供するデータソースです。
* ``deviceInfo``: デバイスのメタデータです ( ``device_id`` 、 ``platform`` など)。
* ``userInfo``: ユーザーのメタデータです ( ``user_id`` など)。
* ``jobTimeout``: ジョブ操作のタイムアウト (秒) です。
* ``inFilters``: データ処理用の入力フィルターです (省略可)。
* ``outFilters``: 結果処理用の出力フィルターです (省略可)。
* ``resolverRegistry``: コンポーネントリゾルバーのレジストリです (省略可)。

リゾルバーとは?
-------------------

**リゾルバー** とは、文字列の識別子を実際のクラス実装に対応付けるコンポーネントです。FLARE のエッジ SDK においてリゾルバーは、サーバーから受け取った構成データに基づいて、トレーニングコンポーネント、フィルター、その他のプラグインを動的にインスタンス化するために使用されます。

例えば、サーバーがトレーナーコンポーネントを指定するジョブ構成を送信すると、リゾルバーは "ETTrainerExecutor" のような文字列識別子を検索し、インスタンス化すべき実際のクラスに対応付けます。これにより、特定の実装をハードコーディングすることなく、構成ドリブンで柔軟にコンポーネントを読み込めるようになります。

``resolverRegistry`` パラメータを使うと、独自のコンポーネント用にカスタムリゾルバーを登録でき、必要に応じてシステムがそれらを動的に読み込んでインスタンス化できるようになります。

プロパティ
------------

.. code-block:: kotlin

   val jobName: String
   // The name of the federated learning job

メソッド
----------

run()
~~~~~~

連合学習のメインループを開始します。このメソッドは、ジョブが完了するか停止されるまで継続的に実行されます。

.. code-block:: kotlin

   fun run()

**使い方:**

.. code-block:: kotlin

   lifecycleScope.launch {
       flareRunner.run()
   }

stop()
~~~~~~

連合学習の処理を停止し、リソースを解放します。

.. code-block:: kotlin

   fun stop()

**使い方:**

.. code-block:: kotlin

   override fun onDestroy() {
       super.onDestroy()
       flareRunner.stop()
   }

組み込みのコンポーネントリゾルバー
------------------------------------

``AndroidFlareRunner`` には、一般的なコンポーネント向けの組み込みリゾルバーが含まれています。

* ``Executor.ETTrainerExecutor``: ExecuTorch ベースのトレーニングエグゼキューターです。
* ``Trainer.DLTrainer``: ディープラーニングのトレーナーです ( ``ETTrainerExecutor`` に対応付けられます)。
* ``Filter.NoOpFilter``: 何もしないフィルターです。
* ``EventHandler.NoOpEventHandler``: 何もしないイベントハンドラーです。
* ``Batch.SimpleBatch``: シンプルなバッチ処理です。

Connection
==========

FLARE サーバーとの HTTP/HTTPS 通信を管理します。認証、証明書の検証、リクエスト/レスポンスの処理を担当します。

コンストラクター
------------------

.. code-block:: kotlin

   Connection(context: Context)

パラメータ
~~~~~~~~~~~~

* ``context``: Android のアプリケーションコンテキスト

プロパティ
------------

.. code-block:: kotlin

   val hostname: MutableLiveData<String>
   // Server hostname (observable)

   val port: MutableLiveData<Int>
   // Server port (observable)

   val isValid: Boolean
   // Whether the connection configuration is valid

   fun getUserInfo(): Map<String, String>
   // Get current user information

メソッド
----------

setCapabilities(capabilities)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

接続に対してデバイスのケイパビリティを設定します。

.. code-block:: kotlin

   fun setCapabilities(capabilities: Map<String, Any>)

**パラメータ:**
* ``capabilities``: デバイスのケイパビリティのマップです。

setUserInfo(userInfo)
~~~~~~~~~~~~~~~~~~~~~

接続に対してユーザー情報を設定します。

.. code-block:: kotlin

   fun setUserInfo(userInfo: Map<String, String>)

**パラメータ:**
* ``userInfo``: ユーザー情報のマップです。

setScheme(scheme)
~~~~~~~~~~~~~~~~~

HTTP のスキーム (http/https) を設定します。

.. code-block:: kotlin

   fun setScheme(scheme: String)

**パラメータ:**
* ``scheme``: ``"http"`` または ``"https"`` です。

setAllowSelfSignedCerts(allow)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

自己署名証明書を許可するかどうかを設定します。

.. code-block:: kotlin

   fun setAllowSelfSignedCerts(allow: Boolean)

**パラメータ:**
* ``allow``: 自己署名証明書を許可する場合は ``true`` です。

.. warning::
   自己署名証明書を許可すると、セキュリティ上の脆弱性が生じます。開発環境または管理された環境でのみ使用してください。

getJob(jobName, deviceInfo, userInfo)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

サーバーにジョブを要求します。

.. code-block:: kotlin

   suspend fun getJob(
       jobName: String,
       deviceInfo: Map<String, String>,
       userInfo: Map<String, String>
   ): JobResponse?

**パラメータ:**
* ``jobName``: 要求するジョブの名前です。
* ``deviceInfo``: デバイス情報です。
* ``userInfo``: ユーザー情報です。

**戻り値:** 成功した場合は ``JobResponse`` 、それ以外の場合は ``null`` です。

getTask(jobId, taskName)
~~~~~~~~~~~~~~~~~~~~~~~~

サーバーにタスクを要求します。

.. code-block:: kotlin

   suspend fun getTask(
       jobId: String,
       taskName: String
   ): TaskResponse?

**パラメータ:**
* ``jobId``: ジョブの識別子です。
* ``taskName``: 要求するタスクの名前です。

**戻り値:** 成功した場合は ``TaskResponse`` 、それ以外の場合は ``null`` です。

reportResult(jobId, taskId, result)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

タスクの結果をサーバーに報告します。

.. code-block:: kotlin

   suspend fun reportResult(
       jobId: String,
       taskId: String,
       result: Map<String, Any>
   ): ResultResponse?

**パラメータ:**
* ``jobId``: ジョブの識別子です。
* ``taskId``: タスクの識別子です。
* ``result``: タスクの実行結果です。

**戻り値:** 成功した場合は ``ResultResponse`` 、それ以外の場合は ``null`` です。

ETTrainer
=========

オンデバイスでのモデルトレーニングを行う ExecuTorch ベースのトレーナーです。適切なリソース管理のために ``AutoCloseable`` を実装しています。

コンストラクター
------------------

.. code-block:: kotlin

   ETTrainer(
       context: android.content.Context,
       meta: Map<String, Any>,
       dataset: Dataset? = null
   )

パラメータ
~~~~~~~~~~~~

* ``context``: Android のアプリケーションコンテキストです。
* ``meta``: モデルのメタデータです。
* ``dataset``: トレーニング用のデータセットです (省略可)。

メソッド
----------

train(config, dataset, modelData)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

指定された構成とデータセットを用いてモデルをトレーニングします。

.. code-block:: kotlin

   @Throws(Exception::class)
   fun train(
       config: TrainingConfig,
       dataset: Dataset,
       modelData: ByteArray
   ): Map<String, Any>

**パラメータ:**
* ``config``: トレーニングの構成です。
* ``dataset``: トレーニング用データセットです。
* ``modelData``: ExecuTorch 形式のモデルデータです。

**戻り値:** 損失と予測を含むトレーニング結果です。

**スロー:** トレーニングに失敗した場合は ``Exception`` をスローします。

**使い方:**

.. code-block:: kotlin

   ETTrainer(context, meta, dataset).use { trainer ->
       val result = trainer.train(config, dataset, modelData)
   }

close()
~~~~~~~

トレーナーを閉じてリソースを解放します。

.. code-block:: kotlin

   override fun close()

DataSource インターフェース
============================

FL システムにトレーニングデータを提供するためのインターフェースです。

インターフェース定義
----------------------

.. code-block:: kotlin

   interface DataSource {
       fun getDataset(jobName: String, context: Context): Dataset
   }

メソッド
----------

getDataset(jobName, context)
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

指定されたジョブ用のデータセットを取得します。

.. code-block:: kotlin

   fun getDataset(jobName: String, context: Context): Dataset

**パラメータ:**
* ``jobName``: 連合学習ジョブの名前です。
* ``context``: FLARE のコンテキストです。

**戻り値:** トレーニング用の ``Dataset`` インスタンスです。

**実装例:**

.. code-block:: kotlin

   class MyDataSource : DataSource {
       override fun getDataset(jobName: String, context: Context): Dataset {
           return when (jobName) {
               "cifar10_job" -> CIFAR10Dataset(context)
               "xor_job" -> XORDataset("train")
               else -> throw IllegalArgumentException("Unknown job: $jobName")
           }
       }
   }

Dataset インターフェース
==========================

トレーナーにトレーニング用サンプルを提供するためのインターフェースです。

インターフェース定義
----------------------

.. code-block:: kotlin

   interface Dataset {
       fun size(): Int
       fun getBatch(batchSize: Int): List<Map<String, Any>>
   }

メソッド
----------

size()
~~~~~~

データセットに含まれるサンプルの総数を返します。

.. code-block:: kotlin

   fun size(): Int

**戻り値:** サンプル数です。

getBatch(batchSize)
~~~~~~~~~~~~~~~~~~~

トレーニング用サンプルのバッチを取得します。

.. code-block:: kotlin

   fun getBatch(batchSize: Int): List<Map<String, Any>>

**パラメータ:**
* ``batchSize``: 返すサンプルの数です。

**戻り値:** トレーニング用サンプルのリストです。

**実装例:**

.. code-block:: kotlin

   class MyDataset : Dataset {
       private val data = mutableListOf<Map<String, Any>>()

       override fun size(): Int = data.size

       override fun getBatch(batchSize: Int): List<Map<String, Any>> {
           return data.shuffled().take(batchSize)
       }
   }

TrainingConfig
==============

トレーニングのパラメータを保持する構成クラスです。

プロパティ
------------

.. code-block:: kotlin

   val localEpochs: Int
   // Number of local training epochs

   val localBatchSize: Int
   // Batch size for local training

   val localLearningRate: Float
   // Learning rate for local training

   val localMomentum: Float
   // Momentum for local training

   val inFilters: List<Filter>?
   // Input filters

   val outFilters: List<Filter>?
   // Output filters

使用例
==============

基本的なセットアップ
----------------------

.. code-block:: kotlin

   class MainActivity : AppCompatActivity() {
       private lateinit var flareRunner: AndroidFlareRunner

       override fun onCreate(savedInstanceState: Bundle?) {
           super.onCreate(savedInstanceState)

           // Create connection
           val connection = Connection(this)
           connection.setScheme("https")
           connection.setAllowSelfSignedCerts(false) // Use true for development only

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

カスタムデータソース
----------------------

.. code-block:: kotlin

   class CIFAR10DataSource : DataSource {
       override fun getDataset(jobName: String, context: Context): Dataset {
           return CIFAR10Dataset(context)
       }
   }

カスタムデータセット
----------------------

.. code-block:: kotlin

   class XORDataset(private val split: String) : Dataset {
       private val data = generateXORData()

       override fun size(): Int = data.size

       override fun getBatch(batchSize: Int): List<Map<String, Any>> {
           return data.shuffled().take(batchSize)
       }

       private fun generateXORData(): List<Map<String, Any>> {
           // Generate XOR training data
           return listOf(
               mapOf("input" to floatArrayOf(0f, 0f), "label" to 0f),
               mapOf("input" to floatArrayOf(0f, 1f), "label" to 1f),
               mapOf("input" to floatArrayOf(1f, 0f), "label" to 1f),
               mapOf("input" to floatArrayOf(1f, 1f), "label" to 0f)
           )
       }
   }

エラー処理
==============

Android SDK は、例外とロギングを通じて包括的なエラー処理を提供します。

よくある例外
-----------------

* ``NVFlareError`` ( ``com.nvidia.nvflare.sdk.core.NVFlareError`` ): FLARE 関連のエラーを表すカスタムの基底例外です。
* ``IOException`` ( ``java.io.IOException`` ): ネットワーク通信エラーを表す標準的な Java の例外です。
* ``RuntimeException`` ( ``java.lang.RuntimeException`` ): 一般的な実行時エラーを表す標準的な Java の例外です。

例外の階層
-------------------

この SDK では、``NVFlareError`` が ``Exception`` を継承し、具体的なエラー型を提供するカスタムの例外階層を使用しています。実際には、Android アプリは主に ``ServerRequestedStop`` のみを個別に処理し、それ以外のエラーは汎用的に処理します。

.. code-block:: kotlin

   sealed class NVFlareError : Exception() {
       // Network related
       data class JobFetchFailed(override val message: String) : NVFlareError()
       data class TaskFetchFailed(override val message: String) : NVFlareError()
       data class InvalidRequest(override val message: String) : NVFlareError()
       data class AuthError(override val message: String) : NVFlareError()
       data class ServerError(override val message: String) : NVFlareError()
       data class NetworkError(override val message: String) : NVFlareError()

       // Training related
       data class InvalidMetadata(override val message: String) : NVFlareError()
       data class InvalidModelData(override val message: String) : NVFlareError()
       data class TrainingFailed(override val message: String) : NVFlareError()
       object ServerRequestedStop : NVFlareError()
   }

エラー処理のベストプラクティス
--------------------------------

Android SDK は、汎用的な例外を捕捉しつつ ``NVFlareError.ServerRequestedStop`` については個別に処理する、シンプルなエラー処理のアプローチを採用しています。

.. code-block:: kotlin

   try {
       val result = flareRunner.run()
   } catch (e: Exception) {
       Log.e("FLARE", "Training failed with error: $e")

       // Check for specific NVFlareError types
       if (e is NVFlareError.ServerRequestedStop) {
           Log.i("FLARE", "Server requested stop")
           // Gracefully stop training
       } else {
           // Handle other errors generically
           Log.e("FLARE", "Error: ${e.message}")
       }
   }

.. note::
   Connection クラスでは、より具体的なエラー処理が行われており、``IOException`` を ``NVFlareError.NetworkError`` に変換したり、HTTP のステータスコードに応じて適切な ``NVFlareError`` のサブタイプをスローしたりします。ただし、アプリケーション本体のコードでは、上記のシンプルなアプローチを採用しています。

ロギング
----------

この SDK は Android の標準的なロギングシステムを使用します。詳細な情報を確認するには、デバッグログを有効にしてください。

.. code-block:: kotlin

   if (BuildConfig.DEBUG) {
       Log.d("AndroidFlareRunner", "Starting federated learning")
   }

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

**証明書のエラー**
* 本番環境では適切な証明書検証を行ってください。
* セキュリティを強化するために証明書ピンニングを検討してください。
* 自己署名証明書でのテストは開発環境に限定してください。

ベストプラクティス
====================

* **リソース管理**: ``ETTrainer`` には常に try-with-resources または ``AutoCloseable`` を使用してください。
* **エラー処理**: 包括的なエラー処理とロギングを実装してください。
* **セキュリティ**: 本番環境では適切な証明書検証を行ってください。
* **パフォーマンス**: メモリ使用量を監視し、モデルサイズを最適化してください。
* **テスト**: さまざまなネットワーク状況やデバイス構成でテストしてください。

詳細については、:ref:`モバイル開発ガイド <flare_mobile>` および `エッジのサンプル <https://github.com/NVIDIA/NVFlare/tree/main/examples/advanced/edge>`_ を参照してください。
