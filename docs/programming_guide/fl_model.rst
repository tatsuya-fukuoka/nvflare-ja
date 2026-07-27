.. _fl_model:

FLModel
=======

私たちは、学習結果の交換に必要な共通の属性を捉えた標準データ構造 :mod:`FLModel<nvflare.app_common.abstract.fl_model>` を定義しています。

これは、NVFlareシステムが外部のトレーニングスクリプトやシステムと学習情報を交換する必要がある場合に特に有用です。

外部のトレーニングスクリプトやシステムは、受信したFLModelから必要な情報を抽出し、ローカルトレーニングを実行し、結果を新しいFLModelに格納して送り返すだけで済みます。

各属性の詳細な説明については、APIドキュメントを参照してください:
:mod:`FLModel<nvflare.app_common.abstract.fl_model>`
