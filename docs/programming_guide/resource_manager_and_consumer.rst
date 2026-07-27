.. _resource_manager_and_consumer:

#############################################
リソースマネージャーとリソースコンシューマー
#############################################
NVFlare はバージョン 2.1 で :ref:`job`、リソースマネージャー、リソースコンシューマーという概念を導入しました。

各ジョブには meta.json があり、"deploy_map"、"min_clients"、"mandatory_clients"、"resource_spec" を指定できます。

ユーザーは meta.json でジョブのリソース要件を指定し、対応するリソースマネージャーとリソースコンシューマーを設定できます。


:ref:`ジョブスケジューリング <job_scheduler_configuration>` の際に、サーバー側は各クライアントにリソース要件を満たせるかどうかを問い合わせます。各クライアントは、設定されたリソースマネージャーの check_resources メソッドを呼び出します。ジョブ設定で指定された "resource_spec" が引数として渡されます。check_resources は、ローカルのクライアントサイトのリソースがこのジョブを実行するのに十分かどうかを判断する必要があります。十分であればこのジョブはスケジュールされ、そうでなければジョブはキューに留まります。

なお、このチェックは完全にリソースマネージャーによって行われるため、リソースマネージャーが FL サーバーにリソースが十分であると伝えた場合、NVFlare はそのサイトでジョブを開始してよいと想定します。

NVFlare の外部にある別のプロセスがリソースを占有してリソースが利用できなくなった場合、実行時にジョブの実行が失敗する可能性があります。

ジョブのリソース要件を指定する方法
===================================
ジョブの概念により、ユーザーはこのジョブが実行時にどれだけのリソースを必要とするかを指定できます。

リソース仕様は、サイト名を必要なリソースにマッピングする dict です。例えば次のようになります。

.. code-block::

    {
        "resource_spec": {
            "site-1": { "num_of_gpus": 1, "mem_per_gpu_in_GiB": 1 },
            "site-2": { "num_of_gpus": 1, "mem_per_gpu_in_GiB": 1 }
        }
    }

完全な meta.json は次のようになります。

.. code-block::

    {
        "name": "hello-pt",
        "resource_spec": {
            "site-1": { "num_of_gpus": 1, "mem_per_gpu_in_GiB": 1 },
            "site-2": { "num_of_gpus": 1, "mem_per_gpu_in_GiB": 1 }
        },
        "min_clients" : 2,
        "deploy_map": {
            "app": [
                "@ALL"
            ]
        }
    }

リソースマネージャーとリソースコンシューマーを設定する方法
============================================================

各サイトは、"local" フォルダー内の "resources.json" を使って、独自のリソースマネージャーとリソースコンシューマーを設定できます。

例えば、POC におけるデフォルトは次のようになります。

.. code-block::

    {
        "format_version": 2,
        "client": {
            "retry_timeout": 30,
            "compression": "Gzip"
        },
        "components": [
            {
                "id": "resource_manager",
                "path": "nvflare.app_common.resource_managers.gpu_resource_manager.GPUResourceManager",
                "args": { "num_of_gpus": 1, "mem_per_gpu_in_GiB": 4 }
            },
            {
                "id": "resource_consumer",
                "path": "nvflare.app_common.resource_consumers.gpu_resource_consumer.GPUResourceConsumer",
                "args": {}
            }
        ]
    }

これは、このサイトが GPU を 1 基持ち、GPU あたりのメモリが 1 GiB であることを指定していることを意味します。
GPU をまったく持っていない場合は、num_of_gpus と mem_per_gpu_in_GiB を 0 に設定できます。

GPUResourceManager (:mod:`nvflare.app_common.resource_managers.gpu_resource_manager`) および GPUResourceConsumer (:mod:`nvflare.app_common.resource_consumers.gpu_resource_consumer`)

.. note::

    引数は異なっていてもかまいませんが、各クライアントが同じリソースマネージャークラスとリソースコンシューマークラスを持つようにしてください。

GPUResourceManager と GPUResourceConsumer
==========================================

初期化中に、GPUResourceManager は (nvidia-smi を使用して) 管理対象の GPU 数とメモリが十分かどうかを自動的に検出します。``CUDA_VISIBLE_DEVICES`` に GPU ID が含まれている場合、起動時のチェックはそれらの GPU に限定されます。

なお、現在の GPUResourceManager の実装は、GPU 数とメモリ使用量を継続的に更新しません。つまり、初期化時に nvidia-smi を使ってチェックするだけで、その後はサイト上にこれだけのリソースがあると仮想的に想定します。

(GPUResourceManager の初期化後に) NVFlare の外部の別のプロセスが GPU リソースを占有した場合、GPUResourceManager はその責任を負いません。


独自のリソースマネージャーとリソースコンシューマーを書く方法
==============================================================

以下の API 仕様に従って、独自のリソースマネージャーとリソースコンシューマーを簡単に書くことができます。

.. code-block:: python

    class ResourceConsumerSpec(ABC):
        @abstractmethod
        def consume(self, resources: dict):
            pass


    class ResourceManagerSpec(ABC):
        @abstractmethod
        def check_resources(self, resource_requirement: dict, fl_ctx: FLContext) -> Tuple[bool, str]:
            """Checks whether the specified resource requirement can be satisfied.
            Args:
                resource_requirement: a dict that specifies resource requirement
                fl_ctx: the FLContext
            Returns:
                A tuple of (check_result, token).
                check_result is a bool indicates whether there is enough resources;
                token is for resource reservation / cancellation for this check request.
            """
            pass

        @abstractmethod
        def cancel_resources(self, resource_requirement: dict, token: str, fl_ctx: FLContext):
            ""Cancels reserved resources if any.
            Args:
                resource_requirement: a dict that specifies resource requirement
                token: a resource reservation token returned by check_resources
                fl_ctx: the FLContext
            Note:
                If check_resource didn't return a token, then don't need to call this method
            """
            pass

        @abstractmethod
        def allocate_resources(self, resource_requirement: dict, token: str, fl_ctx: FLContext) -> dict:
            """Allocates resources.
            Note:
                resource requirements and resources may be different things.
            Args:
                resource_requirement: a dict that specifies resource requirement
                token: a resource reservation token returned by check_resources
                fl_ctx: the FLContext
            Returns:
                A dict of allocated resources
            """
            pass

        @abstractmethod
        def free_resources(self, resources: dict, token: str, fl_ctx: FLContext):
            """Frees resources.
            Args:
                resources: resources to be freed
                token: a resource reservation token returned by check_resources
                fl_ctx: the FLContext
            """
            pass

        @abstractmethod
        def report_resources(self, fl_ctx) -> dict:
            """Reports resources."""
            pass


より扱いやすいインターフェース (AutoCleanResourceManager) も提供されています。

.. code-block:: python

    class AutoCleanResourceManager(ResourceManagerSpec, FLComponent, ABC):

        @abstractmethod
        def _deallocate(self, resources: dict):
            """Deallocates the resources.
            Args:
                resources (dict): the resources to be freed.
            """
            raise NotImplementedError

        @abstractmethod
        def _check_required_resource_available(self, resource_requirement: dict) -> bool:
            """Checks if resources are available.
            Args:
                resource_requirement (dict): the resource requested.
            Return:
                A boolean to indicate whether the current resources are enough for the required resources.
            """
            raise NotImplementedError

        @abstractmethod
        def _reserve_resource(self, resource_requirement: dict) -> dict:
            """Reserves resources given the requirements.
            Args:
                resource_requirement (dict): the resource requested.
            Return:
                A dict of reserved resources associated with the requested resource.
            """
            raise NotImplementedError

        @abstractmethod
        def _resource_to_dict(self) -> dict:
            raise NotImplementedError
