.. _cyclic:

Cyclic ワークフロー
---------------------------
.. currentmodule:: nvflare.app_common.workflows.cyclic_ctl.CyclicController

cyclic ワークフローは NVIDIA FLARE 2.0 で追加され、Controller API を用いて実装されています。これにより、
:ref:`scatter_and_gather_workflow` における ``broadcast_and_wait()`` の代わりに ``relay_and_wait()`` に基づいた
:meth:`control_flow()<control_flow>` メソッドを通じて、異なる制御フローを実現できます。

:class:`nvflare.app_common.workflows.cyclic_ctl.CyclicController`

Cyclic ワークフローの例
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
cyclic ワークフローを使用したアプリケーションの例については、:github_nvflare_link:`Hello Cyclic Example <examples/hello-world/hello-cyclic>`
を参照してください。
