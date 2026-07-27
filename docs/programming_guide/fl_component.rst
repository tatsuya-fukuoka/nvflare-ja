.. _fl_component:

FLComponent
===========
.. currentmodule:: nvflare.apis.fl_component.FLComponent

:class:`nvflare.apis.fl_component.FLComponent` は、すべてのFLコンポーネントの基底クラスです。エグゼキューター、コントローラー、フィルター、アグリゲーター、およびそれらのサブタイプ(例えばトレーナー)は、すべてFLComponentです。

.. literalinclude:: ../../nvflare/apis/fl_component.py
    :language: python
    :lines: 28-90

各 ``FLComponent`` は、新しいインスタンスが作成されると、自動的にイベントハンドラーとしてシステムに追加されます。
:meth:`handle_event<handle_event>` を実装することで、FLワークフローに追加のカスタムアクションを組み込むことができます。

イベントを発火するには :meth:`fire_event<fire_event>` を使用でき、参加者をまたいでイベントを発火するには :meth:`fire_fed_event<fire_fed_event>` を使用できます。

ログ出力メソッド :meth:`log_debug<log_debug>`、:meth:`log_info<log_info>`、:meth:`log_warning<log_warning>`、
:meth:`log_error<log_error>`、:meth:`log_exception<log_exception>` を使用すると、ログメッセージにコンテキスト情報の接頭辞が付き、他のシステム機能と統合されます。

システムがそれ以上の動作を妨げるエラーに遭遇した極端なケースでは、:meth:`task_panic<task_panic>` を呼び出してタスクを終了するか、:meth:`system_panic<system_panic>` を呼び出して実行(run)を終了できます。

組み込みFLComponentにおけるデフォルトデータ
---------------------------------------------
NVIDIA FLAREが提供する組み込みのFLComponentについては、以下のデータが ``Shareable`` と ``FLContext`` に設定されることを保証しています。

また、ニーズに合わせて ``Sharable`` オブジェクトの構造を定義し、トレーニングに関連するデータを ``FLContext`` に追加することもできます。
