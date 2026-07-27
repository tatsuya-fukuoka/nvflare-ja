.. _configurations:

###################
設定ファイル
###################

**サポートされる設定ファイル形式**

- `JSON <https://www.json.org/json-en.html>`_
- `YAML <https://yaml.org/>`_
- `Pyhocon <https://github.com/chimpler/pyhocon>`_ - JSON の変種であり、Python 向けの HOCON (Human-Optimized Config Object Notation) パーサーです。
  コメント、変数の置換、および継承をサポートします。
- `OmegaConf <https://omegaconf.readthedocs.io/en/2.3_branch/>`_ - YAML ベースの階層的な設定です。

ユーザーは単一の形式を使用することも、config_fed_client.conf と config_fed_server.json を併用する例のように、複数の形式を組み合わせることもできます。
複数の設定形式が共存する場合、それらの使用は次の検索順序に基づいて優先されます。

``.json -> .conf -> .yml -> .yaml``

設定ファイルのさまざまな機能と種類に関する、より詳しい情報については以下のセクションを参照してください。

.. toctree::
   :maxdepth: 1

   variable_resolution
   job_configuration
   server_port_consolidation
   communication_configuration
   logging_configuration
   site_config
