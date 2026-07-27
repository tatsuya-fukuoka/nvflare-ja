.. _cross_site_model_evaluation:

クロスサイトモデル評価／フェデレーテッド評価
--------------------------------------------------------------
:class:`クロスサイトモデル評価ワークフロー<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>`
は、クライアントのデータを使用して他のクライアントのモデルによる評価を実行します。
データは共有されず、代わりにモデルの集合が各クライアントサイトへ配布され、ローカル検証が実行されます。ローカル検証の
結果はサーバによって収集され、モデル性能とクライアントデータセットの全対全の行列が構築されます。

サーバのグローバルモデルも各クライアントへ配布され、クライアントのローカルデータセット上でグローバルモデルの評価が
行われます。

:github_nvflare_link:`hello-numpy-cross-val example <examples/hello-world/hello-numpy-cross-val>` は、
:class:`クロスサイトモデル評価ワークフロー<nvflare.app_common.workflows.cross_site_model_eval.CrossSiteModelEval>` を実装したシンプルな例です。

.. note::

   バージョン 2.0 より前の NVFlare では、クロスサイト検証はフレームワーク自体に組み込まれており、クロスサイト検証の
   結果を取得するための管理コマンドがありました。NVFlare 2.0 では、ワークフローをカスタマイズできるようになったことに伴い、
   クロスサイト検証は NVFlare フレームワークには含まれず、代わりにワークフローによって処理されます。
   :github_nvflare_link:`cifar10 example <examples/advanced/cifar10>` はクロスサイトモデル評価を実行するように
   構成されており、``config_fed_server.json`` は :class:`ValidationJsonGenerator<nvflare.app_common.widgets.validation_json_generator.ValidationJsonGenerator>`
   を用いて結果をサーバ上の JSON ファイルへ書き出すように構成されています。

クロスサイトモデル評価／フェデレーテッド評価ワークフローの例
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
サーバ制御によるクロスサイト評価ワークフローを使用する例については、:github_nvflare_link:`Hello Numpy Cross-Site Validation <examples/hello-world/hello-numpy-cross-val>` を参照してください。
