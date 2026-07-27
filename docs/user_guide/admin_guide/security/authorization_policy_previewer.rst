:orphan:

.. deprecated:: 2.7
   このツールは非推奨です。

.. _authorization_policy_previewer:

******************************
認可ポリシープレビューア
******************************

:ref:`認可 <federated_authorization>` は NVFLARE の重要なセキュリティ機能です。NVFLARE 2.2 以降、各サイトは独自の認可ポリシーを定義します。
認可ポリシーはシステムのセキュリティにとって不可欠であり、また多くの人がポリシーを定義できるようになったため、
本番環境にデプロイする前にポリシーを検証できることが重要です。

認可ポリシープレビューアは、認可ポリシーの定義を検証するためのツールです。このツールは、ポリシー定義のさまざまな側面を
検証するための対話型ユーザーインターフェースとコマンドを提供します。

    - 定義されているロールと権限を表示する
    - ポリシー定義の内容を表示する
    - パーミッションマトリクス (ロール/権限/条件) を表示する
    - 指定したユーザーに対して権限を評価する

認可ポリシープレビューアの起動
======================================
認可ポリシープレビューアを起動するには、ターミナルで次のコマンドを入力します。

.. code-block:: shell

  nvflare authz_preview -p <authorization_policy_file>

authorization_policy_file は、認可ファイルの形式に従った JSON ファイルである必要があります。

ファイルが有効な JSON ファイルでない場合、または認可ファイルの形式に従っていない場合、このコマンドは例外を発生させて終了します。

認可ポリシープレビューアのコマンドの実行
------------------------------------------------
認可ポリシープレビューアが正常に起動すると、コマンド入力用のプロンプト ``>`` が表示されます。

コマンドの完全な一覧を取得するには、プロンプトで "?" を入力します。

ほとんどのコマンドは説明不要ですが、"eval_right" は例外です。このコマンドを使うと、指定した権限を指定したユーザー
(name:org:role) に対して評価し、結果が正しいことを確認できます。

ロールの権限
--------------------
ポリシーファイル内のほとんどのパーミッションは、コマンドカテゴリを使って定義できます。ただし、ポリシーファイルが読み込まれると、
カテゴリはフォールバックメカニズムに従ってすでに個々のコマンドへと解決されています。

``show_role_rights command`` を使用して、すべてのロールについてすべてのコマンドが正しいパーミッションを持っていることを
確認してください。

権限の評価
----------------
``eval_right`` コマンドの構文は次のとおりです。

.. code-block:: shell

  eval_right site_org right_name user_name:org:role [submitter_name:org:role]

各項目の意味は次のとおりです。

.. code-block::

    site_org - the organization of the site
    right_name - the right to be evaluated. You can use the "show_rights" command to list all available commands.
    User specification - a user spec has three pieces of information separated by colons. Name is the name of the user; org is the organization that the user belongs to; and role is the user's role. You can use the "show_roles" command to list all available roles.
    Submitter specification - some job related commands can evaluate the relation between the user and the submitter of a job. Submitter spec has the same format as user spec.

権限の定義と評価の詳細については、 :ref:`Federated Authorization <federated_authorization>` を参照してください。

認可ポリシープレビューアの停止
--------------------------------------
認可ポリシープレビューアを終了するには、プロンプトで "bye" コマンドを入力します。
