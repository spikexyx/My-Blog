---
title: SGLang EPLB 优化记录 - 加入通信惩罚的算法优化
date: 2025-07-23 11:35:38
tags:
    - SGLang
    - LLM
    - Inference Engine
categories:
    - AI Infra
cover: "imgs/wallhaven/wallhaven-2ymgg9.jpg"
excerpt: "SGLang EPLB 优化记录，算法部分。"
comment: false
---

## 新版算法

核心优化点：

1.  eplb\_manager：在EPLBManager里添加\_post\_rebalance\_handler函数，用来单独计算当前专家分布情况下，将逻辑专家移动到某物理专家号的惩罚，返回一个tensor作为新参数传入rebalance。（因为计算这个惩罚因子tensor本身很耗时，直接在算法本身里实现会让rebalance时间overhead太大，所以解耦出来单独实现）

2.  eplb algorithm：添加新的算法，不使用原生的hierarchical方式，整体采用全局重分布，但专家复制与专家平衡部分均采用向量化方式实现，减少计算时间开销。在load-balance部分嵌入传入的提前计算好的通讯惩罚tensor，计算出communication penalty factor，影响最终专家选择，达到减少移动的同时保持较好平衡度的效果。

总的来说就是让已经有一定平衡度的专家更倾向留在原地，避免原本的完全无状态的更新造成的大量无效移动。

~~核心算法代码：~~(该版本在多机测试中表现不佳，已弃用)

```python
from typing import Tuple, Optional

import torch

from sglang.srt.utils import get_bool_env_var

_REBALANCE_CALL_COUNT = 0

def rebalance_experts_with_affinity(
    weight: torch.Tensor,
    num_physical_experts: int,
    num_local_physical_experts: int,
    comm_penalty: Optional[torch.Tensor] = None,
):
    global _REBALANCE_CALL_COUNT
    if _REBALANCE_CALL_COUNT < 3:
        _REBALANCE_CALL_COUNT += 1

    num_layers, num_logical_experts = weight.shape
    assert num_physical_experts % num_local_physical_experts == 0
    num_gpus = num_physical_experts // num_local_physical_experts
    num_redundancy_experts = num_physical_experts - num_logical_experts

    physical_to_logical_map = torch.empty(
        num_layers,
        num_physical_experts,
        dtype=torch.int,
        device=weight.device,
    )
    logical_to_physical_map = torch.full(
        (num_layers, num_logical_experts, num_redundancy_experts + 1),
        -1,
        dtype=torch.int,
        device=weight.device,
    )
    logical_count = torch.ones(
        num_layers,
        num_logical_experts,
        dtype=torch.int,
        device=weight.device,
    )

    arange_num_moe_layers = torch.arange(
        num_layers, dtype = torch.int, device=weight.device
    )
    arange_num_logical_experts = torch.arange(
        num_logical_experts, dtype = torch.int, device=weight.device
    )
    
    physical_to_logical_map[:, :num_logical_experts] = arange_num_logical_experts[None, :]
    logical_to_physical_map[:, :, 0] = arange_num_logical_experts[None, :]

    # Replicate experts
    weight_all_diff = weight + arange_num_logical_experts * 1e-4
    for i in range(num_redundancy_experts):
        score = weight_all_diff / logical_count
        score1 = weight / (logical_count + 1)
        
        score1 = score1.view_as(score)
        values, indices = score.max(-1, keepdim=True)
        values = values.expand_as(score).contiguous()
        score.scatter_(-1, indices, score1.gather(-1, indices))
        values.scatter_(-1, indices, score.max(-1, keepdim=True).values)
        redundancy_indices = values.argmin(-1)
        physical_to_logical_map[:, num_logical_experts + i] = redundancy_indices
        redundancy_count = (
            logical_count.gather(-1, redundancy_indices.view(num_layers, 1)).squeeze(1)
        )
        
        physical_redundancy_indices = torch.full(
            (num_layers,),
            num_logical_experts + i,
            dtype=torch.int,
            device=weight.device
        )
        logical_to_physical_map[
            arange_num_moe_layers,
            redundancy_indices,
            redundancy_count,
        ] = physical_redundancy_indices
        logical_count[
            arange_num_moe_layers,
            redundancy_indices,
        ] += 1

    # Load-balance between devices
    if num_gpus > 1:
        if comm_penalty is not None:
            comm_penalty = comm_penalty.to(weight.device)

        physical_to_logical_map_int64 = physical_to_logical_map.to(torch.int64)
        counts = logical_count.gather(-1, physical_to_logical_map_int64)
        score = weight.gather(-1, physical_to_logical_map_int64)
        score = score / counts
        
        sorted_scores, sorted_indices = score.sort(-1, descending=True)

        gpu_loads = torch.ones(num_layers, num_gpus, dtype=score.dtype, device=weight.device)
        gpu_ep_counts = torch.zeros(num_layers, num_gpus, dtype=torch.long, device=weight.device)

        # balanced_indices = torch.full_like(score, -1, dtype=torch.long, device=weight.device)
        sorted_expert_final_pos = torch.full_like(sorted_indices, -1)

        for i in range(num_physical_experts):
            expert_score = sorted_scores[:, i]
            expert_idx = sorted_indices[:, i]

            current_logical_experts = physical_to_logical_map.gather(-1, expert_idx.unsqueeze(1)).squeeze(1)
            
            masked_gpu_loads = gpu_loads.clone()
            # full_gpus_mask = (gpu_ep_counts >= num_local_physical_experts)
            # masked_gpu_loads[full_gpus_mask] = torch.finfo(score.dtype).max

            # calculate move penalty
            if comm_penalty is not None and _REBALANCE_CALL_COUNT >= 3:
                gpu_ids = torch.arange(num_gpus, device=weight.device).unsqueeze(0)
                # next_slots = gpu_ids * num_local_physical_experts + gpu_ep_counts
                next_slots = gpu_ids * num_local_physical_experts

                # next_slots = torch.clamp(next_slots, 0, num_physical_experts - 1)

                logical_experts_expanded = current_logical_experts.unsqueeze(1).expand(-1, num_gpus)
                layer_indices = torch.arange(num_layers, device=weight.device).unsqueeze(1).expand_as(logical_experts_expanded)

                # alpha = 1.0
                penalty_values = comm_penalty[layer_indices, logical_experts_expanded, next_slots]
                # penalty_factor = 1.0 + alpha * penalty_values
                penalty_factor = 1.0 + penalty_values

                # masked_gpu_loads = masked_gpu_loads + 1.0
                masked_gpu_loads = penalty_factor * masked_gpu_loads

            full_gpus_mask = (gpu_ep_counts >= num_local_physical_experts)
            masked_gpu_loads[full_gpus_mask] = torch.finfo(score.dtype).max

            target_gpu = masked_gpu_loads.argmin(dim=1)
            slot_on_gpu = gpu_ep_counts.gather(1, target_gpu.unsqueeze(1)).squeeze(1)

            final_pos = target_gpu * num_local_physical_experts + slot_on_gpu

            sorted_expert_final_pos[:, i] = final_pos

            gpu_loads.scatter_add_(1, target_gpu.unsqueeze(1), expert_score.unsqueeze(1))
            gpu_ep_counts.scatter_add_(1, target_gpu.unsqueeze(1), torch.ones_like(target_gpu.unsqueeze(1)))

        balanced_indices = torch.full_like(sorted_indices, -1)
        balanced_indices.scatter_(-1, sorted_expert_final_pos, sorted_indices)

        physical_to_logical_map = physical_to_logical_map.gather(-1, balanced_indices)

        mask = logical_to_physical_map == -1
        logical_to_physical_map[mask] = 0

        inverse_balanced_indices = balanced_indices.argsort(-1)
        logical_to_physical_map = inverse_balanced_indices.gather(
            -1, logical_to_physical_map.view(num_layers, -1).to(torch.int64)
        ).view_as(logical_to_physical_map).to(torch.int)
        
        logical_to_physical_map[mask] = -1

    return physical_to_logical_map, logical_to_physical_map, logical_count

def rebalance_experts(
    weight: torch.Tensor,
    num_physical_experts: int,
    num_local_physical_experts: int,
    comm_penalty: Optional[torch.Tensor] = None,
):
    weight = weight.float().cpu()
    phy2log, log2phy, logcnt = rebalance_experts_with_affinity(
        weight, num_physical_experts, num_local_physical_experts, comm_penalty
    )
    return phy2log, log2phy, logcnt


__all__ = ["rebalance_experts"]
```

核心算法代码：（2025.08.05 update）

```python
from typing import Tuple, Optional

import torch

from sglang.srt.utils import get_bool_env_var

_REBALANCE_CALL_COUNT = 0

def balanced_packing(
    weight: torch.Tensor, num_packs: int
) -> Tuple[torch.Tensor, torch.Tensor]:
    """
    Pack n weighted objects to m packs, such that each bin contains exactly n/m objects and the weights of all packs
    are as balanced as possible.
    Parameters:
        weight: [X, n], the weight of each item
        num_packs: number of packs
    Returns:
        pack_index: [X, n], the pack index of each item
        rank_in_pack: [X, n], the rank of the item in the pack
    """
    num_layers, num_groups = weight.shape
    assert num_groups % num_packs == 0
    groups_per_pack = num_groups // num_packs

    if groups_per_pack == 1:
        pack_index = torch.arange(
            weight.size(-1), dtype=torch.int64, device=weight.device
        ).expand(weight.shape)
        rank_in_pack = torch.zeros_like(weight, dtype=torch.int64)
        return pack_index, rank_in_pack

    indices = weight.float().sort(-1, descending=True).indices.cpu()
    pack_index = torch.full_like(weight, fill_value=-1, dtype=torch.int64, device="cpu")
    rank_in_pack = torch.full_like(pack_index, fill_value=-1)
    for i in range(num_layers):
        pack_weights = [0] * num_packs
        pack_items = [0] * num_packs
        for group in indices[i]:
            pack = min(
                (i for i in range(num_packs) if pack_items[i] < groups_per_pack),
                key=pack_weights.__getitem__,
            )
            assert pack_items[pack] < groups_per_pack
            pack_index[i, group] = pack
            rank_in_pack[i, group] = pack_items[pack]
            pack_weights[pack] += weight[i, group]
            pack_items[pack] += 1
    return pack_index, rank_in_pack

def balanced_packing_vectorized(
    weight: torch.Tensor, num_packs: int
) -> Tuple[torch.Tensor, torch.Tensor]:
    num_layers, num_groups = weight.shape
    assert num_groups % num_packs == 0
    groups_per_pack = num_groups // num_packs

    if groups_per_pack == 1:
        pack_index = torch.arange(
            weight.size(-1), dtype=torch.int64, device=weight.device
        ).expand(weight.shape)
        rank_in_pack = torch.zeros_like(weight, dtype=torch.int64)
        return pack_index, rank_in_pack

    # [num_layers, num_groups]
    indices = weight.float().sort(-1, descending=True).indices

    pack_index = torch.full_like(weight, fill_value=-1, dtype=torch.int64)
    rank_in_pack = torch.full_like(weight, fill_value=-1, dtype=torch.int64)

    pack_weights = torch.zeros(num_layers, num_packs, dtype=weight.dtype, device=weight.device)  # [num_layers, num_packs]
    pack_items = torch.zeros(num_layers, num_packs, dtype=torch.int64, device=weight.device)     # [num_layers, num_packs]

    for j in range(num_groups):
        groups = indices[:, j]  # [num_layers]

        available_mask = (pack_items < groups_per_pack)  # [num_layers, num_packs]

        masked_pack_weights = torch.where(available_mask, pack_weights, torch.inf)
        packs = torch.argmin(masked_pack_weights, dim=1)  # [num_layers]

        rank_in_pack_current = pack_items[torch.arange(num_layers), packs]  # [num_layers]
        pack_index[torch.arange(num_layers), groups] = packs
        rank_in_pack[torch.arange(num_layers), groups] = rank_in_pack_current

        pack_weights.scatter_add_(1, packs.unsqueeze(1), weight[torch.arange(num_layers), groups].unsqueeze(1))
        pack_items[torch.arange(num_layers), packs] += 1

    return pack_index, rank_in_pack

def balanced_packing_vectorized_with_comm(
    weight: torch.Tensor, 
    num_packs: int,
    phy2mlog: Optional[torch.Tensor] = None,
    comm_penalty: Optional[torch.Tensor] = None,
) -> Tuple[torch.Tensor, torch.Tensor]:
    """
    Pack n weighted objects to m packs, such that each bin contains exactly n/m objects and the weights of all packs
    are as balanced as possible.
    
    Parameters:
        weight: [X, n], the weight of each item
        num_packs: number of packs
        comm_penalty: [X, n, num_packs], communication penalty for placing item j in pack k for layer i
    Returns:
        pack_index: [X, n], the pack index of each item
        rank_in_pack: [X, n], the rank of the item in the pack
    """
    num_layers, num_groups = weight.shape
    assert num_groups % num_packs == 0
    groups_per_pack = num_groups // num_packs

    if groups_per_pack == 1:
        pack_index = torch.arange(
            weight.size(-1), dtype=torch.int64, device=weight.device
        ).expand(weight.shape)
        rank_in_pack = torch.zeros_like(weight, dtype=torch.int64)
        return pack_index, rank_in_pack

    # [num_layers, num_groups]
    indices = weight.float().sort(-1, descending=True).indices

    pack_index = torch.full_like(weight, fill_value=-1, dtype=torch.int64)
    rank_in_pack = torch.full_like(weight, fill_value=-1, dtype=torch.int64)

    pack_weights = torch.zeros(num_layers, num_packs, dtype=weight.dtype, device=weight.device)  # [num_layers, num_packs]
    pack_items = torch.zeros(num_layers, num_packs, dtype=torch.int64, device=weight.device)     # [num_layers, num_packs]

    for j in range(num_groups):
        groups = indices[:, j]  # [num_layers] - phy_ep id

        available_mask = (pack_items < groups_per_pack)  # [num_layers, num_packs]

        base_costs = pack_weights.clone()  # [num_layers, num_packs]

        if comm_penalty is not None and _REBALANCE_CALL_COUNT >= 3:
            logical_expert_ids = phy2mlog[torch.arange(num_layers), groups]

            current_penalties = comm_penalty[
                torch.arange(num_layers),  
                logical_expert_ids,
                :
            ].view(num_layers, num_packs, groups_per_pack)[:, :, 0]

            penalty_factor = 1.0 + current_penalties

            adjusted_costs = base_costs * penalty_factor
        else:
            adjusted_costs = base_costs

        masked_adjusted_costs = torch.where(available_mask, adjusted_costs, torch.inf)
        packs = torch.argmin(masked_adjusted_costs, dim=1)  # [num_layers]

        rank_in_pack_current = pack_items[torch.arange(num_layers), packs]  # [num_layers]
        pack_index[torch.arange(num_layers), groups] = packs
        rank_in_pack[torch.arange(num_layers), groups] = rank_in_pack_current

        pack_weights.scatter_add_(1, packs.unsqueeze(1), weight[torch.arange(num_layers), groups].unsqueeze(1))
        pack_items[torch.arange(num_layers), packs] += 1

    return pack_index, rank_in_pack

def replicate_experts(
    weight: torch.Tensor, num_phy: int
) -> Tuple[torch.Tensor, torch.Tensor, torch.Tensor]:
    """
    Replicate `num_log` experts to `num_phy` replicas, such that the maximum load of all replicas is minimized.
    Parameters:
        weight: [X, num_log]
        num_phy: total number of experts after replication
    Returns:
        phy2log: [X, num_phy], logical expert id of each physical expert
        rank: [X, num_phy], the replica rank
        logcnt: [X, num_log], number of replicas for each logical expert
    """
    n, num_log = weight.shape
    num_redundant = num_phy - num_log
    assert num_redundant >= 0
    device = weight.device
    phy2log = torch.arange(num_phy, dtype=torch.int64, device=device).repeat(n, 1)
    rank = torch.zeros(n, num_phy, dtype=torch.int64, device=device)
    logcnt = torch.ones(n, num_log, dtype=torch.int64, device=device)
    arangen = torch.arange(n, dtype=torch.int64, device=device)
    for i in range(num_log, num_phy):
        redundant_indices = (weight / logcnt).max(dim=-1).indices
        phy2log[:, i] = redundant_indices
        rank[:, i] = logcnt[arangen, redundant_indices]
        logcnt[arangen, redundant_indices] += 1
    return phy2log, rank, logcnt


def rebalance_experts_hierarchical(
    weight: torch.Tensor,
    num_physical_experts: int,
    num_groups: int,
    num_nodes: int,
    num_gpus: int,
    comm_penalty: Optional[torch.Tensor] = None,
):
    """
    Parameters:
        weight: [num_moe_layers, num_logical_experts]
        num_physical_experts: number of physical experts after replication
        num_groups: number of expert groups
        num_nodes: number of server nodes, where the intra-node network (e.g, NVLink) is faster
        num_gpus: number of GPUs, must be a multiple of `num_nodes`
    Returns:
        physical_to_logical_map: [num_moe_layers, num_physical_experts]
        logical_to_physical_map: [num_moe_layers, num_logical_experts, X]
        logical_count: [num_moe_layers, num_logical_experts]
    """
    num_layers, num_logical_experts = weight.shape
    assert num_logical_experts % num_groups == 0
    group_size = num_logical_experts // num_groups
    assert num_groups % num_nodes == 0
    groups_per_node = num_groups // num_nodes
    assert num_gpus % num_nodes == 0
    assert num_physical_experts % num_gpus == 0
    phy_experts_per_gpu = num_physical_experts // num_gpus

    def inverse(perm: torch.Tensor) -> torch.Tensor:
        inv = torch.empty_like(perm)
        inv.scatter_(
            1,
            perm,
            torch.arange(perm.size(1), dtype=torch.int64, device=perm.device).expand(
                perm.shape
            ),
        )
        return inv

    # Step 1: pack groups to nodes
    tokens_per_group = weight.unflatten(-1, (num_groups, group_size)).sum(-1)
    group_pack_index, group_rank_in_pack = balanced_packing(tokens_per_group, num_nodes)
    log2mlog = (
        (
            (group_pack_index * groups_per_node + group_rank_in_pack) * group_size
        ).unsqueeze(-1)
        + torch.arange(group_size, dtype=torch.int64, device=group_pack_index.device)
    ).flatten(-2)
    mlog2log = inverse(log2mlog)

    # Step 2: construct redundant experts within nodes
    # [num_layers * num_nodes, num_logical_experts // num_nodes]
    tokens_per_mlog = weight.gather(-1, mlog2log).view(
        -1, num_logical_experts // num_nodes
    )
    phy2mlog, phyrank, mlogcnt = replicate_experts(
        tokens_per_mlog, num_physical_experts // num_nodes
    )

    # Step 3: pack physical_experts to GPUs
    # [num_layers * num_nodes, num_physical_experts // num_nodes]
    tokens_per_phy = (tokens_per_mlog / mlogcnt).gather(-1, phy2mlog)
    pack_index, rank_in_pack = balanced_packing_vectorized_with_comm(tokens_per_phy, num_gpus // num_nodes, phy2mlog, comm_penalty)
    phy2pphy = pack_index * phy_experts_per_gpu + rank_in_pack
    pphy2phy = inverse(phy2pphy)

    pphy2mlog = phy2mlog.gather(
        -1, pphy2phy
    )  # [num_layers * num_nodes, num_log_per_nodes]
    pphy2mlog = (
        pphy2mlog.view(num_layers, num_nodes, -1)
        + torch.arange(
            0,
            num_logical_experts,
            num_logical_experts // num_nodes,
            device=group_pack_index.device,
        ).view(1, -1, 1)
    ).flatten(-2)
    pphy2log = mlog2log.gather(-1, pphy2mlog)
    pphyrank = phyrank.gather(-1, pphy2phy).view(num_layers, -1)
    logcnt = mlogcnt.view(num_layers, -1).gather(-1, log2mlog)
    return pphy2log, pphyrank, logcnt


def rebalance_experts(
    weight: torch.Tensor,
    num_replicas: int,
    num_gpus: int,
    comm_penalty: Optional[torch.Tensor] = None,
) -> Tuple[torch.Tensor, torch.Tensor, torch.Tensor]:
    """
    Entry point for expert-parallelism load balancer.
    Parameters:
        weight: [layers, num_logical_experts], the load statistics for all logical experts
        num_replicas: number of physical experts, must be a multiple of `num_gpus`
        num_groups: number of expert groups
        num_nodes: number of server nodes, where the intra-node network (e.g, NVLink) is faster
        num_gpus: number of GPUs, must be a multiple of `num_nodes`
    Returns:
        physical_to_logical_map: [layers, num_replicas], the expert index of each replica
        logical_to_physical_map: [layers, num_logical_experts, X], the replica indices for each expert
        expert_count: [layers, num_logical_experts], number of physical replicas for each logical expert
    """
    global _REBALANCE_CALL_COUNT
    if _REBALANCE_CALL_COUNT < 3:
        _REBALANCE_CALL_COUNT += 1

    num_layers, num_logical_experts = weight.shape
    weight = weight.float().cpu()
    # use global load-balance policy
    phy2log, phyrank, logcnt = rebalance_experts_hierarchical(
        weight, num_replicas, 1, 1, num_gpus, comm_penalty
    )
    maxlogcnt = logcnt.max().item()
    log2phy: torch.Tensor = torch.full(
        (num_layers, num_logical_experts, maxlogcnt),
        -1,
        dtype=torch.int64,
        device=logcnt.device,
    )
    log2phy.view(num_layers, -1).scatter_(
        -1,
        phy2log * maxlogcnt + phyrank,
        torch.arange(num_replicas, dtype=torch.int64, device=log2phy.device).expand(
            num_layers, -1
        ),
    )
    return phy2log, log2phy, logcnt


__all__ = ["rebalance_experts"]
```

## 双机测试

++(2025.08.05 update: 换用新版算法进行)++

### 测试环境

2 \* 8 \* H20

![截屏2025-07-23 10.53.30.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/jP2lRYjBLP0oEO8g/img/33bdeba7-02ce-4f50-9fe9-b812963618c0.png?x-oss-process=image/crop,x_0,y_0,w_1462,h_1493/ignore-error,1)

测试指令：

```shell
python3 -m sglang.bench_serving --backend sglang --dataset-name random --random-input 2560 --random-output 2560 --random-range-ratio 1.0 --num-prompts 800
```

### 优化结果汇总

表1：ep rebalance相关指标

|               | rebalance时间(s) | rebalance中的计算时间(s) | rebalance中的通讯时间(s) | 稳定期平衡度(%) | rebalance通讯操作数占比 |
| ------------- | ---------------- | ------------------------ | ------------------------ | --------------- | ----------------------- |
| deepseek      | ~1.09            | ~0.88                    | ~0.20                    | 88              | ~70%                    |
| deepseek\_opt | ~0.50            | ~0.40                    | ~0.09                    | 85              | ~20%                    |

表2：benchmark相关指标

|               | 总吞吐(tok/s) | Mean E2E Lat.(ms) | Mean TTFT(ms) | Mean ITL(ms) |
| ------------- | ------------- | ----------------- | ------------- | ------------ |
| deepseek      | 6508.52       | 420204.25         | 70260.20      | 136.75       |
| deepseek\_opt | 6806.97       | 399745.02         | 67296.81      | 129.91       |

### 原生deepseek算法

启动指令：

```shell
# node 0
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path Qwen/Qwen3-30B-A3B --tp-size 16 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30000 --dist-init-addr 172.16.10.136:50000 --nnodes 2 --node-rank 0

# node 1
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path Qwen/Qwen3-30B-A3B --tp-size 16 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30000 --dist-init-addr 172.16.10.136:50000 --nnodes 2 --node-rank 1
```

**eplb rebalance 通讯与计算情况**：

通讯：（一般在 0.20s 左右）

![截屏2025-08-06 13.53.45.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/e62a6391-2523-48a3-a29a-ff8c59e88873.png)

计算：（一般在 0.88s 左右）

![截屏2025-08-06 13.53.29.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/ef846360-92c5-493c-b979-bd82959d2b22.png)

总：（一般在 1.09s 左右）

![截屏2025-08-06 13.53.45.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/3e3af9dd-aa5b-4c47-98c0-39a931b6200c.png)

**通讯信息**：（一般情况专家移动通讯操作数占比70%左右）

![截屏2025-08-06 13.54.20.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/0d030af4-9e3a-417b-8d25-f81353f54e47.png)

**平衡度情况**：（稳定期一般在 0.88 左右）

![截屏2025-08-06 13.54.43.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/e2ad7756-aa40-4de6-8395-3557c609c733.png)

**最终结果**：

![截屏2025-08-06 14.01.55.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/a97bec93-46fb-44f3-8c28-51141e4a1329.png)

### 新版deepseek\_opt算法

```shell
# node 0
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path Qwen/Qwen3-30B-A3B --tp-size 16 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.2 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30000 --dist-init-addr 172.16.10.136:50000 --nnodes 2 --node-rank 0

# node 1
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path Qwen/Qwen3-30B-A3B --tp-size 16 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.2 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30000 --dist-init-addr 172.16.10.136:50000 --nnodes 2 --node-rank 1
```

**eplb通讯情况：**

通讯：（一般在 0.09s 左右）

![截屏2025-08-06 14.09.59.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/5490c06b-8b77-4f12-95e5-c2531b323be7.png)

计算：（一般在 0.40s 左右）

![截屏2025-08-06 14.10.49.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/afe7788c-b7b6-4cad-bca7-edb2aecfcefe.png)

总：（一般在 0.50s 左右）

![截屏2025-08-06 14.10.11.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/2d20e96f-0124-4bbc-a19c-e0059e173051.png)

**通讯信息：**（专家移动通讯操作数一般占比在 20% 左右）

![截屏2025-08-06 14.10.27.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/89cc126d-de23-4a42-bff7-8c3469a1c15b.png)

**平衡度情况**：（稳定期一般在 0.85 ~ 0.86 左右，相比原版算法会有少许下降，但通过微调新参数可以基本不影响性能）

![截屏2025-08-06 14.09.35.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/e360d9ed-6e71-4cae-97f1-ca70e0260925.png)

**最终结果**：

![截屏2025-08-06 14.27.26.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/bd6f15f4-289e-4434-ad7a-81399aa2a441.png)

## 四机测试

++(2025.08.05 update：换用新版算法进行)++

### 测试环境

4 \* 8 \* H20-3e

测试命令：

```shell
python3 -m sglang.bench_serving --backend sglang --dataset-name random --random-input 2560 --random-output 2560 --random-range-ratio 1.0 --num-prompts 800
```

### 优化结果汇总

表1：ep rebalance相关指标

|               | rebalance时间(s) | rebalance中的计算时间(s) | rebalance中的通讯时间(s) | 稳定期平衡度(%) | rebalance通讯操作数占比 |
| ------------- | ---------------- | ------------------------ | ------------------------ | --------------- | ----------------------- |
| deepseek      | ~1.44            | ~1.13                    | ~0.30                    | 88              | ~85%                    |
| deepseek\_opt | ~0.57            | ~0.43                    | ~0.14                    | 85              | ~35%                    |

表2：benchmark相关指标

|               | 总吞吐(tok/s) | Mean E2E Lat.(ms) | Mean TTFT(ms) | Mean ITL(ms) |
| ------------- | ------------- | ----------------- | ------------- | ------------ |
| deepseek      | 9536.16       | 429281.23         | 18994.86      | 160.33       |
| deepseek\_opt | 10241.21      | 399687.57         | 18789.23      | 148.85       |

### 原生deepseek算法

```shell
# node 0
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 0

# node 1
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 1

# node 2
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 2

# node 3
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 3
```

**eplb rebalance 情况**：

通讯：~ 0.3s

![截屏2025-08-06 11.11.49.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/8835f76b-a957-43c2-9ee7-998d05bec4df.png)

计算：~ 1.13s

![截屏2025-08-06 11.12.05.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/bdc5709a-42d4-46e5-abf7-0500559d137d.png)

总：~ 1.44s

![截屏2025-08-06 11.11.49.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/72c159ce-cb0a-462d-9507-26169f786bd4.png)

**通讯信息**：~ 85% ~ 90%

![截屏2025-08-06 11.05.31.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/9cf00a72-d737-4ca3-9f1f-0b2f00aa608f.png)

**平衡度情况**：~0.87 ~0.88

![截屏2025-08-06 11.12.48.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/5055d1fe-fa3c-47af-ade8-6531e024e092.png)

**最终结果**：

![截屏2025-08-05 14.06.24.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/e690e861-a506-4177-a8f7-1c2a73e5fc03.png)

### 新版deepseek\_opt算法

```shell
# node 0
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.1 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 0

# node 1
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.1 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 1

# node 2
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.1 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 2

# node 3
NCCL_SOCKET_IFNAME=eth0 GLOO_SOCKET_IFNAME=eth0 NCCL_IB_GID_INDEX=3 python3 -m sglang.launch_server --model-path /mnt/nfs/cenxiao/models/Qwen/Qwen3-30B-A3B --tp-size 32 --ep-size 32 --enable-expert-distribution-metrics --enable-ep-moe --enable-eplb --expert-distribution-recorder-mode stat --ep-num-redundant-experts 128 --eplb-rebalance-num-iterations 200 --mem-fraction-static 0.8 --disable-cuda-graph --trust-remote-code --init-expert-location trivial --eplb-algorithm deepseek_opt --eplb-intra-node-penalty 0.1 --eplb-inter-node-penalty 0.4 --host 0.0.0.0 --port 30100 --dist-init-addr 172.16.1.250:50100 --nnodes 4 --node-rank 3
```

**eplb rebalance 情况**：

通讯：~ 0.14s （下降越50%+）

![截屏2025-08-06 11.25.11.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/0a3291bb-6af9-4800-8cb0-c1ce23973254.png)

计算：~ 0.43s （下降约60%+）

![截屏2025-08-06 11.25.36.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/b2ce7dda-fe87-4889-a5c9-c278a88c8482.png)

总：~ 0.57s （下降约60%）

![截屏2025-08-06 11.25.21.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/a5b61735-3669-4845-95ee-9907ecd15810.png)

**通讯信息**：~ 35% ~ 40% （下降约55%）

![截屏2025-08-05 16.01.01.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/aa934fa8-b55c-4b9c-89be-1b3557033d40.png)

**平衡度情况**：~ 0.84 ~ 0.85

![截屏2025-08-06 11.31.10.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/32f43964-7e9b-4d91-b193-61689348f1af.png)

**最终结果**：

![截屏2025-08-05 13.58.36.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/3BMqYybppMewbqwZ/img/8c5b7493-0bb7-46bb-81d5-cb7abcc20c11.png)