.. _shareable:

Shareable
=========
:class:`Shareable<nvflare.apis.shareable.Shareable>` オブジェクトは、サーバーとクライアント間の通信を表します。
技術的には、Shareable オブジェクトは Python の dict として実装されています。この dict には 2 種類の情報が含まれます。

ヘッダー
^^^^^^^^
Shareable の特別な項目として "headers" があり、これ自体も dict です。ヘッダーは通信に関するメタ情報 (例えばピアの識別名、ピアのジョブ ID / 実行番号、クッキー、リターンコードなど) を運ぶために使用されます。ヘッダーは通常、フレームワークによって追加および処理されます。

コンテンツ
^^^^^^^^^^
Shareable オブジェクト内のその他すべての項目は、通信のコンテンツです。Shareable オブジェクトには任意の要素を格納できますが、コンテンツ要素のキーとして ReservedHeaderKey.HEADERS を決して使用しないでください。

Shareable のすべてのメソッドについては、:class:`nvflare.apis.shareable.Shareable` を参照してください。

ピアプロパティ
--------------
リクエストやレスポンス (いずれも Shareable オブジェクトです) を処理する際に、以下を使ってピアサイトに関する情報を取得できます::

    peer_props = shareable.get_peer_props()

これはピアサイトの情報を含む Python の辞書です。

クッキー
--------
ワークフロー開発者の場合、クライアントの Get Task リクエストを処理する際に、クライアントに送信されるタスク割り当てとともに何らかのコンテキスト情報を保持し、その情報がクライアントのタスク結果の提出時にエコーバックされることを期待することがあります。これはクッキーの仕組みを通じて実現できます。

クッキーは名前付きのデータ (キー / 値のペア) にすぎません。タスクリクエストの処理中に、関与するコンポーネント (Controller、Filter、イベントハンドラーなど) はいずれも FLContext にクッキーを追加できます::

    shareable.add_cookie(name='foo', data=whatEverData)

Shareable オブジェクトのヘッダーは、"Cookie Jar" と呼ばれる特別なパブリックプロパティを保持できます。これは単なる Python の dict です。

NVIDIA FLARE フレームワークは、クライアントがサーバーにタスク結果を提出する際に、cookie jar プロパティがサーバーに送り返されることを保証します。

.. note::

    クッキーのデータはサーバーが利用するためだけのものです。クライアントの処理ロジックは、クッキーのデータに関する知識に依存すべきではありません。

リターンコード
--------------
クライアントのタスク結果提出 (これも Shareable オブジェクトです) におけるもう 1 つの特別なヘッダー要素は "return code" です。これは、タスクが正常に実行されたかどうかを示す文字列です。この要素がヘッダーに存在しない場合は、成功したとみなされます。

リターンコードは以下で取得できます::

    shareable.get_return_code()

リターンコードは、タスクの実行を妨げたエラー状態を示します::

    MISSING_PEER_CONTEXT = "MISSING_PEER_CONTEXT"
    BAD_PEER_CONTEXT = "BAD_PEER_CONTEXT"
    RUN_MISMATCH = "RUN_MISMATCH"
    TASK_UNKNOWN = "TASK_UNKNOWN"
    TASK_DATA_FILTER_ERROR = "TASK_DATA_FILTER_ERROR"
    TASK_RESULT_FILTER_ERROR = "TASK_RESULT_FILTER_ERROR"
    EXECUTION_EXCEPTION = "EXECUTION_EXCEPTION"
    EXECUTION_RESULT_ERROR = "EXECUTION_RESULT_ERROR"

一部のエラー状態は決して発生しないはずのものです (例: MISSING_PEER_CONTEXT、BAD_PEER_CONTEXT)。

RUN_MISMATCH - クライアントとサーバーで RUN の同期が取れていません。これが発生すると、クライアントは自動的に RUN を終了します。

TASK_UNKNOWN - クライアントが、割り当てられたタスクに対する Executor を見つけられません。これは通常、タスクテーブルの設定ミスによって引き起こされます。

TASK_DATA_FILTER_ERROR - クライアントがタスクデータのフィルタリングに失敗しました。通常はいずれかのフィルターのバグです。

TASK_RESULT_FILTER_ERROR - クライアントが、タスク Executor によって生成された結果のフィルタリングに失敗しました。通常はいずれかのフィルターのバグです。

EXECUTION_EXCEPTION - クライアントがタスクの実行に失敗しました。通常は Executor のコードのバグです。

EXECUTION_RESULT_ERROR - Executor が結果として Shareable オブジェクトを生成できませんでした。コードのバグです。
