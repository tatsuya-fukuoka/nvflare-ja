.. _contributing:

コントリビューション
========================

NVIDIA FLARE へようこそ！皆さんの参加と貢献を心より歓迎します。この
ドキュメントは、NVIDIA FLARE への貢献に関心のある個人および組織を対象と
しています。NVIDIA FLARE はオープンソースプロジェクトであり、その成功は
改善を続けようとするコントリビューターのコミュニティに支えられています。
皆さんの貢献はコードベースへの貴重な追加となります。経験豊富なオープン
ソースコントリビューターの方でも、初めて貢献される方でも、このページを
読んで私たちのコントリビューションプロセスを理解していただくようお願い
します。

私たちとのコミュニケーション
------------------------------------

NVIDIA FLARE に対するニーズやプロジェクトへの貢献のアイデアについて、
喜んでお話しします。その方法のひとつは、考えを議論する issue を作成する
ことです。よく似た機能が開発中であったり、すでに存在していたりする可能性
もあるため、issue は素晴らしい出発点になります。

コントリビューションプロセス
------------------------------------

*早めのプルリクエスト*

プルリクエストは早めに作成することをお勧めします。マージの準備ができて
いるかどうかにかかわらず、開発中のコントリビューションを追跡するのに
役立ちます。正式なレビューの準備が整うまでは、プルリクエストのタイトルを
``[WIP]`` で始めるか、`ドラフトプルリクエストを作成
<https://docs.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-requests#draft-pull-requests>`__\ してください。

プルリクエストの準備
----------------------------

コード品質を確保するため、NVIDIA FLARE はいくつかの lint ツール
(`flake8 とそのプラグイン <https://gitlab.com/pycqa/flake8>`__、
`black <https://github.com/psf/black>`__、
`isort <https://github.com/timothycrosley/isort>`__)を利用しています。

このセクションでは、プルリクエストを送る前に必要なすべての準備手順を
説明します。効率的にコラボレーションするために、このセクションを読んで
従ってください。

-  `コーディングスタイルのチェック <#checking-the-coding-style>`__
-  `ユニットテスト <#unit-testing>`__
-  `ドキュメントのビルド <#building-the-documentation>`__
-  `作業への署名 <#signing-your-work>`__

コーディングスタイルのチェック
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

コードスタイルは flake8 と isort でチェックします。すべてのテストを
ローカルで実行するための bash スクリプト(``runtest.sh``)が用意されて
います。

ライセンス情報: すべてのソースコードファイルは、次の段落で始まる必要が
あります。

::

   # Copyright (c) 2021-2022, NVIDIA CORPORATION.  All rights reserved.
   #
   # Licensed under the Apache License, Version 2.0 (the "License");
   # you may not use this file except in compliance with the License.
   # You may obtain a copy of the License at
   #
   #     http://www.apache.org/licenses/LICENSE-2.0
   #
   # Unless required by applicable law or agreed to in writing, software
   # distributed under the License is distributed on an "AS IS" BASIS,
   # WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
   # See the License for the specific language governing permissions and
   # limitations under the License.

ユニットテスト
^^^^^^^^^^^^^^^^^^^^

NVIDIA FLARE のテストは test/ 以下にあります。ユニットテストのファイル名は
``test_[module_name].py`` のパターンに従います。

bash スクリプト ``runtest.sh`` はユニットテストも実行します。

ドキュメントのビルド
^^^^^^^^^^^^^^^^^^^^^^^^^^

ドキュメントをビルドするには、まずすべての必要要件が揃っていることを
確認してください。

.. code:: bash

   python -m pip upgrade
   python -m pip install -e .[doc]

ドキュメントをビルドするには、次を実行してください。

.. code:: bash

   ./build_doc.sh --html

ビルドが完了すると、``docs/_build folder`` でドキュメントを閲覧できます。
ドキュメントをクリーンアップするには、次を実行してください。

.. code:: bash

   ./build_doc.sh --clean

作業への署名
^^^^^^^^^^^^^^^^^^

NVIDIA FLARE は、すべてのプルリクエストに対して `Developer Certificate of
Origin <https://developercertificate.org/>`__\ (DCO)を義務付けています。

コミットへの署名の詳細なガイドについては、GitHub の `Signing
commits <https://docs.github.com/en/authentication/managing-commit-signature-verification/signing-commits>`__
を参照してください。

コミット署名の検証
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

NVIDIA FLARE は、GitHub が提供するセキュリティ機能であるコミット署名の
検証を義務付けています。開発者は `Commit Signature
Verification <https://docs.github.com/en/authentication/managing-commit-signature-verification/about-commit-signature-verification#gpg-commit-signature-verification>`__
に記載されている手順に従って GPG キーをセットアップする必要があります。

DCO の全文:

::

   Developer Certificate of Origin
   Version 1.1

   Copyright (C) 2004, 2006 The Linux Foundation and its contributors.
   1 Letterman Drive
   Suite D4700
   San Francisco, CA, 94129

   Everyone is permitted to copy and distribute verbatim copies of this
   license document, but changing it is not allowed.


   Developer's Certificate of Origin 1.1

   By making a contribution to this project, I certify that:

   (a) The contribution was created in whole or in part by me and I
       have the right to submit it under the open source license
       indicated in the file; or

   (b) The contribution is based upon previous work that, to the best
       of my knowledge, is covered under an appropriate open source
       license and I have the right under that license to submit that
       work with modifications, whether created in whole or in part
       by me, under the same open source license (unless I am
       permitted to submit under a different license), as indicated
       in the file; or

   (c) The contribution was provided directly to me by some other
       person who certified (a), (b) or (c) and I have not modified
       it.

   (d) I understand and agree that this project and the contribution
       are public and that a record of the contribution (including all
       personal information I submit with it, including my sign-off) is
       maintained indefinitely and may be redistributed consistent with
       this project or the open source license(s) involved.

プルリクエストの提出
----------------------------

``main`` ブランチへのすべてのコード変更は、`プルリクエスト
<https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/proposing-changes-to-your-work-with-pull-requests>`__
を通して行う必要があります。
1. 新しいチケットを作成するか、`issue リスト
<https://github.com/NVIDIA/NVFlare/issues>`__ から既知のチケットを
引き受けます。 2. そのタスク専用のブランチがすでに存在しないか確認します。
3. タスクがまだ着手されていなければ、コードベースの\ `フォークに新しい
ブランチを作成
<https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/creating-a-pull-request-from-a-fork>`__\ します。
新しいブランチは、最新の ``main`` ブランチをベースにするのが理想的です。
4. ブランチに変更を加えます(`可能であれば詳細なコミットメッセージを使用
<https://chris.beams.io/posts/git-commit/>`__)。 5.
新しいテストが変更をカバーしていること、変更後のコードベースが
`ローカルですべてのテストにパスすること <#unit-testing>`__ を確認します。
6. タスクブランチから ``main`` ブランチへ、このプルリクエストの目的の
詳細な説明を添えて\ `新しいプルリクエストを作成
<https://help.github.com/en/desktop/contributing-to-projects/creating-a-pull-request>`__\ します。
7. `プルリクエストの CI/CD ステータス
<https://github.com/NVIDIA/NVFlare/actions>`__ を確認し、すべての CI/CD
テストがパスしていることを確認します。 8. レビュアーを2名アサインします。
レビュアーのうち1名は、該当コード領域のコードオーナーでなければなりません。
9. レビューを待ちます。レビューがあれば、一つひとつに応答し、必要に応じて
さらにコードを変更します。 10. プルリクエストのブランチと ``main``
ブランチの間にコンフリクトがある場合は、``main`` から変更を取り込み、
ローカルでコンフリクトを解消します。 11. すべてのコメントが解決されるまで、
レビュアーとコントリビューターの間で議論が往復することがあります。PR が
パスするには、すべての会話が解決されている必要があります。 12.
プルリクエストがマージされるのを待ちます。

プルリクエストのレビュー
--------------------------------

すべてのコードレビューコメントは、具体的、建設的、かつ実行可能なもので
なければなりません。 1. `プルリクエストの CI/CD ステータス
<https://github.com/NVIDIA/NVFlare/actions>`__ を確認し、レビューの前に
すべての CI/CD テストがパスしていることを確認します(必要に応じてブランチ
オーナーに連絡します)。 1. プルリクエストの説明と変更されたファイルを
注意深く読み、必要に応じてコメントを書きます。 1. 特定のコードセグメントに
インラインコメントを付け、必要に応じて\ `変更をリクエスト
<https://help.github.com/en/github/collaborating-with-issues-and-pull-requests/about-pull-request-reviews>`__\ します。
1. コントリビューターがすべてのコメントに対応するまで、追加のコード変更を
レビューします。 1. プルリクエストを main ブランチにマージします。 1.
`issue リスト <https://github.com/NVIDIA/NVFlare/issues>`__ 上の対応する
タスクチケットをクローズします。
