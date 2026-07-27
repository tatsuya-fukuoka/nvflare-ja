.. _variable_resolution:

ジョブ設定における変数解決
================================

FLARE のジョブは、``config_fed_client.json`` と ``config_fed_server.json`` という設定ファイルで定義されます。
これら 2 つのファイルは、サーバープロセスおよび FL クライアントプロセスで使用されるコンポーネント (Python オブジェクト) を設定します。
コンポーネントの設定には、Python オブジェクトのクラスパス (``path`` または ``class_path`` による指定) と、そのオブジェクトのコンストラクタへの引数が含まれます。
設定ファイルは、サーバー / クライアントのジョブプロセスの開始時に処理され、それらのコンポーネントが作成されます。

以下は典型的なジョブ設定の例です。

.. code-block:: json

   {
      "format_version": 2,
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.np.np_trainer.NPTrainer",
               "args": {
                  "sleep_time": 1.5,
                  "model_dir": "model"
               }
            }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
      ]
   }

上の例に示すように、``executor`` コンポーネントには 2 つの引数 (sleep_time と model_dir) があり、どちらも明示的に指定されています。

変数解決
--------------

ユーザーがコンポーネントの引数値をいろいろ変えて試したい場合、それらの実験用引数をファイル内から探して修正するのではなく、共通の場所 (例: 設定ファイルの先頭) でまとめて管理したいことがあります。
これは、実験対象のコンポーネントが複数ある場合に特に当てはまります。

FLARE では、変数解決 (Variable Resolution) と呼ばれる仕組みでこれを可能にしています。
各設定引数に値をハードコードする代わりに、引数の値として変数参照 (Variable Reference) を使用し、その変数の値を別の場所 (例: 設定ファイルの先頭) で定義できます。

以下は、上記の例を変数解決を用いて設定したものです。

.. code-block:: json

   {
      "format_version": 2,
      "result_dir": "result",
      "sleep_time": 1.5,
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.np.np_trainer.NPTrainer",
               "args": {
                  "sleep_time": "{sleep_time}",
                  "model_dir": "{result_dir}"
               }
            }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
      ]
   }


この例からわかるように、変数定義 (Variable Definition、Var Def) は、変数名 (Variable Name、Var Name) に対する値を定義する単純な JSON 要素です。
変数参照 (Variable Reference、Var Ref) は、参照する変数名を波括弧で囲んで埋め込んだ文字列です: ``{VarName}``。

var ref は、他の情報と組み合わせて文字列の中で使用できます。
例えば、``model_dir`` 引数にプレフィックスを含めるように定義できます:
``/tmp/fl_work/{result_dir}``

1 つの引数値の中で複数の変数を参照することもできます:
``{root_dir}/{result_dir}``

引数値が単一の var ref のみで構成されている場合、それは単純変数参照 (Simple Var Ref、SVR) と呼ばれます。
他の情報を伴う var ref や、複数の var ref といったその他の用法は、複合変数参照 (Complex Var Ref、CVR) と呼ばれます。
参照を解決して引数値を計算する際、SVR と CVR には重要な違いがあります。
SVR は対応する変数定義の本来の型に解決されるのに対し、CVR は常に、参照された変数の値を用いた文字列に解決されます。
SVR はプリミティブな変数 (数値、真偽値、文字列) と非プリミティブな変数 (リストや dict) の両方を参照できますが、CVR ではプリミティブな変数しか使用できません。

定義済みのシステム変数
============================

参照される変数は必ず定義されている必要があります。ユーザー定義の変数の場合、通常は上記の例のように、設定ファイルのどこか (例: ファイルの先頭) に第 1 レベルの要素として定義します。

FLARE では以下のシステム変数があらかじめ定義されており、ジョブ設定内で使用できます。

- SITE_NAME - サイトの名前 (サーバーまたは FL クライアント)
- WORKSPACE - サイトのワークスペースのディレクトリ
- JOB_ID - ジョブ ID
- ROOT_URL - FL サーバーに接続するための url
- SECURE_MODE - 通信がセキュアモードかどうか

システム変数は大文字で命名されている点に注意してください。ユーザー定義の変数とシステム変数との名前の衝突を避けるため、ユーザー定義の変数はすべて小文字で命名してください。

次の例では、CellPipe の設定におけるシステム変数の使用方法を示します。

OS 環境変数
==================

OS の環境変数は、ドル記号を用いてジョブ設定内で参照できます。

``{$EnvVarName}``

これにより、ジョブ設定を OS の環境変数で制御できるようになります。
例えば、環境変数 (例: NVFLARE_MODEL_DIR) を使って学習済みモデルの保存先を指定しておけば、システム運用者はジョブ設定を変更することなくモデルの保存場所を変更できます。
なお、``$VarName`` という名前の変数がすでにジョブ設定内で定義されている場合は、その定義が対応する OS 環境変数よりも優先されます。

以下の例は、OS 環境変数を使って model_dir の場所を制御する方法を示しています。

.. code-block:: json

   {
      "format_version": 2,
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.np.np_trainer.NPTrainer",
               "args": {
                  "model_dir": "{$NVFLARE_MODEL_DIR}"
               }
            }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
      ]
   }

他の変数定義と同様に、OS 環境変数は SVR と CVR の両方で参照できます。

パラメータ化された変数定義
================================

この応用的なトピックを説明する前に、比較のために、まずこの手法を使っていないジョブ設定の例を示します。

.. code-block:: json

   {
      "format_version": 2,
      "pipe_token": "pipe_123",
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.executors.task_exchanger.TaskExchanger",
               "args": {
                  "pipe_id": "task_pipe"
               }
            }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "components": [
         {
            "id": "task_pipe",
            "path": "nvflare.fuel.utils.pipe.cell_pipe.CellPipe",
            "args": {
               "mode": "passive",
               "site_name": "{SITE_NAME}",
               "token": "{pipe_token}",
               "root_url": "{ROOT_URL}",
               "secure_mode": "{SECURE_MODE}",
               "workspace_dir": "{WORKSPACE}"
            }
         },
         {
            "id": "metric_pipe",
            "path": "nvflare.fuel.utils.pipe.cell_pipe.CellPipe",
            "args": {
               "mode": "passive",
               "site_name": "{SITE_NAME}",
               "token": "{pipe_token}",
               "root_url": "{ROOT_URL}",
               "secure_mode": "{SECURE_MODE}",
               "workspace_dir": "{WORKSPACE}"
            }
         },
         {
            "id": "metric_receiver",
            "path": "nvflare.widgets.metric_receiver.MetricReceiver",
            "args": {
               "pipe_id": "metric_pipe"
            }
         }
      ]
   }


このジョブでは 2 つのパイプが必要です。1 つはタスク交換用 (task_pipe)、もう 1 つはメトリクス収集用 (metric_pipe) です。
これらの設定をよく見ると、設定すべき引数が多く、2 つのパイプの設定は ``id`` の値を除いてまったく同一であることがわかります。多数の引数を複数の場所で設定するのは、手間がかかり、間違いも起こりやすくなります。

改善策の 1 つは、2 つのパイプの引数に SVR を利用することです。

.. code-block:: json

   {
      "format_version": 2,
      "pipe_token": "pipe_123",
      "executors": [
         {
            "tasks": [
               "train"
            ],
            "executor": {
               "path": "nvflare.app_common.executors.task_exchanger.TaskExchanger",
               "args": {
                  "pipe_id": "task_pipe"
               }
            }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "pipe_args": {
         "mode": "passive",
         "site_name": "{SITE_NAME}",
         "token": "{pipe_token}",
         "root_url": "{ROOT_URL}",
         "secure_mode": "{SECURE_MODE}",
         "workspace_dir": "{WORKSPACE}"
      },
      "components": [
         {
            "id": "task_pipe",
            "path": "nvflare.fuel.utils.pipe.cell_pipe.CellPipe",
            "args": "{pipe_args}"
         },
         {
            "id": "metric_pipe",
            "path": "nvflare.fuel.utils.pipe.cell_pipe.CellPipe",
            "args": "{pipe_args}"
         },
         {
            "id": "metric_receiver",
            "path": "nvflare.widgets.metric_receiver.MetricReceiver",
            "args": {
               "pipe_id": "metric_pipe"
            }
         }
      ]
   }

この例のこのバージョンでは、2 つのパイプの引数を var def の ``pipe_args`` に移し、コンポーネントの ``args`` はその var def を参照するだけになっています。
これは元のバージョンより優れていますが、2 つのパイプの path は依然として両方のコンポーネントで繰り返し記述する必要があります。

パラメータ化された変数定義を使うと、さらに改善できます。

.. code-block:: json

   {
      "format_version": 2,
      "pipe_token": "pipe_123",
      "executors": [
         {
         "tasks": [
            "train"
         ],
         "executor": {
            "path": "nvflare.app_common.executors.task_exchanger.TaskExchanger",
            "args": {
               "pipe_id": "task_pipe"
            }
         }
         }
      ],
      "task_result_filters": [],
      "task_data_filters": [],
      "@pipe_def": {
         "id": "{pipe_id}",
         "path": "nvflare.fuel.utils.pipe.cell_pipe.CellPipe",
         "args": {
         "mode": "passive",
         "site_name": "{SITE_NAME}",
         "token": "{pipe_token}",
         "root_url": "{ROOT_URL}",
         "secure_mode": "{SECURE_MODE}",
         "workspace_dir": "{WORKSPACE}"
         }
      },
      "components": [
         "{@pipe_def:pipe_id=task_pipe}",
         "{@pipe_def:pipe_id=metric_pipe}",
         {
            "id": "metric_receiver",
            "path": "nvflare.widgets.metric_receiver.MetricReceiver",
            "args": {
               "pipe_id": "metric_pipe"
            }
         }
      ]
   }

ここで見られるように、``@pipe_def`` はパラメータ化された変数定義 (parameterized variable definition、PVD) です。
PVD の名前は ``@`` 記号で始まる必要があります。PVD は通常、他の変数への参照を含めて定義され、その値は PVD が参照される時点で与えられます。
この例では、``@pipe_def`` PVD が、具体的なパイプ設定に解決可能なパイプ設定テンプレートを定義しています。
``components`` セクションでは、この PVD が task_pipe と metric_pipe という 2 つのパイプの設定に使用されています。

PVD は SVR (単純変数参照) でのみ参照できます。
PVD を参照する際には、その PVD 内の変数に値を与えます。
この例では、``pipe_id`` が変数であり、2 つの異なるパイプに対して 2 つの異なる値を取ります。

PVD への参照は、次の一般的な形式になります。

``{PvdName:N1=V1:N2=V2:...}``

PvdName は PVD の名前です。
PVD 内の各変数の値は N=V の形式で与えます。ここで N は変数名、V はその値です。
なお、V はさらに他の変数を参照することもできます。

参照の外側で N に対する値が定義されている場合、参照内で与えられた値が優先されることに注意してください。
例えば、参照で ``pipe_token`` の値を与えた場合、その値がファイル先頭で定義された値よりも優先されます。

``"{@pipe_def:pipe_id=task_pipe:pipe_token=pipe_789}"``

この場合、パイプ ``task_pipe`` を作成する際の ``pipe_token`` の値は、ファイル先頭で定義された ``pipe_123`` ではなく ``pipe_789`` になります。
