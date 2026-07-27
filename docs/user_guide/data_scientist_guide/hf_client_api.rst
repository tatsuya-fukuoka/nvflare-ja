.. _hf_client_api:

################################
HuggingFace Client API
################################

HuggingFace Client API を使うと、``flare.receive()`` と ``flare.send()`` を
手動で呼び出すことなく、既存の HuggingFace ``Trainer`` または TRL
``SFTTrainer`` スクリプトを連合化できます。``nvflare.client.hf`` を
インポートしてトレーナーにパッチを適用し、トレーニングループでは
使い慣れた ``trainer.evaluate()`` と ``trainer.train()`` の呼び出しを
そのまま維持します。

ローカルトレーニングスクリプトが既に HuggingFace ``Trainer`` スタイルの
コードを使用しており、FL タスクの交換、グローバル重みの読み込み、
ローカル学習量(バジェット)による停止、チェックポイント状態、
rank-0 通信を FLARE に任せたい場合に、この API を使用してください。

最小限のクライアント変更
================================

通常の HuggingFace トレーナースクリプトから始めます。トレーナーを構築した後、
``flare.patch(trainer)`` を呼び出し、FL ラウンドごとに evaluate/train の
ペアを 1 回実行します:

.. code-block:: python

    import nvflare.client.hf as flare

    trainer = Trainer(
        model=model,
        args=training_args,
        train_dataset=train_dataset,
        eval_dataset=eval_dataset,
    )

    flare.patch(trainer)

    while flare.is_running():
        trainer.evaluate()
        trainer.train()

``nvflare.client.hf`` は標準の Client API シンボルを再エクスポートしているため、
``FLModel``、``get_job_id()``、``log()`` などの Client API 呼び出しにも
同じモジュールを使用できます。その ``is_running()`` 実装は HuggingFace を
認識しており、パッチされたトレーナーの状態と連携して動作します。

``patch()`` が行うこと
==============================

``flare.patch(trainer)`` はトレーナーのメソッドをラップし、FLARE の
コールバックを登録します。各 FL ラウンドで次の処理を行います:

* グローバルプロセスランクで FLARE Client API を初期化する
* rank 0 で FL タスクとグローバルモデルを受信する
* タスクのメタデータとパラメータを他の分散ランクにブロードキャストする
* 受信したパラメータを HuggingFace モデルに読み込む
* 各ローカルラウンドを、設定されたローカルステップまたはエポックの学習量に制限する
* 有効な場合、ラウンド間で Trainer のチェックポイント状態を復元する
* サーバー側のモデル選択のために評価メトリクスを取得する
* rank 0 からトレーニング済みパラメータとメトリクスを FLARE に送り返す

モデルの構築、データセットの読み込み、``TrainingArguments``/``SFTConfig``、
オプティマイザー、スケジューラー、および通常の HuggingFace コールバックは、
引き続きトレーニングスクリプト側が管理します。

パッチオプション
==========================

最もよく使われるオプションは次のとおりです:

``restore_state``
   FL ラウンドをまたいで HuggingFace Trainer の状態を復元するかどうか。
   デフォルトで有効になっており、オプティマイザー、スケジューラー、
   グローバルステップの状態がラウンド間で継続します。有効な場合、
   ``save_only_model=True`` はサポートされません。

``params_scope``
   交換するモデルパラメータの範囲。デフォルトは ``"auto"`` です。PEFT の
   ``PeftModel`` トレーナーではアダプターのみのパラメータを使用し、
   それ以外ではフルモデルのパラメータを使用します。``"model"`` を指定すると
   フルモデルの交換を強制し、``"adapter"`` を指定すると PEFT アダプターの
   交換を必須にします。

``server_key_prefix``
   トレーナーに読み込む際にサーバーパラメータから取り除き、結果を送信する
   際に付け直す、オプションのキープレフィックス。サーバー側のモデルラッパーが
   トレーナーモデルと異なる state-dict 名前空間を持つ場合に使用します。

``local_epochs`` / ``local_steps``
   オプションで明示的に指定するローカルトレーニングの学習量。指定できるのは
   最大 1 つです。どちらも設定しない場合、ラッパーは ``TrainingArguments.max_steps``
   が正の値であればそれを使用し、そうでなければ ``TrainingArguments.num_train_epochs``
   をトレーナーのデータローダーからオプティマイザーステップ数に換算します。

``load_state_dict_strict``
   受信パラメータがローカルモデルのキー空間と厳密に一致しなければならないか
   どうか。意図的に部分的なパラメータ交換を想定している場合を除き、有効の
   ままにしてください。

``stream_metrics``
   rank 0 で有限のスカラー値の HuggingFace ログを FLARE トラッキング API を
   通じてストリーミングするかどうか。

例:

.. code-block:: python

    flare.patch(
        trainer,
        params_scope="auto",
        server_key_prefix="model.",
        local_epochs=1,
        stream_metrics=True,
    )

メトリクスとモデル選択
==============================================

サーバーがローカルトレーニングの前に検証メトリクスを受け取る必要がある場合は、
``trainer.train()`` の前に ``trainer.evaluate()`` を呼び出します:

.. code-block:: python

    while flare.is_running():
        metrics = trainer.evaluate()
        trainer.train()

HuggingFace Client API はトレーナーが返したメトリクスを報告します。
サーバー側のレシピまたはセレクターには、値が大きいほど良いメトリクスキーを
設定してください。レシピにベストモデルを選択させたくない場合は、
``key_metric=""`` を設定します:

.. code-block:: python

    from nvflare.app_opt.pt.recipes.fedavg import FedAvgRecipe

    recipe = FedAvgRecipe(
        name="hf_sft",
        model=model,
        min_clients=2,
        num_rounds=3,
        train_script="client.py",
        launch_external_process=True,
        key_metric="",
    )

サーバーが評価タスクを送信した場合、``trainer.evaluate()`` はトレーニングを
実行せずにメトリクスを送信します。サーバーが ``submit_model`` を要求した場合、
``trainer.evaluate()`` と ``trainer.train()`` のどちらでも、最後に完了した
チェックポイントの提出をトリガーでき、チェックポイントがない場合は現在の
メモリ上のモデルにフォールバックします。

パラメータスコープとキープレフィックス
============================================================================

ほとんどのジョブでは ``params_scope="auto"`` を使用します:

* フルモデルの SFT では、フルモデルのパラメータを使用します。
* ``PeftModel`` を使用する PEFT/LoRA ジョブでは、アダプターパラメータのみを使用します。

``server_key_prefix`` は、サーバー側モデルとトレーナーモデルの state-dict の
キーが異なる場合にのみ使用します。たとえば、サーバー側のラッパーがベースモデルを
``self.model`` に格納している場合、サーバーのキーは ``model.transformer...``
のようになる一方、ローカルトレーナーのキーには ``model.`` が付きません。
その場合は次のようにします:

.. code-block:: python

    flare.patch(trainer, server_key_prefix="model.")

サーバーモデルが既にアダプター形式のキーを公開している PEFT ジョブでは、
プレフィックスは未設定のままにします:

.. code-block:: python

    flare.patch(trainer, params_scope="auto", server_key_prefix=None)

ジョブレシピのセットアップ
====================================

レシピを使用してトレーナースクリプトをパッケージ化し、Client API
エグゼキューターを設定します。LLM ジョブは一般に外部プロセスで実行され、
``torchrun``、``accelerate``、CUDA などのトレーニングランタイム設定を
FLARE クライアントのジョブプロセスから分離した状態に保ちます。

.. code-block:: python

    from nvflare.app_opt.pt.recipes.fedavg import FedAvgRecipe
    from nvflare.client.config import ExchangeFormat
    from nvflare.recipe import SimEnv

    recipe = FedAvgRecipe(
        name="hf_sft",
        model={"class_path": "model.CausalLMModel", "args": {"model_name_or_path": "gpt2"}},
        min_clients=2,
        num_rounds=3,
        train_script="client.py",
        train_args="--model_name_or_path gpt2 --local_epochs 1",
        server_expected_format=ExchangeFormat.PYTORCH,
        launch_external_process=True,
        key_metric="",
    )

    env = SimEnv(num_clients=2)
    recipe.execute(env)

``server_expected_format=ExchangeFormat.PYTORCH`` を使用すると、PyTorch
テンソルを維持し、``bfloat16`` などのテンソルの dtype を保持できます。
サーバー側のワークフローが NumPy 配列を想定している場合は
``ExchangeFormat.NUMPY`` を使用します。半精度テンソルは NumPy サーバーに
送信される前に ``float32`` にキャストされます。

分散トレーニング
==========================

マルチ GPU またはマルチノードでの HuggingFace トレーニングでは、
``flare.patch(trainer)`` を呼び出す前に、``torch.distributed`` を初期化して
グローバルな ``RANK``/``WORLD_SIZE`` を設定する ``torchrun`` などのランチャーで
スクリプトを起動します。

すべてのランクは、同じパッチ済み Trainer メソッドを同じ順序で呼び出す必要があります:

.. code-block:: python

    while flare.is_running():
        trainer.evaluate()
        trainer.train()

FLARE Client API の receive/send パスを呼び出すのは rank 0 のプロセスだけです。
他のランクは ``torch.distributed`` のブロードキャスト経由でタスクデータを
受け取ります。同じタスクに対して、あるランクが ``trainer.evaluate()`` を呼び、
別のランクが ``trainer.train()`` を呼んだ場合、ラッパーは分散デッドロックを
許容する代わりにエラーを送出します。

分散パラメータのペイロードはデフォルトでメモリ内オブジェクトブロードキャストを
使用するため、パラメータの受け渡しに共有ストレージは不要です。サイズしきい値を
超えた場合の自動ファイルステージングを有効化するには、次を設定します:

.. code-block:: bash

    export NVFLARE_HF_PARAMS_EXCHANGE_STRATEGY=auto

``NVFLARE_HF_PARAMS_EXCHANGE_STRATEGY=file`` でファイルステージングを強制する
こともできます。どちらのファイルモードも rank-0 のパラメータを
``<output_dir>/_fl_exchange`` 以下にステージングし、すべてのランクが
``TrainingArguments.output_dir`` を共有している必要があります。``auto`` の
しきい値は ``NVFLARE_HF_PARAMS_FILE_EXCHANGE_MIN_BYTES`` で制御されます。
これとは別に、``restore_state=True`` のチェックポイント再開には、すべての
ランクが ``output_dir`` 以下の同じチェックポイントパスを参照できることが
必要です。

チェックポイントの動作
============================================

``restore_state=True`` の場合:

* ラッパーは、2 ラウンド目以降では FLARE が最後に記録した HuggingFace
  チェックポイントから再開します
* ``TrainingArguments.save_total_limit`` が未設定の場合は ``2`` に設定され、
  現在と直前の FL チェックポイントを保持できるようにします
* ``load_best_model_at_end=True`` は拒否されます。グローバルなベストモデルの
  選択は FL サーバーの責務であるためです
* ``save_only_model=True`` は拒否されます。Trainer の再開にはオプティマイザーと
  スケジューラーの状態が必要であるためです

``trainer.train()`` に独自の ``resume_from_checkpoint`` を渡した場合、FLARE は
その train 呼び出しにそのチェックポイントを使用し、ユーザー提供の
チェックポイントを書き換える代わりに、受信したグローバル重みをメモリ内で
適用します。

ラッパーは、パッチされたトレーナープロセスについて最後のチェックポイントパスを
メモリ内に保持します。FL チェックポイントの来歴をディスクに永続化することは
なく、ランチャーが既に失敗として報告した実行中トレーニングタスクの透過的な
リカバリも提供しません。Phase 1 では、``restore_state=True`` は単一の
トレーナープロセスのライフサイクルを必要とし、明示的に設定された
``launch_once=False`` は拒否されます。タスクごとにトレーナーを起動する場合は
``restore_state=False`` を使用してください。

``restore_state=False`` は、FL ラウンドごとに Trainer 管理のオプティマイザーと
スケジューラーのインスタンスを新規作成します。パッチ適用時点で構築済みの
``optimizer`` または ``lr_scheduler`` インスタンスを持つ Trainer は拒否されます。
これらのインスタンスを黙って破棄すると、ユーザーが設定したトレーニングの
セマンティクスが変わってしまうためです。構築済みインスタンスを使う場合は
``restore_state=True`` を使用するか、Trainer に構築させてください。

デフォルトでは、FLARE は検証済みの ``transformers`` バージョンに対して
メモリ内グローバル重みオーバーライドを使用し、それ以外の場合は
チェックポイントインジェクションにフォールバックします。ご自身の環境を
検証した上でこの選択を上書きするには、``NVFLARE_HF_WEIGHT_OVERRIDE_STRATEGY``
を次のいずれかに設定します:

``auto``
   検証済みバージョンのゲートを使用します。これがデフォルトです。

``in_memory``
   前回のチェックポイントから再開し、``on_train_begin`` の間に受信した
   グローバル重みをメモリ内で適用します。これにより、ラウンドごとにモデル重みを
   チェックポイントストレージに書き直すことを回避できます。

``checkpoint_injection``
   再開前に、受信したグローバル重みを記録済みチェックポイントに書き込みます。
   メモリ内オーバーライドが、インストールされている HuggingFace スタックや
   分散バックエンドと互換性がない場合に使用します。

サポートされない構成
========================================

HuggingFace Client API の最初の実装では、意図的に以下をサポートしていません:

* DeepSpeed
* FSDP
* ``load_best_model_at_end=True``
* ``restore_state=True`` と組み合わせた ``save_only_model=True``
* 同一 Python プロセス内で複数のパッチ済み HuggingFace ``Trainer``

トラブルシューティング
============================================

``Previous HuggingFace FL task ... is still pending``
   現在のタスクを完了する前に、ループが再度 ``flare.is_running()`` を
   呼び出しました。先に想定されている ``trainer.train()`` または
   ``trainer.evaluate()`` を呼び出してください。

``Divergent HuggingFace Trainer call across ranks``
   分散ランクが異なる Trainer メソッドを呼び出しました。すべてのランクに
   同じ evaluate/train シーケンスを実行させてください。

``rank > 0, but torch.distributed is not initialized``
   分散プロセスグループが初期化されていない状態で、0 以外のグローバルランクが
   検出されました。分散ジョブは ``torchrun`` で起動するか、単一プロセス実行の
   場合は古い ``RANK`` 環境変数を削除してください。

``None of the model parameters matched``
   サーバーとトレーナーの state-dict のキー空間が異なります。正しい
   ``server_key_prefix`` を設定するか、``params_scope`` を調整してください。

完全なサンプル
==========================

完全な HuggingFace SFT/PEFT のサンプルは次を参照してください:

* :github_nvflare_link:`examples/hello-world/hello-huggingface <examples/hello-world/hello-huggingface>`

Client API の一般的な概念については :ref:`client_api_usage` を参照してください。
