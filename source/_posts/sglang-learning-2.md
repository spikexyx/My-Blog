---
title: SGLang源码笔记（2）- Model权重加载相关
date: 2025-06-16 19:29:03
tags:
    - SGLang
    - LLM
    - Inference Engine
categories:
    - AI Infra
cover: "imgs/sglang-logo.png"
excerpt: "SGLang源码梳理笔记：模型权重加载相关代码。"
comment: false
---

## 从server启动到加载模型权重

仍然是从launch\_server开始看，sglang/launch\_server.py中启动server入口：

```python
if __name__ == "__main__":
    server_args = prepare_server_args(sys.argv[1:])

    try:
        launch_server(server_args)
    finally:
        kill_process_tree(os.getpid(), include_parent=False)
```

进入http\_server中的launch\_server：

```python
def launch_server(
    server_args: ServerArgs,
    pipe_finish_writer: Optional[multiprocessing.connection.Connection] = None,
    launch_callback: Optional[Callable[[], None]] = None,
):
    
    tokenizer_manager, scheduler_info = _launch_subprocesses(server_args=server_args)
    ......
```

然后进入\_launch\_subprocesses启动子进程（Scheduler, TokenizerManager和DetokenizerManager）

```python
def _launch_subprocesses(
    server_args: ServerArgs, port_args: Optional[PortArgs] = None
) -> Tuple[TokenizerManager, Dict]:
    """
    Launch the TokenizerManager in the main process, the Scheduler in a subprocess, and the DetokenizerManager in another subprocess.
    """
    # ......
        for pp_rank in pp_rank_range:
            for tp_rank in tp_rank_range:
                reader, writer = mp.Pipe(duplex=False)
                gpu_id = (
                    server_args.base_gpu_id
                    + ((pp_rank % pp_size_per_node) * tp_size_per_node)
                    + (tp_rank % tp_size_per_node) * server_args.gpu_id_step
                )
                proc = mp.Process(
                    target=run_scheduler_process,
                    args=(
                        server_args,
                        port_args,
                        gpu_id,
                        tp_rank,
                        pp_rank,
                        None,
                        writer,
                    ),
                )
                with memory_saver_adapter.configure_subprocess():
                    proc.start()
                scheduler_procs.append(proc)
                scheduler_pipe_readers.append(reader)
    # ......
```

仍然重点关心Scheduler子进程：

```python
def run_scheduler_process(
    server_args: ServerArgs,
    port_args: PortArgs,
    gpu_id: int,
    tp_rank: int,
    pp_rank: int,
    dp_rank: Optional[int],
    pipe_writer,
):
    # ......
        if disaggregation_mode == DisaggregationMode.NULL:
            if server_args.pp_size > 1:
                scheduler.event_loop_pp()
            elif scheduler.enable_overlap:
                scheduler.event_loop_overlap()
            else:
                scheduler.event_loop_normal()
        elif disaggregation_mode == DisaggregationMode.PREFILL:
            if scheduler.enable_overlap:
                scheduler.event_loop_overlap_disagg_prefill()
            else:
                scheduler.event_loop_normal_disagg_prefill()

        elif disaggregation_mode == DisaggregationMode.DECODE:
            if scheduler.enable_overlap:
                scheduler.event_loop_overlap_disagg_decode()
            else:
                scheduler.event_loop_normal_disagg_decode()
    # ......
```

进入event\_loop，这里我们看event\_loop\_normal

```python
    @DynamicGradMode()
    def event_loop_normal(self):
        """A normal scheduler loop."""
        while True:
            recv_reqs = self.recv_requests()
            self.process_input_requests(recv_reqs)

            batch = self.get_next_batch_to_run()
            self.cur_batch = batch

            if batch:
                result = self.run_batch(batch)
                self.process_batch_result(batch, result)
            else:
                # When the server is idle, do self-check and re-init some states
                self.check_memory()
                self.new_token_ratio = self.init_new_token_ratio

            self.last_batch = batch
```

从这个event\_loop进去执行再往后的过程就是上一篇笔记里的流程，在关注模型权重加载时这里可以先不看event\_loop，重点是Scheduler的\_\_init\_\_的时候，会初始化TpModelWorker，进而初始化ModelRunner，在ModelRunner里面会加载模型。

看下Scheduler的init：

```python
class Scheduler(
    SchedulerOutputProcessorMixin,
    SchedulerDisaggregationDecodeMixin,
    SchedulerDisaggregationPrefillMixin,
):
    """A scheduler that manages a tensor parallel GPU worker."""
    def __init__(
        self,
        server_args: ServerArgs,
        port_args: PortArgs,
        gpu_id: int,
        tp_rank: int,
        pp_rank: int,
        dp_rank: Optional[int],
    ):
    # ......
        # Launch a tensor parallel worker
        if self.enable_overlap:
            TpWorkerClass = TpModelWorkerClient
        else:
            TpWorkerClass = TpModelWorker

        self.tp_worker = TpWorkerClass(
            server_args=server_args,
            gpu_id=gpu_id,
            tp_rank=tp_rank,
            pp_rank=pp_rank,
            dp_rank=dp_rank,
            nccl_port=port_args.nccl_port,
        )
    # ......
```

进而看一下TpModelWorker的初始化：（tp\_worker\_overlap会维护一个异步的TpModelWorker，与这个同步版本在使用时是互斥的，在tp\_worker\_overlap里实现了一个TpWorkerClient类，实际上就是把这个同步版本的Worker做了一个向上的解耦，对于模型加载等任务还是使用的同步版本里的实现，具体实现在另一个文件sglang/srt/managers/tp\_worker\_overlap\_thread.py里）

```python
class TpModelWorker:
    """A tensor parallel model worker."""

    def __init__(
        self,
        server_args: ServerArgs,
        gpu_id: int,
        tp_rank: int,
        pp_rank: int,
        dp_rank: Optional[int],
        nccl_port: int,
        is_draft_worker: bool = False,
        req_to_token_pool: Optional[ReqToTokenPool] = None,
        token_to_kv_pool_allocator: Optional[TokenToKVPoolAllocator] = None,
    ):
        # Parse args
        self.tp_size = server_args.tp_size
        self.tp_rank = tp_rank
        self.pp_rank = pp_rank

        # Init model and tokenizer
        self.model_config = ModelConfig.from_server_args(
            server_args,
            model_path=(
                server_args.model_path
                if not is_draft_worker
                else server_args.speculative_draft_model_path
            ),
            is_draft_model=is_draft_worker,
        )

        self.model_runner = ModelRunner(
            model_config=self.model_config,
            mem_fraction_static=server_args.mem_fraction_static,
            gpu_id=gpu_id,
            tp_rank=tp_rank,
            tp_size=server_args.tp_size,
            pp_rank=pp_rank,
            pp_size=server_args.pp_size,
            nccl_port=nccl_port,
            server_args=server_args,
            is_draft_worker=is_draft_worker,
            req_to_token_pool=req_to_token_pool,
            token_to_kv_pool_allocator=token_to_kv_pool_allocator,
        )
        # .....
```

进而看一下ModelRunner的初始化：

```python
class ModelRunner:
    """ModelRunner runs the forward passes of the models."""

    def __init__(
        self,
        model_config: ModelConfig,
        mem_fraction_static: float,
        gpu_id: int,
        tp_rank: int,
        tp_size: int,
        pp_rank: int,
        pp_size: int,
        nccl_port: int,
        server_args: ServerArgs,
        is_draft_worker: bool = False,
        req_to_token_pool: Optional[ReqToTokenPool] = None,
        token_to_kv_pool_allocator: Optional[TokenToKVPoolAllocator] = None,
    ):
      # ......
      self.initialize(min_per_gpu_memory)
      # ......
```

```python
    def initialize(self, min_per_gpu_memory: float):
        # ......
        # Load the model
        self.sampler = Sampler()
        self.load_model()
        # ......
```

```python
    def load_model(self):
        # ......
        with self.memory_saver_adapter.region():
            self.model = get_model(
                model_config=self.model_config,
                load_config=self.load_config,
                device_config=DeviceConfig(self.device),
            )
        # ......
```

这里就调用了get\_model来具体获取模型信息。

```python
def get_model(
    *,
    model_config: ModelConfig,
    load_config: LoadConfig,
    device_config: DeviceConfig,
) -> nn.Module:
    loader = get_model_loader(load_config)
    return loader.load_model(
        model_config=model_config,
        device_config=device_config,
    )
```

看一下model loader:

```python
def get_model_loader(load_config: LoadConfig) -> BaseModelLoader:
    """Get a model loader based on the load format."""

    if isinstance(load_config.load_format, type):
        return load_config.load_format(load_config)

    if load_config.load_format == LoadFormat.DUMMY:
        return DummyModelLoader(load_config)

    if load_config.load_format == LoadFormat.SHARDED_STATE:
        return ShardedStateLoader(load_config)

    if load_config.load_format == LoadFormat.BITSANDBYTES:
        return BitsAndBytesModelLoader(load_config)

    if load_config.load_format == LoadFormat.GGUF:
        return GGUFModelLoader(load_config)

    if load_config.load_format == LoadFormat.LAYERED:
        return LayeredModelLoader(load_config)

    if load_config.load_format == LoadFormat.REMOTE:
        return RemoteModelLoader(load_config)

    return DefaultModelLoader(load_config)
```

看默认的DefaultModelLoader：

```python
class DefaultModelLoader(BaseModelLoader):
    """Model loader that can load different file types from disk."""
# ......
    def load_model(
        self,
        *,
        model_config: ModelConfig,
        device_config: DeviceConfig,
    ) -> nn.Module:
        target_device = torch.device(device_config.device)
        with set_default_torch_dtype(model_config.dtype):
            with target_device:
                model = _initialize_model(
                    model_config,
                    self.load_config,
                )

            self.load_weights_and_postprocess(
                model, self._get_all_weights(model_config, model), target_device
            )

        return model.eval()
# ......

    @staticmethod
    def load_weights_and_postprocess(model, weights, target_device):
        model.load_weights(weights)

        for _, module in model.named_modules():
            quant_method = getattr(module, "quant_method", None)
            if quant_method is not None:
                # When quant methods need to process weights after loading
                # (for repacking, quantizing, etc), they expect parameters
                # to be on the global target device. This scope is for the
                # case where cpu offloading is used, where we will move the
                # parameters onto device for processing and back off after.
                with device_loading_context(module, target_device):
                    quant_method.process_weights_after_loading(module)
# ......
    # 里面的secondary_weights没发现在哪里使用，应该一般都使用primary_weights
    def _get_all_weights(
        self,
        model_config: ModelConfig,
        model: nn.Module,
    ) -> Generator[Tuple[str, torch.Tensor], None, None]:

        primary_weights = DefaultModelLoader.Source.init_new(model_config, model)
        yield from self._get_weights_iterator(primary_weights)

        secondary_weights = cast(
            Iterable[DefaultModelLoader.Source], getattr(model, "secondary_weights", ())
        )
        for source in secondary_weights:
            yield from self._get_weights_iterator(source)
# ......

    def _get_weights_iterator(
        self, source: "Source"
    ) -> Generator[Tuple[str, torch.Tensor], None, None]:
        """Get an iterator for the model weights based on the load format."""
        hf_folder, hf_weights_files, use_safetensors = self._prepare_weights(
            source.model_or_path, source.revision, source.fall_back_to_pt
        )
        if self.load_config.load_format == LoadFormat.NPCACHE:
            # Currently np_cache only support *.bin checkpoints
            assert use_safetensors is False
            weights_iterator = np_cache_weights_iterator(
                source.model_or_path,
                self.load_config.download_dir,
                hf_folder,
                hf_weights_files,
            )
        elif use_safetensors:
            weights_iterator = safetensors_weights_iterator(hf_weights_files)
        else:
            weights_iterator = pt_weights_iterator(hf_weights_files)

        # Apply the prefix.
        return ((source.prefix + name, tensor) for (name, tensor) in weights_iterator)
# ......
```

load\_weights\_and\_postprocess里调用的model.load\_weights是sglang给每个模型实现的单独的优化过的加载模型函数，以deepseek\_v2为例：

```python
# ......
    def load_weights(self, weights: Iterable[Tuple[str, torch.Tensor]], is_nextn=False):
      # ......
                        weight_loader = getattr(
                            param, "weight_loader", default_weight_loader
                        )
                        weight_loader(param, loaded_weight)
      # ......

```

里面会根据模型特性处理权重和参数，然后在最后会调用weight\_loader来进行具体的权重加载，看一下default\_weight\_loader的加载函数：

```python
# ......
def default_weight_loader(param: torch.Tensor, loaded_weight: torch.Tensor) -> None:
    """Default weight loader."""
    try:
        if param.numel() == 1 and loaded_weight.numel() == 1:
            # Sometimes scalar values aren't considered tensors with shapes
            # so if both param and loaded_weight are a scalar,
            # "broadcast" instead of copy
            param.data.fill_(loaded_weight.item())
        else:
            assert param.size() == loaded_weight.size(), (
                f"Attempted to load weight ({loaded_weight.size()}) "
                f"into parameter ({param.size()})"
            )

            param.data.copy_(loaded_weight)
    except Exception:
        # NOTE: This exception is added for the purpose of setting breakpoint to
        # debug weight loading issues.
        raise
# ......
```

其实就是把处理后的权重拷贝到模型参数里去。

至此，初始化阶段对于模型权重的加载就完成了。