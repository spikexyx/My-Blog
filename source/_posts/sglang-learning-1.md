---
title: SGLang源码笔记（1）- 从server启动到EPLB
date: 2025-06-03 14:04:38
tags:
    - SGLang
    - LLM
    - Inference Engine
categories:
    - AI Infra
cover: "imgs/sglang-logo.png"
excerpt: "SGLang源码梳理笔记：EPLB相关代码，从启动到进入EP管理器，以及对应的记录器与算法处理流程。"
comment: false
---

## 整体代码流程总览

![image](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/mxPOG5zdZQpQknKa/img/d23d4fa1-6cb5-4bae-9a79-18271e118422.svg)

上图为sglang runtime后端的整体流程图，来自[https://github.com/zhaochenyang20/Awesome-ML-SYS-Tutorial/blob/main/sglang/code-walk-through/sglang-architecture.svg](https://github.com/zhaochenyang20/Awesome-ML-SYS-Tutorial/blob/main/sglang/code-walk-through/sglang-architecture.svg)

后段主要可以分为http server和offline inference engine两部分，其中，http server主要面向客户前端，接受各类http请求并进行forward方便后端处理，这块主要就是把这种前后端通讯API相关的模块给单独抽了出来，在本次任务中可以暂时不细看。

> 接下来的代码版本为2025.06.03的main branch，由于eplb相关代码最近sglang的开发者还在高强度pr，所以可能会存在一些滞后

## Server启动

### python/sglang/launch\_server.py

在python/sglang/launch\_server.py里，是启动server的入口指令，先解析参数（server\_args），然后调用srt的launch\_server来进行启动。

```python
from sglang.srt.entrypoints.http_server import launch_server
from sglang.srt.server_args import prepare_server_args
from sglang.srt.utils import kill_process_tree

if __name__ == "__main__":
    server_args = prepare_server_args(sys.argv[1:])

    try:
        launch_server(server_args)
    finally:
        kill_process_tree(os.getpid(), include_parent=False)
```

### python/sglang/srt/entrypoints/http\_server.py

接着进入http server的launch\_server函数，在这里会启动tokenizerManager和scheduler等子线程，并对相应参数进行处理，入口代码如下

```python
def launch_server(
    server_args: ServerArgs,
    pipe_finish_writer: Optional[multiprocessing.connection.Connection] = None,
    launch_callback: Optional[Callable[[], None]] = None,
):
    """
    Launch SRT (SGLang Runtime) Server.

    The SRT server consists of an HTTP server and an SRT engine.

    - HTTP server: A FastAPI server that routes requests to the engine.
    - The engine consists of three components:
        1. TokenizerManager: Tokenizes the requests and sends them to the scheduler.
        2. Scheduler (subprocess): Receives requests from the Tokenizer Manager, schedules batches, forwards them, and sends the output tokens to the Detokenizer Manager.
        3. DetokenizerManager (subprocess): Detokenizes the output tokens and sends the result back to the Tokenizer Manager.

    Note:
    1. The HTTP server, Engine, and TokenizerManager both run in the main process.
    2. Inter-process communication is done through IPC (each process uses a different port) via the ZMQ library.
    """
    tokenizer_manager, scheduler_info = _launch_subprocesses(server_args=server_args)
    set_global_state(
        _GlobalState(
            tokenizer_manager=tokenizer_manager,
            scheduler_info=scheduler_info,
        )
    )

    # Add api key authorization
    if server_args.api_key:
        add_api_key_middleware(app, server_args.api_key)

    # Add prometheus middleware
    if server_args.enable_metrics:
        add_prometheus_middleware(app)
        enable_func_timer()

    # Send a warmup request - we will create the thread launch it
    # in the lifespan after all other warmups have fired.
    warmup_thread = threading.Thread(
        target=_wait_and_warmup,
        args=(
            server_args,
            pipe_finish_writer,
            _global_state.tokenizer_manager.image_token_id,
            launch_callback,
        ),
    )
    app.warmup_thread = warmup_thread

    try:
        # Update logging configs
        set_uvicorn_logging_configs()
        app.server_args = server_args
        # Listen for HTTP requests
        uvicorn.run(
            app,
            host=server_args.host,
            port=server_args.port,
            log_level=server_args.log_level_http or server_args.log_level,
            timeout_keep_alive=5,
            loop="uvloop",
        )
    finally:
        warmup_thread.join()

```

### python/sglang/srt/entrypoints/engine.py

在python/sglang/srt/enrtypoints/engine.py里，会启动子线程，并进行初始化处理。代码入口：

```python
def _launch_subprocesses(
    server_args: ServerArgs, port_args: Optional[PortArgs] = None
) -> Tuple[TokenizerManager, Dict]:
    """
    Launch the TokenizerManager in the main process, the Scheduler in a subprocess, and the DetokenizerManager in another subprocess.
    """
    # Configure global environment
    configure_logger(server_args)
    server_args.check_server_args()
    _set_envs_and_config(server_args)

    # Allocate ports for inter-process communications
    if port_args is None:
        port_args = PortArgs.init_new(server_args)
        logger.info(f"{server_args=}")

    # If using model from www.modelscope.cn, first download the model.
    server_args.model_path, server_args.tokenizer_path = prepare_model_and_tokenizer(
        server_args.model_path, server_args.tokenizer_path
    )

    scheduler_procs = []
    if server_args.dp_size == 1:
        # Launch tensor parallel scheduler processes
        memory_saver_adapter = TorchMemorySaverAdapter.create(
            enable=server_args.enable_memory_saver
        )

        scheduler_pipe_readers = []

        nnodes_per_tp_group = max(server_args.nnodes // server_args.pp_size, 1)
        tp_size_per_node = server_args.tp_size // nnodes_per_tp_group
        tp_rank_range = range(
            tp_size_per_node * (server_args.node_rank % nnodes_per_tp_group),
            tp_size_per_node * (server_args.node_rank % nnodes_per_tp_group + 1),
        )

        pp_size_per_node = max(server_args.pp_size // server_args.nnodes, 1)
        pp_rank_range = range(
            pp_size_per_node * (server_args.node_rank // nnodes_per_tp_group),
            pp_size_per_node * (server_args.node_rank // nnodes_per_tp_group + 1),
        )

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
    else:
        # Launch the data parallel controller
        reader, writer = mp.Pipe(duplex=False)
        scheduler_pipe_readers = [reader]
        proc = mp.Process(
            target=run_data_parallel_controller_process,
            args=(server_args, port_args, writer),
        )
        proc.start()
        scheduler_procs.append(proc)

    if server_args.node_rank >= 1:
        # In multi-node cases, non-zero rank nodes do not need to run tokenizer or detokenizer,
        # so they can just wait here.

        for reader in scheduler_pipe_readers:
            data = reader.recv()
            assert data["status"] == "ready"

        if os.getenv("SGLANG_BLOCK_NONZERO_RANK_CHILDREN") == "0":
            # When using `Engine` as a Python API, we don't want to block here.
            return None, None

        launch_dummy_health_check_server(server_args.host, server_args.port)

        for proc in scheduler_procs:
            proc.join()
            logger.error(
                f"Scheduler or DataParallelController {proc.pid} terminated with {proc.exitcode}"
            )
        return None, None

    # Launch detokenizer process
    detoken_proc = mp.Process(
        target=run_detokenizer_process,
        args=(
            server_args,
            port_args,
        ),
    )
    detoken_proc.start()

    # Launch tokenizer process
    tokenizer_manager = TokenizerManager(server_args, port_args)
    if server_args.chat_template:
        load_chat_template_for_openai_api(
            tokenizer_manager, server_args.chat_template, server_args.model_path
        )
    else:
        guess_chat_template_name_from_model_path(server_args.model_path)

    if server_args.completion_template:
        load_completion_template_for_openai_api(server_args.completion_template)

    # Wait for the model to finish loading
    scheduler_infos = []
    for i in range(len(scheduler_pipe_readers)):
        try:
            data = scheduler_pipe_readers[i].recv()
        except EOFError:
            logger.error(
                f"Rank {i} scheduler is dead. Please check if there are relevant logs."
            )
            scheduler_procs[i].join()
            logger.error(f"Exit code: {scheduler_procs[i].exitcode}")
            raise

        if data["status"] != "ready":
            raise RuntimeError(
                "Initialization failed. Please see the error messages above."
            )
        scheduler_infos.append(data)

    # Assume all schedulers have the same scheduler_info
    scheduler_info = scheduler_infos[0]
    tokenizer_manager.max_req_input_len = scheduler_info["max_req_input_len"]
    return tokenizer_manager, scheduler_info

```

### python/sglang/srt/managers/scheduler.py

在启动子线程过程中，会开启scheduler，这里会选择tp和dp等进行初始化线程，核心代码在python/sglang/srt/managers/scheduler.py。\_laucnh\_subprocesses里调用的核心代码入口：

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
    # Generate the prefix
    prefix = ""
    if dp_rank is not None:
        prefix += f" DP{dp_rank}"
    if server_args.tp_size > 1:
        prefix += f" TP{tp_rank}"
    if server_args.pp_size > 1:
        prefix += f" PP{pp_rank}"

    # Config the process
    kill_itself_when_parent_died()
    setproctitle.setproctitle(f"sglang::scheduler{prefix.replace(' ', '_')}")
    faulthandler.enable()
    parent_process = psutil.Process().parent()

    # [For Router] if env var "SGLANG_DP_RANK" exist, set dp_rank to the value of the env var
    if dp_rank is None and "SGLANG_DP_RANK" in os.environ:
        dp_rank = int(os.environ["SGLANG_DP_RANK"])

    # Configure the logger
    configure_logger(server_args, prefix=prefix)
    suppress_other_loggers()

    # Set cpu affinity to this gpu process
    if get_bool_env_var("SGLANG_SET_CPU_AFFINITY"):
        set_gpu_proc_affinity(server_args.tp_size, server_args.nnodes, gpu_id)

    embedding_cache_size = 100
    if "SGLANG_VLM_CACHE_SIZE_MB" in os.environ:
        embedding_cache_size = int(os.environ["SGLANG_VLM_CACHE_SIZE_MB"])
    init_embedding_cache(embedding_cache_size * 1024 * 1024)
    # Create a scheduler and run the event loop
    try:
        scheduler = Scheduler(server_args, port_args, gpu_id, tp_rank, pp_rank, dp_rank)
        pipe_writer.send(
            {
                "status": "ready",
                "max_total_num_tokens": scheduler.max_total_num_tokens,
                "max_req_input_len": scheduler.max_req_input_len,
            }
        )
        disaggregation_mode: DisaggregationMode = scheduler.disaggregation_mode

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

    except Exception:
        traceback = get_exception_traceback()
        logger.error(f"Scheduler hit an exception: {traceback}")
        parent_process.send_signal(signal.SIGQUIT)

```

选取其中的event\_loop\_normal为例进行查看：（是Scheduler class的一个成员函数，同样在scheduler.py文件里定义）

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

可以看出它的核心处理流程就是：recv\_requests -> get\_next\_batch\_to\_run -> run\_batch -> process\_batch\_result，与最上面的流程图基本保持一致。

## Runtime Backend

从get\_next\_batch\_to\_run进入开始查看，首先会确定下一个执行的batch是什么，代码入口同样在scheduler.py：

```python
    def get_next_batch_to_run(self) -> Optional[ScheduleBatch]:
        # Merge the prefill batch into the running batch
        chunked_req_to_exclude = set()
        if self.chunked_req:
            # Move the chunked request out of the batch so that we can merge
            # only finished requests to running_batch.
            chunked_req_to_exclude.add(self.chunked_req)
            self.tree_cache.cache_unfinished_req(self.chunked_req)
            # chunked request keeps its rid but will get a new req_pool_idx
            self.req_to_token_pool.free(self.chunked_req.req_pool_idx)
        if self.last_batch and self.last_batch.forward_mode.is_extend():
            if self.last_batch.chunked_req is not None:
                # In the context pipeline parallelism, after the last chunk, the current microbatch still track outdated chunked_req.
                # We need to discard it.
                chunked_req_to_exclude.add(self.last_batch.chunked_req)

            # Filter batch
            last_bs = self.last_batch.batch_size()
            self.last_batch.filter_batch(
                chunked_req_to_exclude=list(chunked_req_to_exclude)
            )
            if self.last_batch.batch_size() < last_bs:
                self.running_batch.batch_is_full = False

            # Merge the new batch into the running batch
            if not self.last_batch.is_empty():
                if self.running_batch.is_empty():
                    self.running_batch = self.last_batch
                else:
                    # Merge running_batch with prefill batch
                    self.running_batch.merge_batch(self.last_batch)

        new_batch = self.get_new_batch_prefill()
        if new_batch is not None:
            # Run prefill first if possible
            ret = new_batch
        else:
            # Run decode
            if not self.running_batch.is_empty():
                self.running_batch = self.update_running_batch(self.running_batch)
                ret = self.running_batch if not self.running_batch.is_empty() else None
            else:
                ret = None

        # Handle DP attention
        if self.server_args.enable_dp_attention or self.server_args.enable_sp_layernorm:
            ret, _ = self.prepare_dp_attn_batch(ret)

        return ret

```

优先处理prefill，其次才是decode batch。（非pd分离的情况）

然后看一下run\_batch，代码同样在scheduler.py，代码：

```python
    def run_batch(
        self, batch: ScheduleBatch
    ) -> Union[GenerationBatchResult, EmbeddingBatchResult]:
        """Run a batch."""
        self.forward_ct += 1

        # Check profiler
        if (
            self.profiler_target_forward_ct
            and self.profiler_target_forward_ct <= self.forward_ct
        ):
            self.send_to_tokenizer.send_pyobj(self.stop_profile())

        if self.forward_sleep_time is not None:
            logger.info(f"Scheduler.run_batch sleep {self.forward_sleep_time}s")
            time.sleep(self.forward_sleep_time)

        # Run forward
        if self.is_generation:
            if self.spec_algorithm.is_none():
                model_worker_batch = batch.get_model_worker_batch()
                if self.pp_group.is_last_rank:
                    logits_output, next_token_ids, can_run_cuda_graph = (
                        self.tp_worker.forward_batch_generation(model_worker_batch)
                    )
                else:
                    pp_hidden_states_proxy_tensors, _, can_run_cuda_graph = (
                        self.tp_worker.forward_batch_generation(model_worker_batch)
                    )
                bid = model_worker_batch.bid
            else:
                (
                    logits_output,
                    next_token_ids,
                    bid,
                    num_accepted_tokens,
                    can_run_cuda_graph,
                ) = self.draft_worker.forward_batch_speculative_generation(batch)
                self.spec_num_total_accepted_tokens += (
                    num_accepted_tokens + batch.batch_size()
                )
                self.spec_num_total_forward_ct += batch.batch_size()
                self.num_generated_tokens += num_accepted_tokens

            if self.pp_group.is_last_rank:
                batch.output_ids = next_token_ids

            # These 2 values are needed for processing the output, but the values can be
            # modified by overlap schedule. So we have to copy them here so that
            # we can use the correct values in output processing.
            if batch.return_logprob:
                extend_input_len_per_req = [req.extend_input_len for req in batch.reqs]
                extend_logprob_start_len_per_req = [
                    req.extend_logprob_start_len for req in batch.reqs
                ]
            else:
                extend_input_len_per_req = None
                extend_logprob_start_len_per_req = None

            ret = GenerationBatchResult(
                logits_output=logits_output if self.pp_group.is_last_rank else None,
                pp_hidden_states_proxy_tensors=(
                    pp_hidden_states_proxy_tensors
                    if not self.pp_group.is_last_rank
                    else None
                ),
                next_token_ids=next_token_ids if self.pp_group.is_last_rank else None,
                extend_input_len_per_req=extend_input_len_per_req,
                extend_logprob_start_len_per_req=extend_logprob_start_len_per_req,
                bid=bid,
                can_run_cuda_graph=can_run_cuda_graph,
            )
        else:  # embedding or reward model
            model_worker_batch = batch.get_model_worker_batch()
            embeddings = self.tp_worker.forward_batch_embedding(model_worker_batch)
            ret = EmbeddingBatchResult(
                embeddings=embeddings, bid=model_worker_batch.bid
            )
        return ret

```

### python/sglang/srt/managers/tp\_worker.py

会将决定好的当前需要执行的batch取出来，然后进行实际执行，里面会区分几种forward类型，以非spec类型为例，会调用tp\_worker的forward\_batch\_generation，根据是否开启overlap会分成两个forward\_batch\_generation，我们查看不开启overlap的模式下的执行，代码在python/sglang/srt/managers/tp\_worker.py（TpModelWorker类的成员函数）：

```python
def forward_batch_generation(
        self,
        model_worker_batch: ModelWorkerBatch,
        launch_done: Optional[threading.Event] = None,
        skip_sample: bool = False,
    ) -> Tuple[
        Union[LogitsProcessorOutput, torch.Tensor], Optional[torch.Tensor], bool
    ]:
        forward_batch = ForwardBatch.init_new(model_worker_batch, self.model_runner)

        pp_proxy_tensors = None
        if not self.pp_group.is_first_rank:
            pp_proxy_tensors = PPProxyTensors(
                self.pp_group.recv_tensor_dict(
                    all_gather_group=self.get_attention_tp_group()
                )
            )

        if self.pp_group.is_last_rank:
            logits_output, can_run_cuda_graph = self.model_runner.forward( # 会调用model runner的forward函数进行前向传播
                forward_batch, pp_proxy_tensors=pp_proxy_tensors
            )
            if launch_done is not None:
                launch_done.set()

            if skip_sample:
                next_token_ids = None
            else:
                next_token_ids = self.model_runner.sample(
                    logits_output, model_worker_batch
                )

            return logits_output, next_token_ids, can_run_cuda_graph
        else:
            pp_proxy_tensors, can_run_cuda_graph = self.model_runner.forward(
                forward_batch,
                pp_proxy_tensors=pp_proxy_tensors,
            )
            return pp_proxy_tensors.tensors, None, can_run_cuda_graph

```

### python/sglang/srt/model\_executor/model\_runner.py

会根据我们传入的model\_worker\_batch和当前的model runner创建实际执行的forward\_batch，进而在区分各类情况后，会调用model runner的forward函数，继续进行查看，代码在python/sglang/srt/model\_executor/model\_runner.py。

```python
    def forward(
        self,
        forward_batch: ForwardBatch,
        skip_attn_backend_init: bool = False,
        pp_proxy_tensors: Optional[PPProxyTensors] = None,
    ) -> Tuple[Union[LogitsProcessorOutput, PPProxyTensors], bool]:
        self.forward_pass_id += 1

        # 这里会在前向传播的同时，用记录器对我们需要的信息进行record
        with get_global_expert_distribution_recorder().with_forward_pass(
            self.forward_pass_id,
            forward_batch,
        ):
            output = self._forward_raw(
                forward_batch, skip_attn_backend_init, pp_proxy_tensors
            )

        # 在前向传播的尾部插入eplb的hook
        if self.eplb_manager is not None:
            self.eplb_manager.on_forward_pass_end(self.forward_pass_id)

        return output
```

++**可以看到，我们需要改动的eplb部分也在forward部分会进行触发**++**。**

## EPLB相关

### python/sglang/srt/managers/expert\_distribution.py

其中会使用with，在进行实际的前向传播的同时，进入一个ep分布的记录器进行对应的记录，这个记录器的定义在python/sglang/srt/managers/expert\_distribution.py里，有一个基类

```python
class ExpertDistributionRecorder(ABC):
    """Global expert distribution recording"""
    ...
```

实际代码流程中用到的是底下的：

```python
class _ExpertDistributionRecorderReal(ExpertDistributionRecorder):
    def __init__(
        self,
        server_args: ServerArgs,
        expert_location_metadata: "ExpertLocationMetadata",
        rank: int,
    ):
        self._server_args = server_args
        self._expert_location_metadata = expert_location_metadata

        self._recording = False
        self._current_forward_pass_id = Withable()
        self._current_layer_idx = Withable()
        self._current_debug_name = Withable()
        self._accumulator = _Accumulator.init_new(
            server_args, expert_location_metadata, rank
        )
        self._single_pass_gatherers = {
            k: _SinglePassGatherer.init_new(server_args, expert_location_metadata, rank)
            for k in self._accumulator.get_single_pass_gatherer_keys()
        }

        if server_args.enable_expert_distribution_metrics:
            logger.info(
                "ExpertDistributionRecorder auto start record since enable_expert_distribution_metrics"
            )
            self.start_record()

    def with_current_layer(self, layer_idx):
        return self._current_layer_idx.with_value(layer_idx)

    def with_debug_name(self, debug_name):
        return self._current_debug_name.with_value(debug_name)

    @contextmanager
    def with_forward_pass(self, forward_pass_id: int, forward_batch: ForwardBatch):
        with self._current_forward_pass_id.with_value(forward_pass_id):
            self._on_forward_pass_start(forward_batch)
            try:
                yield
            finally:
                self._on_forward_pass_end(forward_pass_id)

    def _on_forward_pass_start(self, forward_batch: ForwardBatch):
        if not self._recording:
            return
        for gatherer_key, gatherer in self._single_pass_gatherers.items():
            gatherer.reset()
            gatherer.on_forward_pass_start(forward_batch)

    def _on_forward_pass_end(self, forward_pass_id: int):
        if not self._recording:
            return
        for gatherer_key, gatherer in self._single_pass_gatherers.items():
            single_pass_data = gatherer.collect()
            self._accumulator.append(forward_pass_id, gatherer_key, single_pass_data)

    def on_select_experts(self, topk_ids: torch.Tensor):
        self._on_hook("on_select_experts", topk_ids=topk_ids)

    def on_deepep_dispatch_normal(
        self,
        local_physical_count_of_layer: List[int],
        num_tokens_per_rank,
        num_tokens_per_rdma_rank,
        num_tokens_per_expert,
    ):
        self._on_hook(
            "on_deepep_dispatch_normal",
            local_physical_count_of_layer=local_physical_count_of_layer,
            num_tokens_per_rank=num_tokens_per_rank,
            num_tokens_per_rdma_rank=num_tokens_per_rdma_rank,
            num_tokens_per_expert=num_tokens_per_expert,
        )

    def on_deepep_dispatch_low_latency(
        self, local_physical_count_of_layer: torch.Tensor
    ):
        self._on_hook(
            "on_deepep_dispatch_low_latency",
            local_physical_count_of_layer=local_physical_count_of_layer,
        )

    def _on_hook(self, hook_name: str, **kwargs):
        if not (self._recording or torch.cuda.is_current_stream_capturing()):
            return
        gatherer = self._single_pass_gatherers[
            self._accumulator.get_single_pass_gatherer_key(
                self._current_debug_name.value
            )
        ]
        getattr(gatherer, hook_name)(layer_idx=self._current_layer_idx.value, **kwargs)

    def _reset(self):
        """Reset the expert distribution recorder."""
        logger.info("Resetting ExpertDistributionRecorder...")
        assert (
            self._current_layer_idx.value is None
        ), f"{self._current_layer_idx.value=}"
        for gatherer in self._single_pass_gatherers.values():
            gatherer.reset()
        self._accumulator.reset()

    def start_record(self):
        """Start recording the expert distribution."""
        if self._recording:
            logger.warning(
                "SGLang server is already recording expert ids. Did you forget to dump the expert ids recorded so far by sending requests to the `/stop_expert_distribution_record` and `/dump_expert_distribution_record` endpoints?"
            )
        self._reset()
        self._recording = True

    def stop_record(self):
        """Stop recording the expert distribution."""
        if not self._recording:
            logger.warning(
                "SGLang server has not been recording expert ids. Did you forget to start recording by sending request to the `/start_expert_distribution_record` endpoint?"
            )
        self._recording = False

    def dump_record(self, output_mode: _OutputMode = "file"):
        """Dump the expert distribution record and reset the recorder after dumping."""
        output = self._accumulator.dump(output_mode=output_mode)
        self._reset()
        return output

    @property
    def recording(self):
        return self._recording

```

看一下记录器在前向传播过程中的函数入口with\_forward\_pass：

```python
    @contextmanager
    def with_forward_pass(self, forward_pass_id: int, forward_batch: ForwardBatch):
        with self._current_forward_pass_id.with_value(forward_pass_id):
            self._on_forward_pass_start(forward_batch)
            try:
                yield
            finally:
                self._on_forward_pass_end(forward_pass_id)

    def _on_forward_pass_start(self, forward_batch: ForwardBatch):
        if not self._recording:
            return
        for gatherer_key, gatherer in self._single_pass_gatherers.items():
            gatherer.reset()
            gatherer.on_forward_pass_start(forward_batch)

    def _on_forward_pass_end(self, forward_pass_id: int):
        if not self._recording:
            return
        for gatherer_key, gatherer in self._single_pass_gatherers.items():
            single_pass_data = gatherer.collect()
            self._accumulator.append(forward_pass_id, gatherer_key, single_pass_data)

```

可以看见，是在前向传播开始的时候，recorder会调用gatherer的on\_forward\_pass\_start进行信息收集，而在此次前向传播结束的时候，调用gatherer的on\_forward\_pass\_end进行信息汇总，并用accumulator进行信息处理。

里面的\_single\_pass\_gatherers是一个字典，存储所有gatherer类型和对应的key，在一次前向pass里，会把这些gatherer都进行收集，然后用accumulator进行最终汇总（但我看目前的实现好像primary key是固定的，也就是gatherer只会调用一个），

我们接下来先看一下gatherer的基类:

```python
class _SinglePassGatherer(ABC):
    @staticmethod
    def init_new(
        server_args: ServerArgs,
        expert_location_metadata: "ExpertLocationMetadata",
        rank: int,
    ) -> "_SinglePassGatherer":
        if server_args.expert_distribution_recorder_mode == "per_token":
            return _DetailSinglePassGatherer(
                server_args, expert_location_metadata, rank
            )
        if server_args.enable_deepep_moe:
            if server_args.deepep_mode == "normal":
                return _DeepepNormalSinglePassGatherer(expert_location_metadata, rank)
            elif server_args.deepep_mode == "low_latency":
                return _DeepepLowLatencySinglePassGatherer(
                    expert_location_metadata, rank
                )
            else:
                raise NotImplementedError
        return _SelectExpertsSinglePassGatherer(expert_location_metadata, rank)

    def __init__(self, expert_location_metadata: "ExpertLocationMetadata", rank: int):
        self._expert_location_metadata = expert_location_metadata
        self._rank = rank

    def on_forward_pass_start(self, forward_batch: ForwardBatch):
        pass

    def on_select_experts(self, layer_idx: int, topk_ids: torch.Tensor):
        pass

    def on_deepep_dispatch_normal(
        self,
        layer_idx: int,
        local_physical_count_of_layer: List[int],
        num_tokens_per_rank,
        num_tokens_per_rdma_rank,
        num_tokens_per_expert,
    ):
        pass

    def on_deepep_dispatch_low_latency(
        self, layer_idx: int, local_physical_count_of_layer: torch.Tensor
    ):
        pass

    def reset(self):
        raise NotImplementedError

    def collect(self) -> Dict:
        raise NotImplementedError

```

会选择一个具体的继承类进行init，由于开发者注释里注明了在实际产品中会使用deepep gatherer来获取更高的速度，所以我们接下来会主要查看deepep moe对应的类。我们查看\_DeepepNormalSinglePassGatherer:（先默认使用expert\_distribution\_recorder\_mode为stat，也就是静态模式）

```python
class _DeepepNormalSinglePassGatherer(_LayerBasedSinglePassGatherer):
    def on_deepep_dispatch_normal(
        self,
        layer_idx: int,
        local_physical_count_of_layer: List[int],
        num_tokens_per_rank,
        num_tokens_per_rdma_rank,
        num_tokens_per_expert,
    ):
        assert isinstance(local_physical_count_of_layer, list)
        self._on_layer_data(layer_idx, local_physical_count_of_layer)

    def collect(self) -> Dict:
        local_physical_count = super()._collect_objects(
            pad_len=self._expert_location_metadata.num_local_physical_experts
        )
        global_physical_count = _convert_local_to_global_physical_count(
            local_physical_count,
            rank=self._rank,
            num_local_physical_experts=self._expert_location_metadata.num_local_physical_experts,
            num_physical_experts=self._expert_location_metadata.num_physical_experts,
        )
        return dict(global_physical_count=global_physical_count)

```

里面的collect函数就是用来收集这次pass里的具体的每个rank上的所有layer的physical expert上的被分配的token的情况，其中super().\_collect\_objects()函数会获取本地的data count，再进行local到全体到转换。最后global\_physical\_count得到的是一个\[num\_layers, num\_physical\_experts\]形状的tensor，最终返回的是一个字典。

注意的是，现在的main分支里实现的部分（2025.6.3），对于on\_forward\_pass\_start还没有做什么额外操作，只会对gatherer进行reset，在on\_forward\_pass\_end的时候才会进行collect收集这一批次的信息。

在确定由gatherer收集信息后，我们再来看一下accumulator，先查看一下基类：

```python
class _Accumulator(ABC):
    @staticmethod
    def init_new(
        server_args: ServerArgs,
        expert_location_metadata: "ExpertLocationMetadata",
        rank: int,
    ) -> "_Accumulator":
        return _Accumulator.get_class(server_args)(
            server_args, expert_location_metadata, rank
        )

    @staticmethod
    def get_class(server_args: ServerArgs) -> Type["_Accumulator"]:
        return {
            "stat": _StatAccumulator,
            "per_pass": _DetailAccumulator,
            "per_token": _DetailAccumulator,
        }[server_args.expert_distribution_recorder_mode]

    def __init__(
        self,
        server_args: ServerArgs,
        expert_location_metadata: "ExpertLocationMetadata",
        rank: int,
    ):
        self._server_args = server_args
        self._expert_location_metadata = expert_location_metadata
        self._rank = rank

    def get_single_pass_gatherer_keys(self):
        return [_SINGLE_PASS_GATHERER_KEY_PRIMARY]

    def get_single_pass_gatherer_key(self, debug_name: Optional[str]):
        return _SINGLE_PASS_GATHERER_KEY_PRIMARY

    def append(
        self,
        forward_pass_id: int,
        gatherer_key: str,
        single_pass_data: Dict,
    ):
        pass

    def reset(self):
        pass

    def dump(self, output_mode: _OutputMode):
        pass

```

可以看见有三种模式可选，分别是stat, per\_pass和per\_token，我们先查看stat模式下的情况：

```python
class _StatAccumulator(_UtilizationRateAccumulatorMixin):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._global_physical_count_of_buffered_step = _Buffer.init_new(
            item_shape=(
                self._expert_location_metadata.num_layers,
                # Cannot use local_physical_count to support select_experts
                self._expert_location_metadata.num_physical_experts,
            ),
            buffer_size=self._server_args.expert_distribution_recorder_buffer_size,
            dtype=torch.int32,
            device=self._server_args.device,
        )

    def append(
        self,
        forward_pass_id: int,
        gatherer_key: str,
        single_pass_data: Dict,
    ):
        super().append(forward_pass_id, gatherer_key, single_pass_data)
        # Can optimize if overhead here is large
        self._global_physical_count_of_buffered_step.append(
            single_pass_data["global_physical_count"]
        )

    def reset(self):
        super().reset()
        self._global_physical_count_of_buffered_step.reset()

    def dump(self, output_mode: _OutputMode):
        logical_count_of_buffered_step = _convert_global_physical_count_to_logical_count(
            self._global_physical_count_of_buffered_step.get_all(),
            num_layers=self._expert_location_metadata.num_layers,
            num_logical_experts=self._expert_location_metadata.num_logical_experts,
            physical_to_logical_map=self._expert_location_metadata.physical_to_logical_map,
        )
        torch.distributed.all_reduce(
            logical_count_of_buffered_step, op=torch.distributed.ReduceOp.SUM
        )
        output = dict(
            rank=self._rank,
            logical_count=logical_count_of_buffered_step,
        )

        if output_mode == "file":
            if self._rank == 0:
                _dump_to_file(f"expert_distribution_recorder_{time.time()}.pt", output)
        elif output_mode == "object":
            return output
        else:
            raise NotImplementedError

```

它的append函数也主要是调用的它的父类的append函数，我们查看一下这个\_UtilizationRateAccumulatorMixin:

```python
class _UtilizationRateAccumulatorMixin(_Accumulator):
    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)

        self._enable = self._server_args.enable_expert_distribution_metrics

        if self._enable:
            window_sizes = [10, 100, 1000]
            self._history = _DequeCollection(maxlens=window_sizes)
            self._rank = torch.distributed.get_rank()

    def append(
        self,
        forward_pass_id: int,
        gatherer_key: str,
        single_pass_data: Dict,
    ):
        super().append(forward_pass_id, gatherer_key, single_pass_data)
        if self._enable:
            self._append_utilization_rate(
                forward_pass_id, single_pass_data["global_physical_count"]
            )

    def reset(self):
        super().reset()
        if self._enable:
            self._history.clear()

    def _append_utilization_rate(
        self, forward_pass_id: int, single_pass_global_physical_count: torch.Tensor
    ):
        gpu_physical_count = compute_gpu_physical_count(
            single_pass_global_physical_count,
            num_gpu=self._expert_location_metadata.ep_size,
        )
        gpu_physical_count = gpu_physical_count.to(self._server_args.device)
        torch.distributed.reduce(
            gpu_physical_count, dst=0, op=torch.distributed.ReduceOp.SUM
        )

        if self._rank == 0:
            utilization_rate_tensor = compute_utilization_rate(gpu_physical_count)
            utilization_rate = torch.mean(utilization_rate_tensor).item()
            self._history.append(utilization_rate)

            gpu_physical_count_sum = gpu_physical_count.sum().item()

            logger.info(
                f"[Expert Balancedness] "
                f"forward_pass_id={forward_pass_id} "
                f"current_pass_balancedness={utilization_rate:.03f} "
                f"{''.join(f'last_{size}_average_balancedness={value:.03f} ' for size, value in self._history.mean().items())} "
                f"gpu_physical_count_sum={gpu_physical_count_sum}"
                # f"current_pass_per_layer={[round(x, 2) for x in utilization_rate_tensor.cpu().tolist()]}"
            )

```

这个 append 函数的作用是：在累加器（Accumulator）中追加一条新的专家分布统计数据，并在需要时计算和记录专家利用率（balancedness）指标。首先调用父类的 append 方法，把这次 forward pass 收集到的数据（single\_pass\_data）追加到累加器中，做基础的数据记录。

如果启用了专家分布利用率统计（self.\_enable 为 True），则进一步调用 \_append\_utilization\_rate 方法。传入本次 forward pass 的 id 和全局物理专家分布（global\_physical\_count）。这个方法会计算当前专家分布的“均衡性”（balancedness/utilization rate），并记录到历史窗口（一个DequeCollection类）中，还会在 rank 0 上打印日志，便于监控和分析。

当在启动参数中开启enable\_expert\_distribution\_metrics时，每次forward除了记录器的append，这个mixin类也会对这次forward过程中的物理专家访问数进行保存进这个collection。然后在计算平衡度的时候，就用其中的mean方法返回对应区间内的平衡分布平均值。当开启这个metrics参数后，在执行过程中开启log可以观察到类似下面的log:

```python
2025-06-06 13:41:30,281 - sglang.srt.managers.expert_distribution - INFO - [Expert Balancedness] forward_pass_id=1350 current_pass_balancedness=0.819 last_10_average_balancedness=0.825 last_100_average_balancedness=0.830 last_1000_average_balancedness=0.835  gpu_physical_count_sum=6240
2025-06-06 13:41:30,335 - sglang.srt.managers.expert_distribution - INFO - [Expert Balancedness] forward_pass_id=1351 current_pass_balancedness=0.858 last_10_average_balancedness=0.830 last_100_average_balancedness=0.830 last_1000_average_balancedness=0.835  gpu_physical_count_sum=6240
......
```

因为可以通过这个类获取到保存的一些历史平衡度信息，所以也可以很方便地添加一些自己的中间函数来做类似动态阈值检测等方案。

++**至此，关于一次前向传播中对于expert分布的数据记录的相关代码基本就查看完毕了，接下来我们查看一下尾部的这段代码**++：

```python
# forward pass update eplb on pass end
        if self.eplb_manager is not None:
            self.eplb_manager.on_forward_pass_end(self.forward_pass_id)

```

### python/sglang/srt/managers/eplb\_manager.py

eplb\_manager的对应代码实现在python/sglang/srt/managers/eplb\_manager.py，我们看一下这个类的实现：

```python
class EPLBManager:
    def __init__(self, model_runner: "ModelRunner"):
        super().__init__()
        self._model_runner = model_runner
        self._server_args = model_runner.server_args

        # Otherwise, the circular buffer will contain stale data. If the case is needed, it can be implemented.
        assert (
            self._server_args.eplb_rebalance_num_iterations
            >= self._server_args.expert_distribution_recorder_buffer_size
        ), "eplb_rebalance_num_iterations must be less than expert_distribution_recorder_buffer_size"

        if not get_global_expert_distribution_recorder().recording:
            get_global_expert_distribution_recorder().start_record()

        logger.info(
            f"[EPLBManager] system started, will rebalance per {self._server_args.eplb_rebalance_num_iterations} iterations."
        )

    def on_forward_pass_end(self, forward_pass_id: int):
        if forward_pass_id % self._server_args.eplb_rebalance_num_iterations == 0: # 固定循环次数进行更新
            self.rebalance()

    # 进行重新平衡
    def rebalance(self):
        logger.info("[EPLBManager] rebalance start")
        torch.cuda.synchronize()
        time_start = time.time()

        logical_count = get_global_expert_distribution_recorder().dump_record(
            output_mode="object"
        )["logical_count"]
        expert_location_metadata = ExpertLocationMetadata.init_by_eplb(
            self._server_args, self._model_runner.model_config, logical_count
        )
        self._model_runner.update_expert_location(expert_location_metadata)

        torch.cuda.synchronize()
        time_end = time.time()
        logger.info(f"[EPLBManager] rebalance end time={time_end - time_start:.3f}s")

```

它的on\_forward\_pass\_end现在的实现里，会在每1000次的时候进行一次expert的rebalance。（TODO：优化点1）

这里面的get\_global\_expert\_distribution\_recorder()返回的就是上面的那个recorder记录器。dump\_record就会把之前记录的信息给dump出来。

### python/sglang/srt/managers/expert\_location.py

接下来先看一下ExpertLocationMetadata.init\_by\_eplb这段，也就是为了重新平衡所以通过eplb进行初始化的新的专家元数据，代码在python/sglang/srt/managers/expert\_location.py:

```python
@dataclass
class ExpertLocationMetadata:
    physical_to_logical_map: torch.Tensor  # (layers, num_physical_experts)
    logical_to_all_physical_map: torch.Tensor  # (layers, num_logical_experts, X)
    logical_to_all_physical_map_num_valid: torch.Tensor  # (layers, num_logical_experts)
    logical_to_rank_dispatch_physical_map: torch.Tensor  # (layers, num_logical_experts)

    # -------------------------------- properties ------------------------------------

    @property
    def num_layers(self) -> int:
        return self.physical_to_logical_map.shape[0]

    @property
    def num_physical_experts(self) -> int:
        return self.physical_to_logical_map.shape[1]

    @property
    def num_local_physical_experts(self) -> int:
        ans, remainder = divmod(self.num_physical_experts, self.ep_size)
        assert remainder == 0
        return ans

    @property
    def num_logical_experts(self) -> int:
        return self.logical_to_all_physical_map.shape[1]

    @property
    def ep_size(self):
        # TODO change when EP size != world size
        return torch.distributed.get_world_size()

    def __post_init__(self):
        num_layers_0, num_physical_experts_0 = self.physical_to_logical_map.shape
        num_layers_1, num_logical_experts_0, num_physical_experts_1 = (
            self.logical_to_all_physical_map.shape
        )
        num_layers_2, num_logical_experts_1 = (
            self.logical_to_all_physical_map_num_valid.shape
        )
        num_layers_3, num_logical_experts_2 = (
            self.logical_to_rank_dispatch_physical_map.shape
        )
        assert num_layers_0 == num_layers_1 == num_layers_2 == num_layers_3
        assert num_logical_experts_0 == num_logical_experts_1 == num_logical_experts_2
        assert num_physical_experts_0 == num_physical_experts_1

    # -------------------------------- construction ------------------------------------

    @staticmethod
    def init_trivial(server_args: ServerArgs, model_config: ModelConfig):
        """Trivial location - logical expert i corresponds to physical expert i"""
        common = ExpertLocationMetadata._init_common(server_args, model_config)
        num_physical_experts = common["num_physical_experts"]
        model_config_for_expert_location = common["model_config_for_expert_location"]
        num_layers = model_config_for_expert_location.num_layers
        num_logical_experts = model_config_for_expert_location.num_logical_experts

        physical_to_logical_map = (
            torch.arange(0, num_physical_experts).repeat(num_layers, 1)
            % num_logical_experts
        )

        return ExpertLocationMetadata.init_by_mapping(
            server_args,
            model_config,
            physical_to_logical_map=physical_to_logical_map,
        )

    @staticmethod
    def init_by_mapping(
        server_args: ServerArgs,
        model_config: ModelConfig,
        physical_to_logical_map,
    ):
        if not isinstance(physical_to_logical_map, torch.Tensor):
            physical_to_logical_map = torch.tensor(physical_to_logical_map)
        physical_to_logical_map = physical_to_logical_map.to(server_args.device)

        common = ExpertLocationMetadata._init_common(server_args, model_config)
        model_config_for_expert_location = common["model_config_for_expert_location"]
        logical_to_all_physical_map = _compute_logical_to_all_physical_map(
            physical_to_logical_map,
            num_logical_experts=model_config_for_expert_location.num_logical_experts,
        )

        return ExpertLocationMetadata._init_raw(
            ep_size=common["ep_size"],
            physical_to_logical_map=physical_to_logical_map,
            logical_to_all_physical_map=logical_to_all_physical_map,
        )

    @staticmethod
    def init_by_eplb(
        server_args: ServerArgs, model_config: ModelConfig, logical_count: torch.Tensor
    ):
        if not isinstance(logical_count, torch.Tensor):
            logical_count = torch.tensor(logical_count)
        if len(logical_count.shape) == 2:
            logical_count = logical_count.unsqueeze(0)
        logical_count = logical_count.to(server_args.device)

        common = ExpertLocationMetadata._init_common(server_args, model_config)
        model_config_for_expert_location = common["model_config_for_expert_location"]
        num_physical_experts = common["num_physical_experts"]
        num_groups = model_config_for_expert_location.num_groups
        num_nodes = server_args.nnodes

        physical_to_logical_map, logical_to_all_physical_map, expert_count = (
            eplb_algorithms.rebalance_experts(
                tokens_per_expert=logical_count,
                num_physical_experts=num_physical_experts,
                num_local_physical_experts=num_physical_experts // common["ep_size"],
                num_groups=num_groups,
                num_nodes=num_nodes,
                algorithm=eplb_algorithms.compute_algorithm(
                    raw_algorithm=server_args.eplb_algorithm,
                    num_groups=num_groups,
                    num_nodes=num_nodes,
                ),
            )
        )

        return ExpertLocationMetadata._init_raw(
            ep_size=common["ep_size"],
            physical_to_logical_map=physical_to_logical_map.to(server_args.device),
            logical_to_all_physical_map=logical_to_all_physical_map.to(
                server_args.device
            ),
        )

    @staticmethod
    def _init_common(server_args: ServerArgs, model_config: ModelConfig):
        model_config_for_expert_location = (
            ModelConfigForExpertLocation.from_model_config(model_config)
        )

        num_physical_experts = (
            model_config_for_expert_location.num_logical_experts
            + server_args.ep_num_redundant_experts
        )
        ep_size = server_args.ep_size
        assert num_physical_experts % ep_size == 0
        num_local_physical_experts = num_physical_experts // ep_size

        return dict(
            model_config_for_expert_location=model_config_for_expert_location,
            num_physical_experts=num_physical_experts,
            num_local_physical_experts=num_local_physical_experts,
            ep_size=ep_size,
        )

    @staticmethod
    def _init_raw(
        ep_size: int,
        physical_to_logical_map: torch.Tensor,
        logical_to_all_physical_map: torch.Tensor,
    ):
        _, num_physical_experts = physical_to_logical_map.shape

        logical_to_all_physical_map_padded = F.pad(
            logical_to_all_physical_map,
            (0, num_physical_experts - logical_to_all_physical_map.shape[-1]),
            value=-1,
        )

        logical_to_all_physical_map_num_valid = torch.count_nonzero(
            logical_to_all_physical_map != -1, dim=-1
        )

        return ExpertLocationMetadata(
            physical_to_logical_map=physical_to_logical_map,
            logical_to_all_physical_map=logical_to_all_physical_map_padded,
            logical_to_all_physical_map_num_valid=logical_to_all_physical_map_num_valid,
            logical_to_rank_dispatch_physical_map=compute_logical_to_rank_dispatch_physical_map(
                logical_to_all_physical_map=logical_to_all_physical_map,
                logical_to_all_physical_map_num_valid=logical_to_all_physical_map_num_valid,
                num_gpus=ep_size,
                num_physical_experts=num_physical_experts,
                ep_rank=torch.distributed.get_rank(),
            ),
        )

    # -------------------------------- mutation ------------------------------------

    def update(
        self,
        other: "ExpertLocationMetadata",
    ):
        for field in [
            "ep_size",
        ]:
            assert getattr(self, field) == getattr(other, field)

        for field in [
            "physical_to_logical_map",
            "logical_to_all_physical_map",
            "logical_to_all_physical_map_num_valid",
            "logical_to_rank_dispatch_physical_map",
        ]:
            dst = getattr(self, field)
            dst[...] = getattr(other, field)

    # -------------------------------- usage ------------------------------------

    def logical_to_all_physical(
        self, layer_id: int, logical_expert_id: int
    ) -> List[int]:
        return [
            physical_expert_id
            for physical_expert_id in self.logical_to_all_physical_map[
                layer_id, logical_expert_id
            ].tolist()
            if physical_expert_id != -1
        ]

```

看一下里面的init\_by\_eplb，也就是通过具体的eplb算法进行重新平衡，首先是对common部分进行初始化，可以先不看，重点是第二部分，获取phy2log，log2phy和expert\_count这几个关键参数的部分，会调用eplb\_algorithms.rebalance\_experts，这里就是具体的eplb算法部分里，目前调用的实现就是dpsk的实现版本。算法代码位置在python/sglang/srt/managers/eplb\_algorithms/目录下。这里就不展示了。

在eplb init完成之后，回到eplb\_manager，会进行update\_expert\_location，也就是根据我们收集到的expert信息，更新新的专家分布，看一下这个update\_expert\_location函数（在model runner里）

```python
    def update_expert_location(
        self, new_expert_location_metadata: ExpertLocationMetadata
    ):
        self.expert_location_updater.update(
            self.model.routed_experts_weights_of_layer,
            new_expert_location_metadata,
            nnodes=self.server_args.nnodes,
            rank=self.tp_rank,
        )
```

### python/sglang/srt/model\_executor/expert\_location\_updater.py

可以看见没有什么额外操作，就是根据我们的新的专家元数据来进行更新。这里的expert\_location\_updater代码实现在python/sglang/srt/model\_executor/expert\_location\_updater.py里，如下：

```python
class ExpertLocationUpdater:
    def __init__(self):
        self._first_execution = True

    def update(
        self,
        routed_experts_weights_of_layer: Dict[int, List[torch.Tensor]],
        new_expert_location_metadata: ExpertLocationMetadata,
        nnodes: int,
        rank: int,
    ):
        if self._first_execution:
            self._first_execution = False
            torch.cuda.empty_cache()

        old_expert_location_metadata = get_global_expert_location_metadata()
        _update_expert_weights(
            routed_experts_weights_of_layer,
            old_expert_location_metadata,
            new_expert_location_metadata,
            nnodes,
            rank,
        )
        old_expert_location_metadata.update(new_expert_location_metadata)



```

会保留一下旧的元数据，然后根据新的元数据进行更新，具体更新函数是下面这个\_update\_expert\_weights.

```python
def _update_expert_weights(
    routed_experts_weights_of_layer: Dict[int, List[torch.Tensor]],
    old_expert_location_metadata: ExpertLocationMetadata,
    new_expert_location_metadata: ExpertLocationMetadata,
    nnodes: int,
    rank: int,
):
    temp_buffers = create_temp_buffers(
        next(iter(routed_experts_weights_of_layer.values()))
    )

    world_size = torch.distributed.get_world_size()
    num_local_physical_experts = old_expert_location_metadata.num_local_physical_experts
    num_gpu_per_node = world_size // nnodes

    old_physical_to_logical_map = (
        old_expert_location_metadata.physical_to_logical_map.tolist()
    )
    new_physical_to_logical_map = (
        new_expert_location_metadata.physical_to_logical_map.tolist()
    )

    for layer_id in sorted(routed_experts_weights_of_layer.keys()):
        update_expert_weights_single_layer(
            routed_experts_weights=routed_experts_weights_of_layer[layer_id],
            temp_buffers=temp_buffers,
            old_physical_to_logical_map=old_physical_to_logical_map[layer_id],
            new_physical_to_logical_map=new_physical_to_logical_map[layer_id],
            num_local_physical_experts=num_local_physical_experts,
            num_gpu_per_node=num_gpu_per_node,
            rank=rank,
        )
```

这里面会调用update\_expert\_weights\_single\_layer，也就是对每一层的专家权重进行更新，代码实现在同一个文件，如下：

```python
def update_expert_weights_single_layer(
    routed_experts_weights: List[torch.Tensor],
    temp_buffers: List[torch.Tensor],
    old_physical_to_logical_map: List[int],  # (num_physical_Experts,)
    new_physical_to_logical_map: List[int],  # (num_physical_Experts,)
    num_local_physical_experts: int,
    num_gpu_per_node: int,
    rank: int,
    debug: bool = False,
):
    assert all(
        tensor.shape[0] == num_local_physical_experts
        for tensor in routed_experts_weights
    ), f"{num_local_physical_experts=} {[x.shape for x in routed_experts_weights]=}"
    assert isinstance(old_physical_to_logical_map, list)
    assert isinstance(new_physical_to_logical_map, list)

    output_logs = [] if debug else None

    num_physical_experts = len(old_physical_to_logical_map)
    num_tensors = len(routed_experts_weights)

    self_node_id = rank // num_gpu_per_node

    local_expert_location_range = (
        rank * num_local_physical_experts,
        (rank + 1) * num_local_physical_experts,
    )

    def _entrypoint():
        # List[Tuple[logical_expert_id, List[P2POp]]]
        p2p_op_infos: List[Tuple[int, List[P2POp]]] = []
        # List[Tuple[temp_buffers_expert_location, routed_experts_weights_expert_location]]
        buffer2weight_copy_infos: List[Tuple[int, int]] = []

        _handle_recv(buffer2weight_copy_infos, p2p_op_infos)
        _create_isend_ops(p2p_op_infos)
        _execute_p2p_ops(p2p_op_infos)
        _execute_buffer2weight_copies(buffer2weight_copy_infos)

        if debug:
            output_logs.append(f"{p2p_op_infos=}")
            output_logs.append(f"{buffer2weight_copy_infos=}")

    def _handle_recv(buffer2weight_copy_infos, p2p_op_infos):
        for dst_expert_location in range(*local_expert_location_range):
            _handle_recv_of_dst_expert_location(
                dst_expert_location, buffer2weight_copy_infos, p2p_op_infos
            )

    def _handle_recv_of_dst_expert_location(
        dst_expert_location: int, buffer2weight_copy_infos, p2p_op_infos
    ):
        logical_expert_id = new_physical_to_logical_map[dst_expert_location]

        # case 1: unchanged
        if old_physical_to_logical_map[dst_expert_location] == logical_expert_id:
            if debug:
                output_logs.append(
                    f"handle_recv_of_dst_expert_location {dst_expert_location=} case=unchanged"
                )
            return

        # case 2: same-gpu
        for src_expert_location in range(*local_expert_location_range):
            if old_physical_to_logical_map[src_expert_location] == logical_expert_id:
                for i in range(num_tensors):
                    _get_tensor(temp_buffers, i, dst_expert_location).copy_(
                        _get_tensor(routed_experts_weights, i, src_expert_location)
                    )
                buffer2weight_copy_infos.append(
                    (dst_expert_location, dst_expert_location)
                )
                if debug:
                    output_logs.append(
                        f"handle_recv_of_dst_expert_location {dst_expert_location=} case=same-gpu {src_expert_location=}"
                    )
                return

        # case 3: free-rider
        for src_expert_location in range(
            rank * num_local_physical_experts, dst_expert_location
        ):
            if new_physical_to_logical_map[src_expert_location] == logical_expert_id:
                buffer2weight_copy_infos.append(
                    (src_expert_location, dst_expert_location)
                )
                if debug:
                    output_logs.append(
                        f"handle_recv_of_dst_expert_location {dst_expert_location=} case=free-rider {src_expert_location=}"
                    )
                return

        same_node_mapping, cross_node_mapping, need_comm_self_node_dst_ranks = (
            _compute_comm_info(logical_expert_id=logical_expert_id)
        )

        # case 4: same-node
        if rank in need_comm_self_node_dst_ranks:
            chosen_src_rank = same_node_mapping.chunk_value_from_element_value(
                element_value=rank
            )
            _create_p2p_recv_and_buffer2weight_copy(
                buffer2weight_copy_infos,
                p2p_op_infos,
                src_rank=chosen_src_rank,
                logical_expert_id=logical_expert_id,
                dst_expert_location=dst_expert_location,
            )
            if debug:
                output_logs.append(
                    f"handle_recv_of_dst_expert_location {dst_expert_location=} case=same-node {chosen_src_rank=}"
                )
            return

        # case 5: cross-node
        # Future work: can optimize when there are multiple ranks in the same dst node that uses the same logical expert
        chosen_src_rank = cross_node_mapping.chunk_value_from_element_value(
            element_value=rank
        )
        _create_p2p_recv_and_buffer2weight_copy(
            buffer2weight_copy_infos,
            p2p_op_infos,
            src_rank=chosen_src_rank,
            logical_expert_id=logical_expert_id,
            dst_expert_location=dst_expert_location,
        )
        if debug:
            output_logs.append(
                f"handle_recv_of_dst_expert_location {dst_expert_location=} case=cross-node {chosen_src_rank=}"
            )
        return

    def _create_p2p_recv_and_buffer2weight_copy(
        buffer2weight_copy_infos,
        p2p_op_infos,
        *,
        logical_expert_id: int,
        src_rank: int,
        dst_expert_location: int,
    ):
        p2p_op_infos.append(
            (
                logical_expert_id,
                [
                    P2POp(
                        op=torch.distributed.irecv,
                        tensor=_get_tensor(temp_buffers, i, dst_expert_location),
                        peer=src_rank,
                    )
                    for i in range(num_tensors)
                ],
            )
        )
        buffer2weight_copy_infos.append((dst_expert_location, dst_expert_location))

    def _create_isend_ops(p2p_op_infos):
        handled_logical_expert_ids = set()
        for src_expert_location in range(*local_expert_location_range):
            logical_expert_id = old_physical_to_logical_map[src_expert_location]

            if logical_expert_id in handled_logical_expert_ids:
                continue
            handled_logical_expert_ids.add(logical_expert_id)

            _create_isend_ops_of_logical_expert_id(
                logical_expert_id, src_expert_location, p2p_op_infos
            )

    def _create_isend_ops_of_logical_expert_id(
        logical_expert_id, src_expert_location, p2p_op_infos
    ):
        same_node_mapping, cross_node_mapping, need_comm_self_node_dst_ranks = (
            _compute_comm_info(logical_expert_id=logical_expert_id)
        )

        same_node_dst_ranks = same_node_mapping.element_values_from_chunk_value(
            chunk_value=rank
        )
        cross_node_dst_ranks = cross_node_mapping.element_values_from_chunk_value(
            chunk_value=rank
        )
        all_dst_ranks = same_node_dst_ranks + cross_node_dst_ranks

        if debug:
            output_logs.append(
                f"create_isend_ops_of_logical_expert_id {logical_expert_id=} {src_expert_location=} {same_node_dst_ranks=} {cross_node_dst_ranks=}"
            )

        p2p_op_infos.append(
            (
                logical_expert_id,
                [
                    P2POp(
                        op=torch.distributed.isend,
                        tensor=_get_tensor(
                            routed_experts_weights, i, src_expert_location
                        ),
                        peer=dst_rank,
                    )
                    for dst_rank in all_dst_ranks
                    for i in range(num_tensors)
                ],
            )
        )

    def _compute_comm_info(logical_expert_id: int):
        all_src_ranks = _deduplicate_ordered(
            [
                x // num_local_physical_experts
                for x in range(num_physical_experts)
                if old_physical_to_logical_map[x] == logical_expert_id
            ]
        )
        all_src_nodes = [x // num_gpu_per_node for x in all_src_ranks]
        self_node_src_ranks = [
            x for x in all_src_ranks if x // num_gpu_per_node == self_node_id
        ]

        need_comm_dst_ranks = _deduplicate_ordered(
            [
                x // num_local_physical_experts
                for x in range(num_physical_experts)
                if new_physical_to_logical_map[x] == logical_expert_id
                and x // num_local_physical_experts not in all_src_ranks
            ]
        )
        need_comm_self_node_dst_ranks = (
            [x for x in need_comm_dst_ranks if x // num_gpu_per_node == self_node_id]
            if len(self_node_src_ranks) > 0
            else []
        )
        need_comm_cross_node_dst_ranks = [
            x
            for x in need_comm_dst_ranks
            if (x // num_gpu_per_node) not in all_src_nodes
        ]

        same_node_mapping = _ChunkUtils(
            chunk_values=self_node_src_ranks,
            element_values=need_comm_self_node_dst_ranks,
        )

        cross_node_mapping = _ChunkUtils(
            chunk_values=all_src_ranks,
            element_values=need_comm_cross_node_dst_ranks,
        )

        return same_node_mapping, cross_node_mapping, need_comm_self_node_dst_ranks

    def _execute_p2p_ops(p2p_op_infos):
        sorted_infos = sorted(p2p_op_infos, key=lambda info: info[0])
        p2p_ops = [op for _, ops in sorted_infos for op in ops]
        if len(p2p_ops) == 0:
            return

        reqs = torch.distributed.batch_isend_irecv(p2p_ops)
        for req in reqs:
            req.wait()

    def _execute_buffer2weight_copies(buffer2weight_copy_infos):
        for (
            temp_buffers_expert_location,
            routed_experts_weights_expert_location,
        ) in buffer2weight_copy_infos:
            for i in range(num_tensors):
                _get_tensor(
                    routed_experts_weights, i, routed_experts_weights_expert_location
                ).copy_(_get_tensor(temp_buffers, i, temp_buffers_expert_location))

    def _get_tensor(tensors, tensor_index: int, expert_location: int) -> torch.Tensor:
        return tensors[tensor_index][_get_local_expert_location(expert_location)]

    def _get_local_expert_location(expert_location: int) -> int:
        assert (
            local_expert_location_range[0]
            <= expert_location
            < local_expert_location_range[1]
        )
        return expert_location % num_local_physical_experts

    _entrypoint()

    return output_logs

```

这个函数的具体作用就是根据我们之前更新的专家分布元数据信息，将新的分布实际的发送到对应位置去。细致的看一下整个函数的流程：

**变量准备**：检查输入 tensor 形状和映射类型。计算全局物理专家数、tensor 数量、当前节点 id、当前 rank 负责的物理专家区间。

核心入口 **\_entrypoint**()，所有的同步和搬运逻辑都在 \_entrypoint() 里完成，分为以下几步：

**步骤一：处理接收（\_handle\_recv）**：

遍历本 rank 负责的所有物理专家（dst\_expert\_location）。对每个目标物理专家，调用 \_handle\_recv\_of\_dst\_expert\_location，决定它的权重数据应该怎么获得。\_handle\_recv\_of\_dst\_expert\_location 内部有五种情况：

unchanged：该物理专家的逻辑 id 没变，无需搬运，直接跳过。

same-gpu：目标逻辑专家原本就在本 rank 的其它物理专家上，直接本地拷贝即可。

free-rider：目标逻辑专家在本 rank 的更早位置已经被搬运过，直接复用已有数据。

same-node：目标逻辑专家在同一节点的其它 rank 上，需要跨 rank 通信（P2P recv），并拷贝到本地。

cross-node：目标逻辑专家在其它节点的其它 rank 上，需要跨节点通信（P2P recv），并拷贝到本地。

每种情况会把需要的拷贝操作或通信操作信息记录到 buffer2weight\_copy\_infos 和 p2p\_op\_infos。

**步骤二：创建发送操作（\_create\_isend\_ops）**

遍历本 rank 负责的所有物理专家。

对每个逻辑专家，只处理一次（去重）。

对每个逻辑专家，查找需要发送到哪些其它 rank（同节点/跨节点），并把 isend 操作加入 p2p\_op\_infos。

**步骤三：执行 P2P 通信（\_execute\_p2p\_ops）**

把所有 isend/irecv 操作统一用 torch.distributed.batch\_isend\_irecv 执行，并等待完成。

**步骤四：执行本地 buffer 到权重的拷贝（\_execute\_buffer2weight\_copies）**

把临时 buffer 里的数据拷贝到最终的 routed\_experts\_weights 里，完成权重的最终落位。

**里面的工具函数：**

\_get\_tensor、\_get\_local\_expert\_location：用于定位 tensor 的具体位置。

\_compute\_comm\_info：根据逻辑专家 id，计算需要通信的源/目标 rank。

\_ChunkUtils：辅助做 rank 与专家的分组和分配。

\_deduplicate\_ordered：去重工具。