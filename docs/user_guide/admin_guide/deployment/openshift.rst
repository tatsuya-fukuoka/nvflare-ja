.. _openshift_k8s_deployment:

##################################
OpenShift への FLARE のデプロイ
##################################

OpenShift のデプロイガイドとヘルパースクリプトは、現在 DevOps の
サンプルディレクトリに置かれています。

``examples/devops/openshift``

はじめに参照する場所
=====================

まず ``examples/devops/openshift/README.md`` を開いて、フォルダの概要を簡潔に
確認してください。Dockerfile、ヘルパースクリプト、および代表的なクイックスタート
コマンドが記載されています。

OpenShift デプロイガイドの全文については ``examples/devops/openshift/index.md``
を開いてください。この文書では、前提条件、イメージの要件、スクリプト化された
ワークフロー、手動でのデプロイ手順、OpenShift SCC に関する注意事項、
トラブルシューティング、およびクリーンアップについて説明しています。

スクリプトは NVFlare リポジトリのルートから実行してください。例:

.. code-block:: bash

   bash examples/devops/openshift/scripts/k8s_e2e.sh

OpenShift のサンプルは、汎用的な Kubernetes デプロイのランタイムを基盤としています。
Kubernetes Helm チャートのワークフローとランタイムの詳細については :ref:`helm_chart` を参照してください。
