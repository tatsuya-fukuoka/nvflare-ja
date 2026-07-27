.. _data_exchange_object:

Data Exchange Object (DXO)
==========================
.. currentmodule:: nvflare.apis.dxo.DXO

NVIDIA FLARE の Data Exchange Format(:class:`nvflare.apis.dxo.DXO`)は、通信する当事者間で受け渡されるデータを標準化します。

.. literalinclude:: ../../nvflare/apis/dxo.py
    :language: python
    :lines: 54-76

``data_kind`` は、データの種類(例: "WEIGHTS" や "WEIGHT_DIFF")を管理します。

``meta`` は、追加のプロパティを含めることができる dict です。

:meth:`to_shareable()<to_shareable>` メソッドは :ref:`shareable` を生成し、
:meth:`nvflare.apis.dxo.from_shareable` を使うと :ref:`shareable` から DXO を取り出せます。

FL システム全体でデータ管理の一貫性を保つために、DXO の使用を推奨します。
