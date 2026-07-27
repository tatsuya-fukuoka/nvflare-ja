.. _serialization:

メッセージのシリアライゼーション
================================
NVFLARE は、サーバーとクライアントの間でデータをやり取りする際のメッセージのシリアライゼーションおよび
デシリアライゼーションに、FOBS (Flare OBject Serializer) と呼ばれる安全な仕組みを使用します。


Flare Object Serializer (FOBS)
------------------------------


概要
~~~~~~~~

FOBS はセキュリティ上の目的で用意された Pickle の代替 (ドロップイン置換) です。オブジェクトの
シリアライズには **MessagePack** を使用します。

FOBS は利便性を犠牲にしてセキュリティを確保しています。Pickle ではイントロスペクションによって
ほとんどのオブジェクトが自動的にサポートされます。FOBS でオブジェクトをシリアライズするには、
そのクラス用の **Decomposer** を登録しておく必要があります。よく使われるいくつかのクラスに対する
decomposer は、モジュールにあらかじめ登録されています。

FOBS は、 :code:`Enum` のサブクラスであるすべてのクラスに対して decomposer を自動的に登録することで、
enum 型をサポートします。

FOBS はそれ以外のすべてのクラスを dataclass として扱い、dataclass 用の汎用 decomposer を登録します。
dataclass とは、コンストラクタが副作用なしにオブジェクトの状態のみを変更するクラスのことです。
副作用には、グローバル変数の変更、ネットワーク接続の作成、ファイルの作成などが含まれます。

FOBS は、decomposer が登録されていないオブジェクトに遭遇すると :code:`TypeError` 例外を送出します。
例えば、次のようになります。

.. code-block::

    TypeError: cannot serialize 'xxx' object

使い方
~~~~~~~~

FOBS は Pickle と同様に、以下の 4 つの関数を定義しています。

* :code:`dumps(obj)`: obj をシリアライズして bytes を返します
* :code:`dump(obj, stream)`: obj をシリアライズして結果を stream に書き込みます
* :code:`loads(data)`: data をデシリアライズしてオブジェクトを返します
* :code:`load(stream)`: stream からデータを読み取り、オブジェクトにデシリアライズします


例を示します。

.. code-block::

    from nvflare.fuel.utils import fobs

    data = fobs.dumps(dxo)
    new_dxo = fobs.loads(data)

    # Pickle/json compatible functions can be used also
    data = fobs.dumps(shareable)
    new_shareable = fobs.loads(data)

デコンポーザー
~~~~~~~~~~~~~~~~

デコンポーザー (decomposer) は、抽象基底クラス :code:`fobs.Decomposer` を継承したクラスです。FOBS は
MessagePack を使ってオブジェクトをシリアライズする前に、decomposer を使ってオブジェクトを
**シリアライズ可能なオブジェクト** に分解します。

decomposer はシリアライザーによく似ていますが、オブジェクトを直接 bytes に変換する必要はなく、
シリアライズ可能な別のオブジェクトへ分解するだけでよい点が異なります。

オブジェクトは、その型が MessagePack でサポートされているか、あるいはそのクラス用の decomposer が
登録されている場合に、シリアライズ可能となります。

FOBS は、すべてのオブジェクトが MessagePack でサポートされる型になるまで、再帰的にオブジェクトを
分解します。分解のループはスタックオーバーフローを引き起こすため、避けなければなりません。あるクラスが
別のクラスに分解され、それが最終的に元のクラスに分解される場合、decomposer はループを形成します。
例えば、次のシナリオが最も単純なループです。X が Y に分解され、Y が X に分解し戻される場合です。

MessagePack は以下の型をネイティブにサポートしています。

* None
* bool
* int
* float
* str
* bytes
* bytearray
* memoryview
* list
* dict

以下のクラスに対する decomposer は `fobs` モジュールに含まれており、自動的に登録されます。

* tuple
* set
* OrderedDict
* datetime
* Shareable
* FLContext
* DXO
* Client
* RunSnapshot
* Workspace
* Signal
* AnalyticsDataType
* argparse.Namespace
* Learnable
* _CtxPropReq
* _EventReq
* _EventStats
* numpy.float32
* numpy.float64
* numpy.int32
* numpy.int64
* numpy.ndarray

:code:`fobs/decomposers` フォルダに定義されているすべてのクラスは自動的に登録されます。
それ以外の decomposer は、次のように手動で登録する必要があります。

.. code-block::

    fobs.register(FooDecomposer)
    fobs.register(BarDecomposer())


:code:`fobs.register` は、引数としてクラスまたはインスタンスのいずれかを受け取ります。コンストラクタが
引数を取る decomposer は、インスタンスとして登録する必要があります。

decomposer は、クラスを bytes にシリアライズすることも、シリアライズ可能な型のオブジェクトに分解する
こともできます。ほとんどの場合、メンバーをリストとして保存し、そのリストからオブジェクトを再構築する
だけで済みます。

MessagePack は dict の中で 4GB を超える項目を扱えません。この問題を回避するため、FOBS は大きな項目を
外部化し、バッファには参照のみを保存できます。外部化されたデータの処理には :code:`DatumManager` が
使用されます。4GB を超える dict 項目を扱わないほとんどのオブジェクトでは、DatumManager は不要です。

以下は単純な decomposer の例です。 :code:`datetime` は MessagePack でサポートされていませんが、
`fobs` モジュールに decomposer が含まれているため、それ以上分解する必要はありません。

.. code-block::

    from nvflare.fuel.utils import fobs


    class Simple:

        def __init__(self, num: int, name: str, timestamp: datetime):
            self.num = num
            self.name = name
            self.timestamp = timestamp


    class SimpleDecomposer(fobs.Decomposer):

        def supported_type(self) -> Type[Any]:
            return Simple

        def decompose(self, obj, manager) -> Any:
            return [obj.num, obj.name, obj.timestamp]

        def recompose(self, data: Any, manager) -> Simple:
            return Simple(data[0], data[1], data[2])


    fobs.register(SimpleDecomposer)
    data = fobs.dumps(Simple(1, 'foo', datetime.now()))
    obj = fobs.loads(data)
    assert obj.num == 1
    assert obj.name == 'foo'
    assert isinstance(obj.timestamp, datetime)


同じ decomposer を複数回登録することもできます。有効になるのは最初の 1 つだけで、それ以外は警告
メッセージとともに無視されます。

なお、decomposer が登録されていない場合は ``fobs_initialize()`` の呼び出しが必要になることがあります。

Enum 型
~~~~~~~~~~

:code:`Enum` から派生したすべてのクラスは、すでに登録済みのデフォルトの enum decomposer によって
自動的に処理されます。
つまり、これらの enum に対して手動で何かを設定する必要はありません。
シリアライゼーションとデシリアライゼーションのサポートが組み込みで提供されます。

まれに、 :code:`Enum` から派生したクラスが複雑すぎて汎用の decomposer では処理できない場合には、
専用の decomposer を作成して登録できます。
これにより、FOBS がそのクラスに対して汎用 decomposer を使用しないようにできます。

Dataclass 型
~~~~~~~~~~~~~~~

すべての dataclass は、すでに登録済みのデフォルトの dataclass decomposer によって自動的に処理されます。
つまり、これらのクラスに対して手動で何かを設定する必要はありません。

dataclass の例を示します。

.. code-block:: python

    from dataclasses import dataclass

    @dataclass
    class Student:
        name: str
        height: int

カスタム型
~~~~~~~~~~~~

FOBS でカスタム型をサポートするには、その型に対する decomposer をカスタムコードに含め、登録する必要が
あります。

decomposer は、FOBS を使用する前にサーバー側とクライアント側の両方のコードで登録しなければなりません。
登録場所としては、コントローラーやエグゼキューターのコンストラクタが適しています。 ``START_RUN``
イベントハンドラーで行うこともできます。

カスタムオブジェクトを ``shareable`` に直接入れることはできません。
まず FOBS を使ってシリアライズする必要があります。 ``custom_data`` がカスタム型を含むとすると、
shareable にデータを格納する方法は次のとおりです。

::

    shareable[CUSTOM_DATA] = fobs.dumps(custom_data)

受信側では次のようにします。

::

    custom_data = fobs.loads(shareable[CUSTOM_DATA])

以下は正しく動作しません。

::

    shareable[CUSTOM_DATA] = custom_data


FOBS でカスタム型を使用する場合は、 ``CustomType`` クラスのような各カスタム型を、アプリケーション
ディレクトリの custom フォルダ内でそれぞれ独立したファイルに配置してください。
