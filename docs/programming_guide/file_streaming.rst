.. _file_streaming:

##########################################
FLARE ファイルストリーミング
##########################################

ファイルストリーミングは、1つのファイルを1つ以上の受信者と共有できるようにする機能です。ファイルの所有者は FL サーバでも任意の FL クライアントでもかまいません。ファイルストリーミングは、大量のデータをメッセージで送信する方法に代わる効果的な手段となります。

大きなメッセージを送信する場合、主に2つの問題があります。
- 送信前にメッセージをバイト列へシリアライズするために、大きなメモリ空間が必要になります。メモリが飽和すると、すべての処理が非常に遅くなります。
- 単一のメッセージとして送信される大きなバイト配列はネットワークを飽和させ、全体の処理速度を低下させる可能性があります。

一方でファイルストリーミングは、大きなファイルを多数の小さなメッセージに分けて送信し、各メッセージにはファイルデータのチャンクが含まれます。大きなファイルがメモリに完全にロードされることはありません。ネットワーク上には小さなメッセージしか送信されないため、ネットワークが滞る可能性も低くなります。

プッシュ vs. プル
==================

ファイルをある場所から別の場所へ送る方法には、プッシュとプルの2種類があります。

- **プッシュ**: ファイルの所有者が受信者にファイルを送信します。プッシュの処理はやや厳格で、複数の受信者にファイルを送信する場合、すべての受信者が同じチャンクを同時に処理しなければなりません。いずれか1つでも失敗すると、送信処理全体が失敗します。したがって、単一の受信者にファイルを送信する場合に最も有用です。「プッシュ」方式は `FileStreamer` クラスで実装されています。

- **プル**: ファイルの所有者はまずファイルを準備し、そのファイルの参照 ID (RID) を取得します。次に、任意の方法(ブロードキャストなど)で RID をすべての受信者に送信します。RID を受信した各受信者は、ファイル全体を受信し終わるまでチャンク単位でファイルをプルします。プルは受信者間で同期が不要なため、より緩やかな方式です。各受信者は自分のペースでファイルをプルできるため、複数の受信者とファイルを共有する場合に有用です。「プル」方式は `FileDownloader` クラスで実装されています。

ファイルのライフサイクル管理
=============================

ファイルのダウンロード(プル)は複数の受信者に対してより堅牢ですが、ファイル管理という課題があります。最終的には、不要になったファイルを(必要に応じて)削除するのはファイル所有者の責任です。

各受信者が自分のペースでファイルをダウンロードできるため、ファイル所有者側でそのファイルが不要になる確定的な時点は存在しません。有効な方法の1つはアクティビティタイムアウトです。指定した期間、どの受信者からもダウンロード活動がなければ、そのファイルはもう必要ないと見なすことができます。

FileStreamer
============

`FileStreamer` (プッシュ) は予告なしに受信者へファイルを送信するため、受信者は受信したファイルを処理できるよう事前にセットアップしておく必要があります。これは `FileStreamer.register_stream_processing` を呼び出すことで行います。

.. code-block:: python

   class FileStreamer(StreamerBase):
      @staticmethod
      def register_stream_processing(
          fl_ctx: FLContext,
          channel: str,
          topic: str,
          dest_dir: str = None,
          stream_done_cb=None,
          chunk_consumed_cb=None,
          **cb_kwargs,
      ):
          """Register for stream processing on the receiving side.

          Args:
              fl_ctx: the FLContext object
              channel: the app channel
              topic: the app topic
              dest_dir: the destination dir for received file. If not specified, system temp dir is used
              stream_done_cb: if specified, the callback to be called when the file is completely received
              chunk_consumed_cb: if specified, the callback to be called when a chunk is processed
              **cb_kwargs: the kwargs for the stream_done_cb

          Returns: None

          Notes: the stream_done_cb must follow stream_done_cb_signature as defined in apis.streaming.
          """

ファイルを共有するには、送信側とすべての受信側の間でチャネルとトピックを取り決めておく必要があります。すべての受信者は、ファイルの受信が想定されるチャネルとトピックごとに、このメソッドを1回呼び出さなければなりません。通常、この呼び出しはアプリケーションの開始時に、START_RUN イベントを処理するイベントハンドラの中で行われます。

`stream_done_cb` は、ファイルが完全に受信された際にアプリケーションへ通知するために呼び出されます。以下のシグネチャに従う必要があります。

.. code-block:: python

   def stream_done_cb_signature(stream_ctx: StreamContext, fl_ctx: FLContext, **kwargs):
      """This is the signature of stream_done_cb.

      Args:
          stream_ctx: context of the stream
          fl_ctx: FLContext object
          **kwargs: the kwargs specified when registering the stream_done_cb.

      Returns: None
      """

`stream_ctx` にはストリームに関する情報が含まれており、ファイル所有者が `stream_file` を呼び出してファイルを送信する際に指定した情報も含まれます。

受信したデータは一時ファイルに保存されます。`stream_ctx` からファイル情報を取得するには、以下のメソッドを使用します。

.. code-block:: python

   @staticmethod
   def get_file_name(stream_ctx: StreamContext):
      """Get the file base name property from stream context.
      This method is intended to be used by the stream_done_cb() function of the receiving side.

      Args:
          stream_ctx: the stream context

      Returns: file base name
      """

.. code-block:: python

   @staticmethod
   def get_file_location(stream_ctx: StreamContext):
      """Get the file location property from stream context.
      This method is intended to be used by the stream_done_cb() function of the receiving side.

      Args:
          stream_ctx: the stream context

      Returns: location (full file path) of the received file
      """

.. code-block:: python

   @staticmethod
   def get_file_size(stream_ctx: StreamContext):
      """Get the file size property from stream context.
      This method is intended to be used by the stream_done_cb() function of the receiving side.

      Args:
          stream_ctx: the stream context

      Returns: size (in bytes) of the received file
      """

受信したファイルをどう扱うか、またそのファイルを削除するかどうか・いつ削除するかは、あなたの責任であることに注意してください。

ファイルの送信
==============

ファイルの所有者は、`FileStreamer` モジュールで定義されている `stream_file` 関数を呼び出すことで、1つ以上の受信者にファイルを送信します。

.. code-block:: python

   def stream_file(
      channel: str,
      topic: str,
      stream_ctx: StreamContext,
      targets: List[str],
      file_name: str,
      fl_ctx: FLContext,
      chunk_size=None,
      chunk_timeout=None,
      optional=False,
      secure=False,
   ) -> (str, bool):
      """Stream a file to one or more targets.

      Args:
          channel: the app channel
          topic: the app topic
          stream_ctx: context data of the stream
          targets: targets that the file will be sent to
          file_name: full path to the file to be streamed
          fl_ctx: a FLContext object
          chunk_size: size of each chunk to be streamed. If not specified, default to 1M bytes.
          chunk_timeout: timeout for each chunk of data sent to targets.
          optional: whether the file is optional
          secure: whether P2P security is required

      Returns: a tuple of (RC, Result):
          - RC is ReturnCode.OK or ReturnCode.ERROR;
          - Result is whether the streaming completed successfully

      Notes: this is a blocking call - only returns after the streaming is done.
      """

引数は見ての通りです。追加情報は dict である `stream_ctx` を通じて送信できる点に注意してください。この情報は、受信者側で登録された `stream_done_cb` から利用できます。

FileDownloader
==============

ファイルのダウンロード処理には3つのステップが必要です。

1. データ所有者は受信者と共有するファイルを準備し、各ファイルに対して1つの参照 ID (RID) を取得します。
2. データ所有者は RID をすべての受信者に送信します。通常はブロードキャストメッセージで行われます。
3. 受信者は受け取った RID を使って、ファイルを1つずつダウンロードします。

ダウンロードの準備
--------------------

データ所有者はまず、`FileDownloader` の `new_transaction` メソッドと `add_file` メソッドを使って、他の受信者と共有するファイルを準備します。これらは次のように定義されています。

.. code-block:: python

   class FileDownloader:

      @classmethod
      def new_transaction(
          cls,
          cell: Cell,
          timeout: float,
          timeout_cb,
          **cb_kwargs,
      ):
          """Create a new file download transaction.

          Args:
              cell: the cell for communication with recipients
              timeout: timeout for the transaction
              timeout_cb: CB to be called when the transaction is timed out
              **cb_kwargs: args to be passed to the CB

          Returns: transaction id

          The timeout_cb must follow this signature:

              cb(tx_id, file_names: List[str], **cb_args)
          """

.. code-block:: python

      @classmethod
      def add_file(
          cls,
          transaction_id: str,
          file_name: str,
          file_downloaded_cb=None,
          **cb_kwargs,
      ) -> str:
          """Add a file to be downloaded to the specified transaction.

          Args:
              transaction_id: ID of the transaction
              file_name: name of the file to be downloaded
              file_downloaded_cb: CB to be called when the file is done downloading
              **cb_kwargs: args to be passed to the CB

          Returns: reference id for the file.

      The file_downloaded_cb must follow this signature:

          cb(ref_id: str, to_receiver: str, status: str, file_name: str, **cb_kwargs)
          """

まず `new_transaction` メソッドを呼び出してトランザクション ID を取得します。1つのトランザクションには、ダウンロード対象のファイルを1つ以上含めることができます。引数は見ての通りです。cell は受信者とのメッセージングに使用します。cell は次のようにして `FLContext` オブジェクトから取得できます。

.. code-block:: python

   engine = fl_ctx.get_engine()
   cell = engine.get_cell()

timeout はトランザクションがいつタイムアウトするかを指定します。これは、そのトランザクション内のいずれのファイルについても、どの受信者からもダウンロード活動が受信されない最大時間です。受信者は分散した存在であるため、それぞれ自分のペースでファイルをダウンロードできます。ある受信者が1つのファイルをダウンロードしている一方で、別の受信者は別のファイルをダウンロードしていることもあります。指定した時間の間、どの受信者もそのトランザクションのどのファイルもダウンロードしていない場合にのみ、トランザクションはタイムアウトしたと見なされます。登録された `timeout_cb` が、そのトランザクションのすべてのファイル名とともに呼び出されます。その後、これらのファイルをどう扱うかを決定できます。

ダウンロードする各ファイルについて `add_file` メソッドを呼び出します。追加したファイルごとにファイル参照 ID (RID) が返されます。その後、RID をメッセージですべての受信者に送信します。

ファイルのダウンロード
------------------------

受信者は RID を受け取ると、データ所有者から参照されたファイルをダウンロードする関数を呼び出します。

.. code-block:: python

   def download_file(
      from_fqcn: str,
      ref_id: str,
      per_request_timeout: float,
      cell: Cell,
      location: str = None,
      secure=False,
      optional=False,
      abort_signal=None,
   ) -> (str, Optional[str]):
      """Download the referenced file from the file owner.

      Args:
          from_fqcn: FQCN of the file owner.
          ref_id: reference ID of the file to be downloaded.
          per_request_timeout: timeout for requests sent to the file owner.
          cell: cell to be used for communicating to the file owner.
          location: dir for keeping the received file. If not specified, will use temp dir.
          secure: P2P private mode for communication
          optional: suppress log messages of communication
          abort_signal: signal for aborting download.

      Returns: tuple of (error message if any, full path of the downloaded file).
      """

引数は見ての通りです。ダウンロードが成功すると、ダウンロードされたファイルのフルパスが取得できます。そのファイルをどう扱うかはあなた次第です。


ファイルストリーミングまたはダウンロードによる大きなオブジェクトのシリアライズ
--------------------------------------------------------------------------------

:ref:`decomposer_for_large_object` を参照してください。
