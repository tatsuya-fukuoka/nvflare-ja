.. _event_system:

NVIDIA FLARE のイベント機構
============================
NVIDIA FLARE には強力なイベント機構が備わっており、:class:`FLComponent<nvflare.apis.fl_component.FLComponent>` の
サブクラスであるすべてのオブジェクトへ動的に通知を送ることができます。

NVIDIA FLARE のすべてのコンポーネント型(例: Filter、Executor、Responder、Controller、Widget など)は
``FLComponent`` のサブクラスであるため、すべてイベントハンドラーです。追加のサブクラスを作成すれば、そのオブジェクトも
自動的にイベントハンドラーになります。

イベント
--------
イベントは、システムロジックの実行中における重要な瞬間を表します。そのような瞬間の典型的な例は次のとおりです:

    - 何らかのアクションが起こる前(例: 集約の前)
    - 何らかのアクションが起こった後(例: 集約の後)
    - 何らかの重要なデータが利用可能になったとき(例: ベストモデルの更新)

独自のイベント型を自由に考案し、処理ロジックの中でイベントを発火させることができます。

イベント型
^^^^^^^^^^
イベント型は単なる文字列です。誰でも新しいイベント型を考案できるため、名前の衝突を避けるために
イベントの命名規則を定義しておくべきです。

イベントデータ
^^^^^^^^^^^^^^
イベントデータは何でも構いません。ほとんどの(ただしすべてではない)イベント型はデータを持ちます。イベントデータは、
明確に定義された名前のプロパティとして FLContext オブジェクトに格納されます。そのイベント型を待ち受けるイベントハンドラーは、
FLContext の get_prop() メソッドでイベントデータを取得します。

すべてのイベント型がデータを持つわけではありません。単にタイミングを知らせるためだけに使われるイベントもあります。

イベントの発火
^^^^^^^^^^^^^^
処理ロジックの中でイベントを発火させるには:
    - まず、処理ロジックのどこでイベントを発火させるかを決めます
    - 次に、イベントにデータがある場合は fl_ctx に追加します。ほとんどの場合、すでに fl_ctx を持っているはずです。
      持っていない場合は、engine.new_context() で新しく作成できます。
    - 最後に、イベントを発火させます。

典型的なコードパターンは次のとおりです::

    fl_ctx.set_prop(key="yourEventDataKey",
                      data=yourEventData,
                      private=True,
                      sticky=False)
    engine = fl_ctx.get_engine()
    engine.fire_event(event_type="youEventTypeName",
                        fl_ctx=fl_ctx)

.. note::

    注記: 通常、イベントデータは private かつ非 sticky にすべきです。ただし、RUN の間データを恒久的に
    保持しておきたい場合は、sticky に設定することもできます。

.. note::

    注記: イベントが複数のデータを持ち得る場合は、データごとに1回ずつ、fl_ctx.set_prop() を複数回
    呼び出すだけで済みます。

イベントにデータがない場合は、最初のステップを省略できます。

イベントの処理
^^^^^^^^^^^^^^
イベントが発火されると、すべてのイベント処理コンポーネントの handle_event() メソッドが呼び出され、イベントを処理します。

.. note::

    注記: コンポーネントが呼び出される順序は非決定的です。したがって、あるコンポーネントが別のコンポーネントより
    先に呼び出されることに依存してはいけません。この動作は、必要に応じて将来変更される可能性があります。

関心のあるイベントを処理するための典型的なコードパターンは次のとおりです::

    def handle_event(self, event_type: str, fl_ctx: FLContext):
       if event_type == 'event1':
           event_data = fl_ctx.get_prop('eventDataKeyName')
           ...
       elif event_type == 'event2':
           # process

イベントハンドラーは順番に呼び出されます。あるハンドラーの失敗(例外)によって、後続のハンドラーの呼び出しが止まることはありません。

組み込みイベント型
------------------
NVIDIA FLARE のシステム定義イベント型は :class:`nvflare.apis.event_type.EventType` で規定されています:

.. csv-table::
   :header: イベント, 説明, データキー, データ型, サーバー, クライアント

    START_RUN,新しい RUN がまもなく開始される,fl_ctx.get_job_id(),int,X,X
    END_RUN,現在の RUN がまもなく終了する,fl_ctx.get_job_id(),int,X,X
    START_WORKFLOW,ワークフローがまもなく開始される,FLContextKey.WORKFLOW,int,X,
    END_WORKFLOW,ワークフローがまもなく終了する,FLContextKey.WORKFLOW,int,X,
    BEFORE_PROCESS_SUBMISSION,タスク結果の提出がまもなく処理される,FLContextKey.TASK_NAME,str,X,
    ,,FLContextKey.TASK_RESULT,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    AFTER_PROCESS_SUBMISSION,タスク結果の処理が完了した,FLContextKey.TASK_NAME,str,X,
    ,,FLContextKey.TASK_RESULT,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    BEFORE_TASK_DATA_FILTER,タスクデータがまもなくフィルターされる,FLContextKey.TASK_NAME,str,X,X
    ,,FLContextKey.TASK_DATA,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    AFTER_TASK_DATA_FILTER,タスクデータがフィルターされた,FLContextKey.TASK_NAME,str,X,X
    ,,FLContextKey.TASK_DATA,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    BEFORE_TASK_RESULT_FILTER,タスク結果がまもなくフィルターされる,FLContextKey.TASK_NAME,str,X,X
    ,,FLContextKey.TASK_RESULT,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    AFTER_TASK_RESULT_FILTER,タスク結果がフィルターされた,FLContextKey.TASK_NAME,str,X,X
    ,,FLContextKey.TASK_RESULT,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    BEFORE_TASK_EXECUTION,タスクの実行がまもなく開始される,FLContextKey.TASK_NAME,str,,X
    ,,FLContextKey.TASK_DATA,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    AFTER_TASK_EXECUTION,タスクの実行が終了した,FLContextKey.TASK_NAME,str,,X
    ,,FLContextKey.TASK_DATA,Shareable,,
    ,,FLContextKey.TASK_RESULT,Shareable,,
    ,,FLContextKey.TASK_ID,str,,
    BEFORE_SEND_TASK_RESULT,タスク結果がまもなくサーバーへ送信される,FLContextKey.TASK_NAME,,,X
    ,,FLContextKey.TASK_DATA,,,
    ,,FLContextKey.TASK_RESULT,,,
    ,,FLContextKey.TASK_ID,,,
    AFTER_SEND_TASK_RESULT,タスク結果がサーバーへ送信された,FLContextKey.TASK_NAME,,,X
    ,,FLContextKey.TASK_RESULT,,,
    ,,FLContextKey.TASK_DATA,,,
    ,,FLContextKey.TASK_ID,,,
    FATAL_SYSTEM_ERROR,致命的なエラーが発生し RUN が中止される,FLContextKey.EVENT_DATA,str(エラーテキスト),X,X
    FATAL_TASK_ERROR,タスク実行中の致命的なエラーによりタスクが中止される,FLContextKey.EVENT_DATA,str(エラーテキスト),,X
    ERROR_LOG_AVAILABLE,エラーログメッセージが利用可能,FLContextKey.EVENT_DATA,str(ログメッセージ),X,X
    EXCEPTION_LOG_AVAILABLE,例外ログメッセージが利用可能,FLContextKey.EVENT_DATA,str(ログメッセージ),X,X

RUN ライフサイクルイベント
--------------------------
すべてのイベント型の中で最も重要なのは START_RUN と END_RUN です。

NVIDIA FLARE では、FL の実験は RUN の中で行われます。FL の研究の過程で、研究者は期待する結果を得るために
通常多くの RUN を実施する必要があります。

START_RUN イベントは、新しい RUN がまもなく開始されるときに発生します。通常は研究者が admin コマンドで
トリガーします。コンポーネントの初期化が必要な場合は、このイベント型を待ち受けて、コンポーネントを動作可能な状態に
準備しなければなりません。

END_RUN イベントは、RUN がまもなく終了するときに発生します。通常はワークフローの完了、または研究者による abort
コマンドによってトリガーされます。必要であれば、このイベントを使ってコンポーネントを適切に終了処理・クリーンアップできます。

コンポーネントが他のコンポーネントから利用できるサービスを提供している場合は、START_RUN のタイミングで、一意に定義した
プロパティ名のもとで private かつ sticky なプロパティとしてコンポーネントを fl_ctx に格納できます。他のコンポーネントは後で
この名前でコンポーネントを取得し、そのサービスを呼び出すことができます。

ローカルイベントと連合イベント
------------------------------
ローカルイベントはクライアント内に閉じたイベントであり、連合イベント(fed イベント)は他のサイトにもブロードキャストされます。

:class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` ウィジェットは、
ローカルイベントを連合イベントに変換します。

:class:`AnalyticsSender<nvflare.app_common.widgets.streaming.AnalyticsSender>` は、クライアント上のローカルイベントとして
"analytix_log_stats" というイベントをトリガーします。サーバー側でこのイベントを受信したい場合は、ローカルイベントを
連合イベントに変換する必要があり、これは :class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` ウィジェットで行えます。

:class:`ConvertToFedEvent<nvflare.app_common.widgets.convert_to_fed_event.ConvertToFedEvent>` ウィジェットはイベントを
連合イベントに変換し、そのイベントにプレフィックスを付加します。この例では "fed.analytix_log_stats" になります。このイベントは
サーバー上の :class:`TBAnalyticsReceiver<nvflare.app_common.pt.tb_receiver.TBAnalyticsReceiver>` コンポーネントによって
処理され、サーバーはストリーミングされた分析データを受信できます。

これらのコンポーネントを使用する例については、:ref:`TensorBoard Streaming <tensorboard_streaming>` の例を参照してください。
