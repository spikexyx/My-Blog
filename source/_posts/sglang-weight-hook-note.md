---
title: SGLang中模型权重内存信息抓取
date: 2025-06-17 10:26:03
tags:
    - SGLang
    - LLM
    - Inference Engine
categories:
    - AI Infra
cover: "imgs/wallhaven/wallhaven-2ymgg9.jpg"
excerpt: "针对SGLang框架的模型权重内存信息（包含设备指针等）的抓取笔记。"
comment: false
---

## 直接修改框架进行模型权重抓取方案

### 初始化阶段抓取 - 在ModelRunner.load\_model完成后进行插入抓取

模型加载位置定位：

SGLang的启动流程是在主进程开启tokenizer，然后在子进程开启scheduler和detokenizer，权重抓取可以不关心tokenizer和detokenizer，重点在scheduler子进程，其中，scheduler会调用tp worker，而tp worker会管理model runner，model runner才是实际进行模型加载和内存池管理的单元。

在python/sglang/srt/model\_executor/model\_runner.py的ModelRunner类里，有load\_model成员函数，该函数会进行模型加载（向下是会调用实际的模型的load\_model，向上是交给tp worker管理，在这一层抓就行）。

在ModelRunner类加入register weight hooks函数：

```python
# Weights_hook function 
    def _register_weight_hooks(self):
        self.weight_infos = {}  # Save weight metadatas
        
        def tensor_hook(tensor: torch.Tensor, name: str):
            if tensor.is_cuda:
                self.weight_infos[name] = {
                    "ptr": tensor.data_ptr(),
                    "size": tensor.numel() * tensor.element_size(),
                    # "actual_size": tensor.storage().size() * tensor.element_size(),
                    "device": str(tensor.device),
                    "dtype": str(tensor.dtype),
                    "shape": list(tensor.shape)
                }

        if not self._acquire_weight_lock():
            raise RuntimeError("Failed to acquire weight metadata update lock")
    
        # Register hooks to capture the initial state of model weights
        for name, param in self.model.named_parameters():
            tensor_hook(param, name)  # Capture parameter weights
        self._save_weight_meta()  # Save weight metadata to a local file
        self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")
        self._save_total_weight_meta()
        # self._merge_weights()  # Merge weights based on pointer continuity
        # self._save_merged_weight_meta()  # Save merged weight metadata to a local file
        self._release_weight_lock()

    # Save the model weight metadata to a JSON file
    def _save_weight_meta(self):
        os.makedirs("weights_metadata", exist_ok=True)
        meta_path = os.path.join("weights_metadata", f"weights_meta_{self.gpu_id}.json")
        # meta_path = f"weights_meta_{self.gpu_id}.json"
        try:
            with open(meta_path, 'w') as f:
                json.dump(self.weight_infos, f, indent=2)
            logger.info(f"Save weight metadata to {meta_path}.")
        except IOError as e:
            logger.error(f"Failed to save weight metadata to {meta_path}: {e}")
            raise

    def _save_total_weight_meta(self):
        os.makedirs("weights_metadata", exist_ok=True)
        meta_path = os.path.join("weights_metadata", f"total_weight_meta_{self.gpu_id}.json")
        # meta_path = f"weights_meta_{self.gpu_id}.json"
        try:
            with open(meta_path, 'w') as f:
                json.dump(self.total_weight_dict, f, indent=2)
            logger.info(f"Save total weight metadata to {meta_path}.")
        except IOError as e:
            logger.error(f"Failed to save total weight metadata to {meta_path}: {e}")
            raise
        
    def _calculate_device_weight_sizes(self, unit: str = "bytes") -> dict:
        """Calculate the total size of weights per device in self.weight_infos.
        
        Args:
            unit (str): The unit to return the size in. 
                       Options: "bytes", "KB", "MB", "GB".
        
        Returns:
            dict: {device: total_size} where total_size is in the specified unit.
        """
        device_sizes = {}  # {device: total_size_in_bytes}
        
        # 遍历所有 weight_infos，按 device 累加 size
        for info in self.weight_infos.values():
            device = info["device"]
            size = info["size"]
            if device in device_sizes:
                device_sizes[device] += size
            else:
                device_sizes[device] = size
        
        # 单位转换
        unit = unit.upper()
        if unit == "KB":
            return {device: size / 1024 for device, size in device_sizes.items()}
        elif unit == "MB":
            return {device: size / (1024 ** 2) for device, size in device_sizes.items()}
        elif unit == "GB":
            return {device: size / (1024 ** 3) for device, size in device_sizes.items()}
        else:  # Default to bytes
            return device_sizes

```

### 模型权重更新的处理

见附：关于SGLang中的模型权重更新相关一节，有对于几个更新函数的介绍，这里目前考虑到主要关心推理阶段，所以重点只在update\_weights\_from\_disk以及update\_weights\_from\_tensor两者里加入我们的权重hook，抓取所需信息。

```python
    # Functions for recording weight metadata during inference when update model weights
    def _handle_weight_update_hooks(self):
        """
        Handle weight updates during inference - clean old data and capture new weight information
        """
        logger.info("Starting weight update hook processing...")
        if not self._acquire_weight_lock():
            raise RuntimeError("Failed to acquire weight metadata update lock")
        
        # Clear old weight information
        self._clear_old_weight_data()
        
        # Re-register hooks to capture updated weights
        self._register_updated_weight_hooks()
        
        # Save updated metadata
        self._save_updated_weight_metadata()

        self._release_weight_lock()
        
        logger.info("Weight update hook processing completed.")

    def _clear_old_weight_data(self):
        """
        Clear old weight information and metadata files
        """
        # Clear in-memory data
        if hasattr(self, 'weight_infos'):
            self.weight_infos.clear()
        else:
            self.weight_infos = {}
        
        if hasattr(self, 'total_weight_dict'):
            self.total_weight_dict.clear()
        else:
            self.total_weight_dict = {}
        
        # Remove old metadata files
        try:
            weights_dir = "weights_metadata"
            if os.path.exists(weights_dir):
                old_weight_file = os.path.join(weights_dir, f"weights_meta_{self.gpu_id}.json")
                old_total_file = os.path.join(weights_dir, f"total_weight_meta_{self.gpu_id}.json")
                
                if os.path.exists(old_weight_file):
                    os.remove(old_weight_file)
                    logger.info(f"Removed old weight metadata file: {old_weight_file}")
                
                if os.path.exists(old_total_file):
                    os.remove(old_total_file)
                    logger.info(f"Removed old total weight metadata file: {old_total_file}")
                    
        except Exception as e:
            logger.warning(f"Failed to clean old metadata files: {e}")

    def _register_updated_weight_hooks(self):
        """
        Register hooks for updated model weights (similar to _register_weight_hooks but for updates)
        """
        def tensor_hook(tensor: torch.Tensor, name: str):
            if tensor.is_cuda:
                self.weight_infos[name] = {
                    "ptr": tensor.data_ptr(),
                    "size": tensor.numel() * tensor.element_size(),
                    "device": str(tensor.device),
                    "dtype": str(tensor.dtype),
                    "shape": list(tensor.shape),
                    "updated": True  # Mark as updated weight
                }
        
        # Capture updated model weights
        # logger.info("Capturing updated model weights...")
        # weight_count = 0
        for name, param in self.model.named_parameters():
            tensor_hook(param, name)
            # weight_count += 1
        
        # logger.info(f"Captured {weight_count} updated weight tensors")
        
        # Calculate device weight sizes for updated weights
        self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")

    def _save_updated_weight_metadata(self):
        """
        Save updated weight metadata to JSON files
        """
        try:
            # Save individual weight metadata
            self._save_weight_meta()
            
            # Save total weight metadata
            self._save_total_weight_meta()
            
            # Additionally save update timestamp and summary
            self._save_weight_update_summary()
            
        except Exception as e:
            logger.error(f"Failed to save updated weight metadata: {e}")
            raise

    def _save_weight_update_summary(self):
        """
        Save a summary of the weight update operation
        """
        import time
        
        summary = {
            "update_timestamp": time.time(),
            "update_time_readable": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()),
            "gpu_id": self.gpu_id,
            "total_weights": len(self.weight_infos),
            "total_devices": len(self.total_weight_dict),
            "device_weight_summary": self.total_weight_dict,
            "memory_usage_gb": sum(self.total_weight_dict.values()) if self.total_weight_dict else 0
        }
        
        os.makedirs("weights_metadata", exist_ok=True)
        summary_path = os.path.join("weights_metadata", f"weight_update_summary_{self.gpu_id}.json")
        
        try:
            with open(summary_path, 'w') as f:
                json.dump(summary, f, indent=2)
            logger.info(f"Saved weight update summary to {summary_path}")
        except IOError as e:
            logger.error(f"Failed to save weight update summary to {summary_path}: {e}")

    def _validate_weight_update(self):
        """
        Validate that weight update was successful by checking if weights have new pointers
        """
        if not self.weight_infos:
            logger.warning("No weight information found after update")
            return False
        
        # Check if we have the expected number of weights
        expected_weight_count = sum(1 for _ in self.model.named_parameters())
        actual_weight_count = len(self.weight_infos)
        
        if actual_weight_count != expected_weight_count:
            logger.warning(f"Weight count mismatch: expected {expected_weight_count}, got {actual_weight_count}")
            return False
        
        # Check if all weights are marked as CUDA tensors
        cuda_weights = sum(1 for info in self.weight_infos.values() if "cuda" in info["device"])
        if cuda_weights == 0:
            logger.warning("No CUDA weights found after update")
            return False
        
        logger.info(f"Weight update validation passed: {actual_weight_count} weights, {cuda_weights} on CUDA")
        return True
```

考虑到cuda层调用可能会出现资源冲突（比如框架这边在更新的时候那边刚好要读文件），添加了一个文件锁，用于避免更新阶段的读取：

```python
    def _acquire_weight_lock(self, timeout=10):
        """acquire weight metadata saving file lock"""
        os.makedirs("weights_metadata", exist_ok=True)
        lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")
        
        try:
            self._lock_fd = os.open(lock_file, os.O_CREAT | os.O_WRONLY)
            start_time = time.time()
            
            while True:
                try:
                    fcntl.flock(self._lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
                    logger.info(f"Acquired weight saving lock for GPU {self.gpu_id}")
                    return True
                except IOError:
                    if time.time() - start_time > timeout:
                        logger.error(f"Failed to acquire weight lock within {timeout} seconds")
                        os.close(self._lock_fd)
                        return False
                    time.sleep(0.1)
        except Exception as e:
            logger.error(f"Error acquiring weight lock: {e}")
            return False

    def _release_weight_lock(self):
        """release weight metadata saving file lock"""
        if hasattr(self, '_lock_fd'):
            try:
                fcntl.flock(self._lock_fd, fcntl.LOCK_UN)
                os.close(self._lock_fd)
                # delete lock file
                lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")
                if os.path.exists(lock_file):
                    os.remove(lock_file)
                logger.info(f"Released weight saving lock for GPU {self.gpu_id}")
            except Exception as e:
                logger.warning(f"Error releasing weight lock: {e}")
            finally:
                delattr(self, '_lock_fd')
```

### GPU内存使用情况检查

主要可以分两块：模型权重，以及cache管理，从Log信息来看最大的内存占用来自init\_memory\_pool里面的token\_to\_kv\_pool，在deepseek-coder-v2-lite例子中，占用了超过60GB。模型权重占用了3.7GB左右。（更新：这个3.7GB是因为开启了tp size = 8，在只使用单卡测试中可以看出完全的权重占比是29.5GB左右，还是比较可观的）。

8卡启动：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/jP2lRYjxZmYeEO8g/img/e5d9b15c-c2ab-4eae-bb96-48c230873180.png)

单卡启动：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/jP2lRYjxZmYeEO8g/img/8210069e-93d0-4e0b-8418-910a760379c2.png)

保存的权重模型信息包含设备指针以及size等信息，打印出来类似如下：

![image.png](https://alidocs.oss-cn-zhangjiakou.aliyuncs.com/res/jP2lRYjxZmYeEO8g/img/2e61a469-1158-4115-8654-a08e871dada7.png)

## Monkey Patch 动态补丁方案

注入的函数基本与上一节直接修改sglang框架代码中的保持一致，改成使用python monkey patch脚本包装的方式，不修改框架源码，运行时动态注入。

weight\_hook\_patch.py:

```python
import sys
import os

# wrapper_dir = os.path.dirname(os.path.abspath(__file__))
# python_source_dir = os.path.join(wrapper_dir, "python")
# sys.path.insert(0, python_source_dir)

import fcntl
# import runpy
import json
import time
import torch
from typing import List, Tuple, Union, Optional
# import sglang.srt.model_executor.model_runner as model_runner_module
from sglang.srt.server_args import ServerArgs, PortArgs

print(f"[SGLANG_PATCH] Patch Module loaded in process: {os.getpid()}")
# ===================================================================
# All patching code for model runner to handle weight metadata saving
# ===================================================================
def _patched_acquire_weight_lock(self, timeout=10):
    """acquire weight metadata saving file lock"""
    os.makedirs("weights_metadata", exist_ok=True)
    lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")

    try:
        self._lock_fd = os.open(lock_file, os.O_CREAT | os.O_WRONLY)
        start_time = time.time()

        while True:
            try:
                fcntl.flock(self._lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
                # logger.info(f"Acquired weight saving lock for GPU {self.gpu_id}")
                return True
            except IOError:
                if time.time() - start_time > timeout:
                    # logger.error(f"Failed to acquire weight lock within {timeout} seconds")
                    os.close(self._lock_fd)
                    return False
                time.sleep(0.1)
    except Exception as e:
        # logger.error(f"Error acquiring weight lock: {e}")
        return False

def _patched_release_weight_lock(self):
    """release weight metadata saving file lock"""
    if hasattr(self, '_lock_fd'):
        try:
            fcntl.flock(self._lock_fd, fcntl.LOCK_UN)
            os.close(self._lock_fd)
            # delete lock file
            lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")
            if os.path.exists(lock_file):
                os.remove(lock_file)
            # logger.info(f"Released weight saving lock for GPU {self.gpu_id}")
        # except Exception as e:
            # logger.warning(f"Error releasing weight lock: {e}")
        finally:
            delattr(self, '_lock_fd')

# Weights_hook function 
def _patched_register_weight_hooks(self):
    # self.weight_infos = {}  # Save weight metadatas
    self._clear_old_weight_data()

    def tensor_hook(tensor: torch.Tensor, name: str):
        if tensor.is_cuda:
            self.weight_infos[name] = {
                "ptr": tensor.data_ptr(),
                "size": tensor.numel() * tensor.element_size(),
                # "actual_size": tensor.storage().size() * tensor.element_size(),
                "device": str(tensor.device),
                "dtype": str(tensor.dtype),
                "shape": list(tensor.shape)
            }

    if not self._acquire_weight_lock():
        raise RuntimeError("Failed to acquire weight metadata update lock")

    # Register hooks to capture the initial state of model weights
    for name, param in self.model.named_parameters():
        tensor_hook(param, name)  # Capture parameter weights
    self._save_weight_meta()  # Save weight metadata to a local file
    self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")
    self._save_total_weight_meta()
    # self._merge_weights()  # Merge weights based on pointer continuity
    # self._save_merged_weight_meta()  # Save merged weight metadata to a local file
    self._release_weight_lock()

# Save the model weight metadata to a JSON file
def _patched_save_weight_meta(self):
    os.makedirs("weights_metadata", exist_ok=True)
    meta_path = os.path.join("weights_metadata", f"weights_meta_{self.gpu_id}.json")
    # meta_path = f"weights_meta_{self.gpu_id}.json"
    try:
        with open(meta_path, 'w') as f:
            json.dump(self.weight_infos, f, indent=2)
        # logger.info(f"Save weight metadata to {meta_path}.")
    except IOError as e:
        # logger.error(f"Failed to save weight metadata to {meta_path}: {e}")
        raise

def _patched_save_total_weight_meta(self):
    os.makedirs("weights_metadata", exist_ok=True)
    meta_path = os.path.join("weights_metadata", f"total_weight_meta_{self.gpu_id}.json")
    # meta_path = f"weights_meta_{self.gpu_id}.json"
    try:
        with open(meta_path, 'w') as f:
            json.dump(self.total_weight_dict, f, indent=2)
        # logger.info(f"Save total weight metadata to {meta_path}.")
    except IOError as e:
        # logger.error(f"Failed to save total weight metadata to {meta_path}: {e}")
        raise

def _patched_calculate_device_weight_sizes(self, unit: str = "bytes") -> dict:
    """Calculate the total size of weights per device in self.weight_infos.
    
    Args:
        unit (str): The unit to return the size in. 
                    Options: "bytes", "KB", "MB", "GB".
    
    Returns:
        dict: {device: total_size} where total_size is in the specified unit.
    """
    device_sizes = {}  # {device: total_size_in_bytes}

    # 遍历所有 weight_infos，按 device 累加 size
    for info in self.weight_infos.values():
        device = info["device"]
        size = info["size"]
        if device in device_sizes:
            device_sizes[device] += size
        else:
            device_sizes[device] = size

    # 单位转换
    unit = unit.upper()
    if unit == "KB":
        return {device: size / 1024 for device, size in device_sizes.items()}
    elif unit == "MB":
        return {device: size / (1024 ** 2) for device, size in device_sizes.items()}
    elif unit == "GB":
        return {device: size / (1024 ** 3) for device, size in device_sizes.items()}
    else:  # Default to bytes
        return device_sizes
    
# Functions for recording weight metadata during inference when update model weights
def _patched_handle_weight_update_hooks(self):
    """
    Handle weight updates during inference - clean old data and capture new weight information
    """
    # logger.info("Starting weight update hook processing...")
    if not self._acquire_weight_lock():
        raise RuntimeError("Failed to acquire weight metadata update lock")

    # Clear old weight information
    self._clear_old_weight_data()

    # Re-register hooks to capture updated weights
    self._register_updated_weight_hooks()

    # Save updated metadata
    self._save_updated_weight_metadata()

    self._release_weight_lock()

    # logger.info("Weight update hook processing completed.")

def _patched_clear_old_weight_data(self):
    """
    Clear old weight information and metadata files
    """
    # Clear in-memory data
    if hasattr(self, 'weight_infos'):
        self.weight_infos.clear()
    else:
        self.weight_infos = {}

    if hasattr(self, 'total_weight_dict'):
        self.total_weight_dict.clear()
    else:
        self.total_weight_dict = {}

    # Remove old metadata files
    try:
        weights_dir = "weights_metadata"
        if os.path.exists(weights_dir):
            old_weight_file = os.path.join(weights_dir, f"weights_meta_{self.gpu_id}.json")
            old_total_file = os.path.join(weights_dir, f"total_weight_meta_{self.gpu_id}.json")

            if os.path.exists(old_weight_file):
                os.remove(old_weight_file)
                # logger.info(f"Removed old weight metadata file: {old_weight_file}")

            if os.path.exists(old_total_file):
                os.remove(old_total_file)
                # logger.info(f"Removed old total weight metadata file: {old_total_file}")

    except Exception as e:
        # logger.warning(f"Failed to clean old metadata files: {e}")
        return

def _patched_register_updated_weight_hooks(self):
    """
    Register hooks for updated model weights (similar to _register_weight_hooks but for updates)
    """
    def tensor_hook(tensor: torch.Tensor, name: str):
        if tensor.is_cuda:
            self.weight_infos[name] = {
                "ptr": tensor.data_ptr(),
                "size": tensor.numel() * tensor.element_size(),
                "device": str(tensor.device),
                "dtype": str(tensor.dtype),
                "shape": list(tensor.shape),
                "updated": True  # Mark as updated weight
            }

    # Capture updated model weights
    # logger.info("Capturing updated model weights...")
    # weight_count = 0
    for name, param in self.model.named_parameters():
        tensor_hook(param, name)
        # weight_count += 1

    # logger.info(f"Captured {weight_count} updated weight tensors")

    # Calculate device weight sizes for updated weights
    self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")

def _patched_save_updated_weight_metadata(self):
    """
    Save updated weight metadata to JSON files
    """
    try:
        # Save individual weight metadata
        self._save_weight_meta()

        # Save total weight metadata
        self._save_total_weight_meta()

        # Additionally save update timestamp and summary
        self._save_weight_update_summary()

    except Exception as e:
        # logger.error(f"Failed to save updated weight metadata: {e}")
        return

def _patched_save_weight_update_summary(self):
    """
    Save a summary of the weight update operation
    """
    import time

    summary = {
        "update_timestamp": time.time(),
        "update_time_readable": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()),
        "gpu_id": self.gpu_id,
        "total_weights": len(self.weight_infos),
        "total_devices": len(self.total_weight_dict),
        "device_weight_summary": self.total_weight_dict,
        "memory_usage_gb": sum(self.total_weight_dict.values()) if self.total_weight_dict else 0
    }

    os.makedirs("weights_metadata", exist_ok=True)
    summary_path = os.path.join("weights_metadata", f"weight_update_summary_{self.gpu_id}.json")

    try:
        with open(summary_path, 'w') as f:
            json.dump(summary, f, indent=2)
        # logger.info(f"Saved weight update summary to {summary_path}")
    except IOError as e:
        # logger.error(f"Failed to save weight update summary to {summary_path}: {e}")
        return

def _patched_validate_weight_update(self):
    """
    Validate that weight update was successful by checking if weights have new pointers
    """
    if not self.weight_infos:
        # logger.warning("No weight information found after update")
        return False

    # Check if we have the expected number of weights
    expected_weight_count = sum(1 for _ in self.model.named_parameters())
    actual_weight_count = len(self.weight_infos)

    if actual_weight_count != expected_weight_count:
        # logger.warning(f"Weight count mismatch: expected {expected_weight_count}, got {actual_weight_count}")
        return False

    # Check if all weights are marked as CUDA tensors
    cuda_weights = sum(1 for info in self.weight_infos.values() if "cuda" in info["device"])
    if cuda_weights == 0:
        # logger.warning("No CUDA weights found after update")
        return False

    # logger.info(f"Weight update validation passed: {actual_weight_count} weights, {cuda_weights} on CUDA")
    return True

# Entry hook for updating weights metadata
def _patched_update_weights_metadata(self):
    """
    Public interface to update weight metadata
    """
    try:
        self._handle_weight_update_hooks()

        # Validate the update
        if self._validate_weight_update():
            # logger.info("Weight metadata update completed successfully")
            return True
        else:
            # logger.error("Weight metadata update validation failed")
            return False

    except Exception as e:
        # logger.error(f"Weight metadata update failed: {e}")
        return False
# ===================================================================
# print("[PATCH] All patches have been applied.")

# ===================================================================
# Monkey patch the ModelRunner class methods

def apply_model_runner_patches():
    print(f"[PATCH] Applying model runner patches in process {os.getpid()}...")
    try:
        from sglang.srt.model_executor.model_runner import ModelRunner

        ModelRunner._acquire_weight_lock = _patched_acquire_weight_lock
        ModelRunner._release_weight_lock = _patched_release_weight_lock
        ModelRunner._register_weight_hooks = _patched_register_weight_hooks
        ModelRunner._save_weight_meta = _patched_save_weight_meta
        ModelRunner._save_total_weight_meta = _patched_save_total_weight_meta
        ModelRunner._calculate_device_weight_sizes = _patched_calculate_device_weight_sizes
        ModelRunner._handle_weight_update_hooks = _patched_handle_weight_update_hooks
        ModelRunner._clear_old_weight_data = _patched_clear_old_weight_data
        ModelRunner._register_updated_weight_hooks = _patched_register_updated_weight_hooks
        ModelRunner._save_updated_weight_metadata = _patched_save_updated_weight_metadata
        ModelRunner._save_weight_update_summary = _patched_save_weight_update_summary
        ModelRunner._validate_weight_update = _patched_validate_weight_update
        ModelRunner.update_weights_metadata = _patched_update_weights_metadata

        if not hasattr(ModelRunner, '_original_load_model'):
            ModelRunner._original_load_model = ModelRunner.load_model
            def patched_load_model(self):
                print("[PATCH] Patching ModelRunner.load_model to handle weight metadata loading")
                self._original_load_model()
                # Register hooks after model is loaded
                self._register_weight_hooks()
            ModelRunner.load_model = patched_load_model
            

        if not hasattr(ModelRunner, '_original_update_weights_from_disk'):
            ModelRunner._original_update_weights_from_disk = ModelRunner.update_weights_from_disk
            def patched_update_weights_from_disk(
                    self, model_path: str, load_format: str
                ) -> tuple[bool, str]:
                print("[PATCH] Patching ModelRunner.update_weights_from_disk to handle update weight metadata loading")
                result = self._original_update_weights_from_disk(model_path, load_format)
                # Register hooks after weights are updated
                self.update_weights_metadata()
                return result
            ModelRunner.update_weights_from_disk = patched_update_weights_from_disk

        if not hasattr(ModelRunner, '_original_update_weights_from_tensor'):
            ModelRunner._original_update_weights_from_tensor = ModelRunner.update_weights_from_tensor
            def patched_update_weights_from_tensor(
                    self,
                    named_tensors: List[Tuple[str, Union[torch.Tensor, "LocalSerializedTensor"]]],
                    load_format: Optional[str] = None,
                ):
                print("[PATCH] Patching ModelRunner.update_weights_from_tensor to handle update weight metadata loading")
                result = self._original_update_weights_from_tensor(named_tensors, load_format)
                # Register hooks after weights are updated
                self.update_weights_metadata()
                return result
            ModelRunner.update_weights_from_tensor = patched_update_weights_from_tensor

    except Exception as e:
        print(f"[PATCH] Failed to apply ModelRunner patches: {e}")
        raise

# ====================================================================
# Patch the run_scheduler_process and run_data_parallel_controller_process functions (subprocesses)
def patched_run_scheduler_process(
        server_args: ServerArgs,
        port_args: PortArgs,
        gpu_id: int,
        tp_rank: int,
        pp_rank: int,
        dp_rank: Optional[int],
        pipe_writer,
    ):
    print(f"[PATCH] Patching run_scheduler_process for GPU {gpu_id}, TP rank {tp_rank}, PP rank {pp_rank}, DP rank {dp_rank} in process {os.getpid()} ...")
    apply_model_runner_patches()

    import sglang.srt.managers.scheduler as scheduler_module

    if not hasattr(scheduler_module, '_original_run_scheduler_process'):
            scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process

    assert hasattr(scheduler_module, '_original_run_scheduler_process')
    scheduler_module._original_run_scheduler_process(        
        server_args, port_args, gpu_id, tp_rank, pp_rank, dp_rank, pipe_writer
    )

def patched_run_data_parallel_controller_process(
        server_args: ServerArgs,
        port_args: PortArgs,
        pipe_writer,
    ):
    print(f"[PATCH] Patching run_data_parallel_controller_process in process {os.getpid()} ...")
    apply_model_runner_patches()

    import sglang.srt.managers.data_parallel_controller as dp_controller_module

    if not hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
            dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

    assert hasattr(dp_controller_module, '_original_run_data_parallel_controller_process')
    dp_controller_module._original_run_data_parallel_controller_process(server_args, port_args, pipe_writer)

# ===================================================================
def apply_entrypoint_patches():
    print(f"[PATCH] Applying entrypoint patches for SGLang server in {os.getpid()} ...")

    try:
        import sglang.srt.managers.scheduler as scheduler_module
        import sglang.srt.managers.data_parallel_controller as dp_controller_module

        if not hasattr(scheduler_module, '_original_run_scheduler_process'):
            scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process

        scheduler_module.run_scheduler_process = patched_run_scheduler_process

        if not hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
            dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

        dp_controller_module.run_data_parallel_controller_process = patched_run_data_parallel_controller_process

        # if hasattr(scheduler_module, '_original_run_scheduler_process') and hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
        #     print("[PATCH] run_scheduler_process and run_data_parallel_controller_process already patched, skipping.")
        #     print("[PATCH] run_scheduler_process already patched, skipping.")
        #     return
        
        # scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process
        # dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

        # Patch the functions
        # scheduler_module.run_scheduler_process = patched_run_scheduler_process
        # dp_controller_module.run_data_parallel_controller_process = patched_run_data_parallel_controller_process

    except Exception as e:
        print(f"[PATCH] Failed to import necessary modules for entrypoint patching: {e}")
        raise

```

launch\_wrapper.py:

```python
import sys
import os
import runpy

print(f"[WRAPPER] Starting main process with PID: {os.getpid()}.")

# --- 1. Set path ---
try:
    wrapper_dir = os.path.dirname(os.path.abspath(__file__))
    python_source_dir = os.path.join(wrapper_dir, "python")
    
    sys.path.insert(0, python_source_dir)
    sys.path.insert(0, wrapper_dir)
    print(f"[WRAPPER] sys.path configured. Added: {python_source_dir} and {wrapper_dir}")
except Exception as e:
    print(f"[WRAPPER] FATAL: Failed to configure sys.path: {e}", file=sys.stderr)
    sys.exit(1)

# --- 2. Entrypoint patch ---
try:
    import weight_hook_patch
    
    weight_hook_patch.apply_entrypoint_patches()

except Exception as e:
    print(f"[WRAPPER] FATAL: Failed to import or apply entrypoint patches: {e}", file=sys.stderr)
    sys.exit(1)

# --- 3. Launch server ---
if __name__ == "__main__":
    try:
        runpy.run_module("sglang.launch_server", run_name="__main__", alter_sys=True)
    except Exception as e:
        print(f"[WRAPPER] FATAL: An error occurred during SGLang server execution: {e}", file=sys.stderr)
        sys.exit(1)

```

*   **使用方式：**

在启动server时，用python launch\_wrapper.py来代替python -m sglang.launch\_server，比如：

```shell
# 原本的启动指令
python -m sglang.launch_server --model-path deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct --tp-size 8 --enable-expert-distribution-metrics --disable-cuda-graph --trust-remote-code --host 0.0.0.0 --port 30000

# 替换后的启动指令
python launch_wrapper.py --model-path deepseek-ai/DeepSeek-Coder-V2-Lite-Instruct --tp-size 8 --enable-expert-distribution-metrics --disable-cuda-graph --trust-remote-code --host 0.0.0.0 --port 30000
```

## [进度: 已完成] Python site-packages 自动加载方案

该方案涉及的补丁逻辑代码主要仍然是之前两个方案中的，重点是会将补丁代码文件和一个.pth文件一起放进python执行环境的site-packages目录，利用python执行时会优先扫描所有site-packages目录下的.pth文件并执行加载的特性，可以让用户无感地仍然使用原本的命令，而补丁自动动态加载。

新的文件组织方式：

sglang\_injector.pth:

```python
import sglang_patch_loader
```

~~sglang\_patch\_loader.py:~~（由于sglang子进程在加载时不能读取出有用的module信息，而sys.argv在执行时也无法读到sglang标识，所以目前暂时使用了一种fall back的方案，就是检查参数组合里必须有的-m和--model-path，来标识sglang启动命令。++2025.07.01 update：已弃用，换用module detect方式++。）

```python
# sglang_patch_loader.py (Optimized: Patch only in the main process)

import os
import sys

def is_main_sglang_process():
    """
    Detects if the current process is the main SGLang server process.
    This combines the most reliable checks we've found.
    """
    argv = sys.argv
    main_module = sys.modules.get('__main__')

    # --- Check 1: The most reliable method via runpy ---
    # `python -m sglang.launch_server` is executed by the `runpy` module.
    if main_module and hasattr(main_module, '__file__') and main_module.__file__:
        if 'runpy.py' in main_module.__file__ and 'sglang.launch_server' in argv:
            print("[SGLANG_PATCH_LOADER] >> Detected main process via runpy.")
            return True

    # --- Check 2: Fallback for your specific environment's argv structure ---
    # sys.argv is ['-m', '--model-path', ...]
    if len(argv) > 1 and argv[0] == '-m' and '--model-path' in argv:
        print("[SGLANG_PATCH_LOADER] >> Detected main process via special argv heuristic.")
        return True
        
    # --- Check 3: Standard argv check (less likely for you but good for general use) ---
    if len(argv) > 0 and 'sglang/launch_server.py' in argv[0]:
        print("[SGLANG_PATCH_LOADER] >> Detected main process via script path in argv.")
        return True

    return False

def run_patch():
    """
    Applies the patch ONLY if the current process is identified as the
    main SGLang server process. Child processes will inherit the patched state
    and will not re-apply the patch.
    """
    # This entire block of code will run in every process (main and children)
    # because of the .pth mechanism.
    
    # However, we only take action in the main process.
    if is_main_sglang_process():
        print(f"[SGLANG_PATCH_LOADER] >> Main SGLang process (PID: {os.getpid()}) confirmed. Applying patch now...")
        try:
            # Import the patch core only when needed
            from sglang_weight_hook_patch_core import apply_entrypoint_patches
            
            apply_entrypoint_patches()
            
            print(f"[SGLANG_PATCH_LOADER] >> Patch successfully applied once in the main process.")
            # We add a sentinel to prevent re-patching even in the same process, just in case.
            # This is a good defensive practice.
            setattr(sys, '_sglang_patch_applied', True)

        except Exception as e:
            print(f"[SGLANG_PATCH_LOADER] !! ERROR: Failed to apply patch in the main process (PID: {os.getpid()}): {e}")
    
    # For child processes or any other python process, this script will now do nothing and exit silently.
    # You can add a print here for debugging if you want to see it running in children.
    # else:
    #     print(f"[SGLANG_PATCH_LOADER] >> Skipping patch in process {os.getpid()} (not main sglang process).")


# --- Entry point called by the .pth file ---
# Add a global guard to ensure this runs only once per process, even if imported multiple times.
if not hasattr(sys, '_sglang_patch_applied'):
    run_patch()
```

sglang\_patch\_loader.py:（++2025.07.01 update++：新版，用module detect延迟加载补丁）

```python
# sglang_patch_loader.py
'''
Usage: 
Use install_sglang_hook_patch.sh to install the SGLang patch.
Use uninstall_sglang_hook_patch.sh to remove the patch.
Or manually:
Put sglang_patch_loader.py & sglang_weight_hook_patch_core.py & sglang_injector.pth into the python site-packages directory of the target environment.
Use this command to find the site-packages directory:
python -c "import site; print(site.getsitepackages()[0])"
'''

import sys
from importlib.abc import MetaPathFinder, Loader
from importlib.machinery import ModuleSpec

import sglang_weight_hook_patch_core

TARGET_MODULES = {
    "sglang.srt.model_executor.model_runner"
}

_patch_applied = False

class SGLangPatcherLoader(Loader):
    def __init__(self, original_loader):
        self.original_loader = original_loader

    def exec_module(self, module):
        self.original_loader.exec_module(module)

        global _patch_applied
        if not _patch_applied:
            print(f"[SGLANG_PATCH_LOADER] Target module '{module.__name__}' loaded. Applying patches...")
            try:
                sglang_weight_hook_patch_core.apply_model_runner_patches()
                _patch_applied = True
                print(f"[SGLANG_PATCH_LOADER] Patches applied successfully in process {sglang_weight_hook_patch_core.os.getpid()}.")
            except Exception as e:
                print(f"[SGLANG_PATCH_LOADER] Error applying patches: {e}", file=sys.stderr)


class SGLangPatcherFinder(MetaPathFinder):
    def find_spec(self, fullname, path, target=None):
        if fullname not in TARGET_MODULES or _patch_applied:
            return None

        original_finder = self
        for finder in sys.meta_path:
            if finder is self:
                continue
            spec = finder.find_spec(fullname, path, target)
            if spec:
                spec.loader = SGLangPatcherLoader(spec.loader)
                return spec
        return None

sys.meta_path.insert(0, SGLangPatcherFinder())

print(f"[SGLANG_PATCH_LOADER] SGLang patch loader initialized in process {sglang_weight_hook_patch_core.os.getpid()}. Waiting for target module import...")

```

sglang\_weight\_hook\_patch\_core.py:

```python
# sglang_weight_hook_patch_core.py
'''
Usage: 
Use install_sglang_hook_patch.sh to install the SGLang patch.
Use uninstall_sglang_hook_patch.sh to remove the patch.
Or manually:
Put sglang_patch_loader.py & sglang_weight_hook_patch_core.py & sglang_injector.pth into the python site-packages directory of the target environment.
Use this command to find the site-packages directory:
python -c "import site; print(site.getsitepackages()[0])"
'''

import sys
import os

# wrapper_dir = os.path.dirname(os.path.abspath(__file__))
# python_source_dir = os.path.join(wrapper_dir, "python")
# sys.path.insert(0, python_source_dir)

import fcntl
# import runpy
import json
import time
import torch
from typing import List, Tuple, Union, Optional
# import sglang.srt.model_executor.model_runner as model_runner_module
from sglang.srt.server_args import ServerArgs, PortArgs

print(f"[SGLANG_PATCH] Patch Module loaded in process: {os.getpid()}")
# ===================================================================
# All patching code for model runner to handle weight metadata saving
# ===================================================================
def _patched_acquire_weight_lock(self, timeout=10):
    """acquire weight metadata saving file lock"""
    os.makedirs("weights_metadata", exist_ok=True)
    lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")

    try:
        self._lock_fd = os.open(lock_file, os.O_CREAT | os.O_WRONLY)
        start_time = time.time()

        while True:
            try:
                fcntl.flock(self._lock_fd, fcntl.LOCK_EX | fcntl.LOCK_NB)
                # logger.info(f"Acquired weight saving lock for GPU {self.gpu_id}")
                return True
            except IOError:
                if time.time() - start_time > timeout:
                    # logger.error(f"Failed to acquire weight lock within {timeout} seconds")
                    os.close(self._lock_fd)
                    return False
                time.sleep(0.1)
    except Exception as e:
        # logger.error(f"Error acquiring weight lock: {e}")
        return False

def _patched_release_weight_lock(self):
    """release weight metadata saving file lock"""
    if hasattr(self, '_lock_fd'):
        try:
            fcntl.flock(self._lock_fd, fcntl.LOCK_UN)
            os.close(self._lock_fd)
            # delete lock file
            lock_file = os.path.join("weights_metadata", f"weight_saving_{self.gpu_id}.lock")
            if os.path.exists(lock_file):
                os.remove(lock_file)
            # logger.info(f"Released weight saving lock for GPU {self.gpu_id}")
        # except Exception as e:
            # logger.warning(f"Error releasing weight lock: {e}")
        finally:
            delattr(self, '_lock_fd')

# Weights_hook function 
def _patched_register_weight_hooks(self):
    # self.weight_infos = {}  # Save weight metadatas
    self._clear_old_weight_data()

    def tensor_hook(tensor: torch.Tensor, name: str):
        if tensor.is_cuda:
            self.weight_infos[name] = {
                "ptr": tensor.data_ptr(),
                "size": tensor.numel() * tensor.element_size(),
                # "actual_size": tensor.storage().size() * tensor.element_size(),
                "device": str(tensor.device),
                "dtype": str(tensor.dtype),
                "shape": list(tensor.shape)
            }

    if not self._acquire_weight_lock():
        raise RuntimeError("Failed to acquire weight metadata update lock")

    # Register hooks to capture the initial state of model weights
    for name, param in self.model.named_parameters():
        tensor_hook(param, name)  # Capture parameter weights
    self._save_weight_meta()  # Save weight metadata to a local file
    self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")
    self._save_total_weight_meta()
    # self._merge_weights()  # Merge weights based on pointer continuity
    # self._save_merged_weight_meta()  # Save merged weight metadata to a local file
    self._release_weight_lock()

# Save the model weight metadata to a JSON file
def _patched_save_weight_meta(self):
    os.makedirs("weights_metadata", exist_ok=True)
    meta_path = os.path.join("weights_metadata", f"weights_meta_{self.gpu_id}.json")
    # meta_path = f"weights_meta_{self.gpu_id}.json"
    try:
        with open(meta_path, 'w') as f:
            json.dump(self.weight_infos, f, indent=2)
        # logger.info(f"Save weight metadata to {meta_path}.")
    except IOError as e:
        # logger.error(f"Failed to save weight metadata to {meta_path}: {e}")
        raise

def _patched_save_total_weight_meta(self):
    os.makedirs("weights_metadata", exist_ok=True)
    meta_path = os.path.join("weights_metadata", f"total_weight_meta_{self.gpu_id}.json")
    # meta_path = f"weights_meta_{self.gpu_id}.json"
    try:
        with open(meta_path, 'w') as f:
            json.dump(self.total_weight_dict, f, indent=2)
        # logger.info(f"Save total weight metadata to {meta_path}.")
    except IOError as e:
        # logger.error(f"Failed to save total weight metadata to {meta_path}: {e}")
        raise

def _patched_calculate_device_weight_sizes(self, unit: str = "bytes") -> dict:
    """Calculate the total size of weights per device in self.weight_infos.
    
    Args:
        unit (str): The unit to return the size in. 
                    Options: "bytes", "KB", "MB", "GB".
    
    Returns:
        dict: {device: total_size} where total_size is in the specified unit.
    """
    device_sizes = {}  # {device: total_size_in_bytes}

    # 遍历所有 weight_infos，按 device 累加 size
    for info in self.weight_infos.values():
        device = info["device"]
        size = info["size"]
        if device in device_sizes:
            device_sizes[device] += size
        else:
            device_sizes[device] = size

    # 单位转换
    unit = unit.upper()
    if unit == "KB":
        return {device: size / 1024 for device, size in device_sizes.items()}
    elif unit == "MB":
        return {device: size / (1024 ** 2) for device, size in device_sizes.items()}
    elif unit == "GB":
        return {device: size / (1024 ** 3) for device, size in device_sizes.items()}
    else:  # Default to bytes
        return device_sizes
    
# Functions for recording weight metadata during inference when update model weights
def _patched_handle_weight_update_hooks(self):
    """
    Handle weight updates during inference - clean old data and capture new weight information
    """
    # logger.info("Starting weight update hook processing...")
    if not self._acquire_weight_lock():
        raise RuntimeError("Failed to acquire weight metadata update lock")

    # Clear old weight information
    self._clear_old_weight_data()

    # Re-register hooks to capture updated weights
    self._register_updated_weight_hooks()

    # Save updated metadata
    self._save_updated_weight_metadata()

    self._release_weight_lock()

    # logger.info("Weight update hook processing completed.")

def _patched_clear_old_weight_data(self):
    """
    Clear old weight information and metadata files
    """
    # Clear in-memory data
    if hasattr(self, 'weight_infos'):
        self.weight_infos.clear()
    else:
        self.weight_infos = {}

    if hasattr(self, 'total_weight_dict'):
        self.total_weight_dict.clear()
    else:
        self.total_weight_dict = {}

    # Remove old metadata files
    try:
        weights_dir = "weights_metadata"
        if os.path.exists(weights_dir):
            old_weight_file = os.path.join(weights_dir, f"weights_meta_{self.gpu_id}.json")
            old_total_file = os.path.join(weights_dir, f"total_weight_meta_{self.gpu_id}.json")

            if os.path.exists(old_weight_file):
                os.remove(old_weight_file)
                # logger.info(f"Removed old weight metadata file: {old_weight_file}")

            if os.path.exists(old_total_file):
                os.remove(old_total_file)
                # logger.info(f"Removed old total weight metadata file: {old_total_file}")

    except Exception as e:
        # logger.warning(f"Failed to clean old metadata files: {e}")
        return

def _patched_register_updated_weight_hooks(self):
    """
    Register hooks for updated model weights (similar to _register_weight_hooks but for updates)
    """
    def tensor_hook(tensor: torch.Tensor, name: str):
        if tensor.is_cuda:
            self.weight_infos[name] = {
                "ptr": tensor.data_ptr(),
                "size": tensor.numel() * tensor.element_size(),
                "device": str(tensor.device),
                "dtype": str(tensor.dtype),
                "shape": list(tensor.shape),
                "updated": True  # Mark as updated weight
            }

    # Capture updated model weights
    # logger.info("Capturing updated model weights...")
    # weight_count = 0
    for name, param in self.model.named_parameters():
        tensor_hook(param, name)
        # weight_count += 1

    # logger.info(f"Captured {weight_count} updated weight tensors")

    # Calculate device weight sizes for updated weights
    self.total_weight_dict = self._calculate_device_weight_sizes(unit="GB")

def _patched_save_updated_weight_metadata(self):
    """
    Save updated weight metadata to JSON files
    """
    try:
        # Save individual weight metadata
        self._save_weight_meta()

        # Save total weight metadata
        self._save_total_weight_meta()

        # Additionally save update timestamp and summary
        self._save_weight_update_summary()

    except Exception as e:
        # logger.error(f"Failed to save updated weight metadata: {e}")
        return

def _patched_save_weight_update_summary(self):
    """
    Save a summary of the weight update operation
    """
    import time

    summary = {
        "update_timestamp": time.time(),
        "update_time_readable": time.strftime("%Y-%m-%d %H:%M:%S", time.localtime()),
        "gpu_id": self.gpu_id,
        "total_weights": len(self.weight_infos),
        "total_devices": len(self.total_weight_dict),
        "device_weight_summary": self.total_weight_dict,
        "memory_usage_gb": sum(self.total_weight_dict.values()) if self.total_weight_dict else 0
    }

    os.makedirs("weights_metadata", exist_ok=True)
    summary_path = os.path.join("weights_metadata", f"weight_update_summary_{self.gpu_id}.json")

    try:
        with open(summary_path, 'w') as f:
            json.dump(summary, f, indent=2)
        # logger.info(f"Saved weight update summary to {summary_path}")
    except IOError as e:
        # logger.error(f"Failed to save weight update summary to {summary_path}: {e}")
        return

def _patched_validate_weight_update(self):
    """
    Validate that weight update was successful by checking if weights have new pointers
    """
    if not self.weight_infos:
        # logger.warning("No weight information found after update")
        return False

    # Check if we have the expected number of weights
    expected_weight_count = sum(1 for _ in self.model.named_parameters())
    actual_weight_count = len(self.weight_infos)

    if actual_weight_count != expected_weight_count:
        # logger.warning(f"Weight count mismatch: expected {expected_weight_count}, got {actual_weight_count}")
        return False

    # Check if all weights are marked as CUDA tensors
    cuda_weights = sum(1 for info in self.weight_infos.values() if "cuda" in info["device"])
    if cuda_weights == 0:
        # logger.warning("No CUDA weights found after update")
        return False

    # logger.info(f"Weight update validation passed: {actual_weight_count} weights, {cuda_weights} on CUDA")
    return True

# Entry hook for updating weights metadata
def _patched_update_weights_metadata(self):
    """
    Public interface to update weight metadata
    """
    try:
        self._handle_weight_update_hooks()

        # Validate the update
        if self._validate_weight_update():
            # logger.info("Weight metadata update completed successfully")
            return True
        else:
            # logger.error("Weight metadata update validation failed")
            return False

    except Exception as e:
        # logger.error(f"Weight metadata update failed: {e}")
        return False
# ===================================================================
# print("[PATCH] All patches have been applied.")

# ===================================================================
# Monkey patch the ModelRunner class methods

def apply_model_runner_patches():
    print(f"[PATCH] Applying model runner patches in process {os.getpid()}...")
    try:
        from sglang.srt.model_executor.model_runner import ModelRunner

        ModelRunner._acquire_weight_lock = _patched_acquire_weight_lock
        ModelRunner._release_weight_lock = _patched_release_weight_lock
        ModelRunner._register_weight_hooks = _patched_register_weight_hooks
        ModelRunner._save_weight_meta = _patched_save_weight_meta
        ModelRunner._save_total_weight_meta = _patched_save_total_weight_meta
        ModelRunner._calculate_device_weight_sizes = _patched_calculate_device_weight_sizes
        ModelRunner._handle_weight_update_hooks = _patched_handle_weight_update_hooks
        ModelRunner._clear_old_weight_data = _patched_clear_old_weight_data
        ModelRunner._register_updated_weight_hooks = _patched_register_updated_weight_hooks
        ModelRunner._save_updated_weight_metadata = _patched_save_updated_weight_metadata
        ModelRunner._save_weight_update_summary = _patched_save_weight_update_summary
        ModelRunner._validate_weight_update = _patched_validate_weight_update
        ModelRunner.update_weights_metadata = _patched_update_weights_metadata

        if not hasattr(ModelRunner, '_original_load_model'):
            ModelRunner._original_load_model = ModelRunner.load_model
            def patched_load_model(self):
                print("[PATCH] Patching ModelRunner.load_model to handle weight metadata loading")
                self._original_load_model()
                # Register hooks after model is loaded
                self._register_weight_hooks()
            ModelRunner.load_model = patched_load_model
            

        if not hasattr(ModelRunner, '_original_update_weights_from_disk'):
            ModelRunner._original_update_weights_from_disk = ModelRunner.update_weights_from_disk
            def patched_update_weights_from_disk(
                    self, model_path: str, load_format: str
                ) -> tuple[bool, str]:
                print("[PATCH] Patching ModelRunner.update_weights_from_disk to handle update weight metadata loading")
                result = self._original_update_weights_from_disk(model_path, load_format)
                # Register hooks after weights are updated
                self.update_weights_metadata()
                return result
            ModelRunner.update_weights_from_disk = patched_update_weights_from_disk

        if not hasattr(ModelRunner, '_original_update_weights_from_tensor'):
            ModelRunner._original_update_weights_from_tensor = ModelRunner.update_weights_from_tensor
            def patched_update_weights_from_tensor(
                    self,
                    named_tensors: List[Tuple[str, Union[torch.Tensor, "LocalSerializedTensor"]]],
                    load_format: Optional[str] = None,
                ):
                print("[PATCH] Patching ModelRunner.update_weights_from_tensor to handle update weight metadata loading")
                result = self._original_update_weights_from_tensor(named_tensors, load_format)
                # Register hooks after weights are updated
                self.update_weights_metadata()
                return result
            ModelRunner.update_weights_from_tensor = patched_update_weights_from_tensor

    except Exception as e:
        print(f"[PATCH] Failed to apply ModelRunner patches: {e}")
        raise

# ====================================================================
# Patch the run_scheduler_process and run_data_parallel_controller_process functions (subprocesses)
def patched_run_scheduler_process(
        server_args: ServerArgs,
        port_args: PortArgs,
        gpu_id: int,
        tp_rank: int,
        pp_rank: int,
        dp_rank: Optional[int],
        pipe_writer,
    ):
    print(f"[PATCH] Patching run_scheduler_process for GPU {gpu_id}, TP rank {tp_rank}, PP rank {pp_rank}, DP rank {dp_rank} in process {os.getpid()} ...")
    apply_model_runner_patches()

    import sglang.srt.managers.scheduler as scheduler_module

    if not hasattr(scheduler_module, '_original_run_scheduler_process'):
            scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process

    assert hasattr(scheduler_module, '_original_run_scheduler_process')
    scheduler_module._original_run_scheduler_process(        
        server_args, port_args, gpu_id, tp_rank, pp_rank, dp_rank, pipe_writer
    )

def patched_run_data_parallel_controller_process(
        server_args: ServerArgs,
        port_args: PortArgs,
        pipe_writer,
    ):
    print(f"[PATCH] Patching run_data_parallel_controller_process in process {os.getpid()} ...")
    apply_model_runner_patches()

    import sglang.srt.managers.data_parallel_controller as dp_controller_module

    if not hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
            dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

    assert hasattr(dp_controller_module, '_original_run_data_parallel_controller_process')
    dp_controller_module._original_run_data_parallel_controller_process(server_args, port_args, pipe_writer)

# ===================================================================
def apply_entrypoint_patches():
    print(f"[PATCH] Applying entrypoint patches for SGLang server in {os.getpid()} ...")

    try:
        import sglang.srt.managers.scheduler as scheduler_module
        import sglang.srt.managers.data_parallel_controller as dp_controller_module

        if not hasattr(scheduler_module, '_original_run_scheduler_process'):
            scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process

        scheduler_module.run_scheduler_process = patched_run_scheduler_process

        if not hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
            dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

        dp_controller_module.run_data_parallel_controller_process = patched_run_data_parallel_controller_process

        # if hasattr(scheduler_module, '_original_run_scheduler_process') and hasattr(dp_controller_module, '_original_run_data_parallel_controller_process'):
        #     print("[PATCH] run_scheduler_process and run_data_parallel_controller_process already patched, skipping.")
        #     print("[PATCH] run_scheduler_process already patched, skipping.")
        #     return
        
        # scheduler_module._original_run_scheduler_process = scheduler_module.run_scheduler_process
        # dp_controller_module._original_run_data_parallel_controller_process = dp_controller_module.run_data_parallel_controller_process

        # Patch the functions
        # scheduler_module.run_scheduler_process = patched_run_scheduler_process
        # dp_controller_module.run_data_parallel_controller_process = patched_run_data_parallel_controller_process

    except Exception as e:
        print(f"[PATCH] Failed to import necessary modules for entrypoint patching: {e}")
        raise

```

*   **使用方式：**

脚本方案：两个脚本，install\_sglang\_hook\_patch.sh 和 uninstall\_sglang\_hook\_patch.sh，直接执行./install\_sglang\_hook\_patch.sh即可安装，卸载同理。

手动方案：自己定位当前python环境的site-packages目录，并将上述三个代码文件拷贝入该目录即可。

后续启动sglang就用原生指令就行，补丁会自动加载。

## 附录

### 关于SGLang中的模型权重更新相关函数

从源码中可以发现，除了server初始化时会静态加载模型权重，还内置了几个update函数（同样在ModelRunner中是实际逻辑代码），分别为：

```python
......
    def update_weights_from_disk(
        self, model_path: str, load_format: str
    ) -> tuple[bool, str]:
        """Update engine weights in-place from the disk."""
    ......

    def init_weights_update_group(
        self,
        master_address,
        master_port,
        rank_offset,
        world_size,
        group_name,
        backend="nccl",
    ):
        """Initialize the Torch process group for model parameter updates.

        `_model_update_group` is used in the RLHF workflow, where rank
        0 is the actor model in the training engine, and the other ranks are
        the inference engine, which is used for rollout.

        In the RLHF workflow, the training engine updates the model
        weights/parameters online, and broadcasts them to the inference
        engine through the `_model_update_group` process group.
        """
      ......

    def update_weights_from_distributed(self, name, dtype, shape):
        """
        Update specific parameter in the model weights online
        through `_model_update_group` process group.

        Args:
            name: the name of the parameter to be updated.
            dtype: the data type of the parameter to be updated.
            shape: the shape of the parameter to be updated.
        """    
      ......

    def update_weights_from_tensor(
        self,
        named_tensors: List[Tuple[str, Union[torch.Tensor, "LocalSerializedTensor"]]],
        load_format: Optional[str] = None,
    ):
      ......
```

其中，init\_update\_group以及update\_weights\_from\_distributed两个函数是与RLHF（Reinforcement Learning from Human Feedback）相关，查询pr记录发现，这个是去年开发者的一个集成尝试，是尝试将SGLang集成进OpenRLHF：

[https://github.com/sgl-project/sglang/issues/2506: https://github.com/sgl-project/sglang/issues/2506](https://github.com/sgl-project/sglang/issues/2506)

由于RLHF用于后训练阶段，所以在推理阶段实际上不会用到这两个相关函数。（看OpenRLHF那边的反馈，似乎这个集成尝试也没有什么后文了）

对于另外两个更新函数update\_weights\_from\_disk以及update\_weights\_from\_tensor：

update\_weights\_from\_tensor是用于VerlEngine，同样是用于后训练阶段，不会在推理阶段使用。

update\_weights\_from\_disk在介绍里也是主要用于训练而非推理，但可以用于推理，但必须是完全相同的模型和权重数量名称等，相当于进行了一次模型热更新（并非完全的热更新，因为在更新过程中也会block request）。这个update方式也是唯一一个官方明确提供native api的：

[https://docs.sglang.ai/backend/native\_api.html#Update-Weights-From-Disk: https://docs.sglang.ai/backend/native\_api.html#Update-Weights-From-Disk](https://docs.sglang.ai/backend/native_api.html#Update-Weights-From-Disk)

### SGLang补丁安装与卸载脚本(site-packages方案使用)

install\_sglang\_hook\_patch.py:

```python
#!/bin/bash

# install_patch.sh: A script to install the SGLang non-intrusive patch.

# --- Configuration ---
# Names of the files to be installed.
PATCH_CORE_FILE="sglang_weight_hook_patch_core.py"
PATCH_LOADER_FILE="sglang_patch_loader.py"
PTH_FILE="sglang_injector.pth"

# --- Style Definitions ---
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

# Exit immediately if a command exits with a non-zero status.
set -e

echo -e "${YELLOW}Starting SGLang Patch Installation...${NC}"

# --- 1. Find the script's own directory to locate source files ---
SCRIPT_DIR=$(cd "$(dirname "${BASH_SOURCE[0]}")" &>/dev/null && pwd)
echo "Searching for source files in: ${SCRIPT_DIR}"

# --- 2. Check if source files exist ---
if [ ! -f "${SCRIPT_DIR}/${PATCH_CORE_FILE}" ] || \
   [ ! -f "${SCRIPT_DIR}/${PATCH_LOADER_FILE}" ] || \
   [ ! -f "${SCRIPT_DIR}/${PTH_FILE}" ]; then
    echo -e "${RED}Error: One or more source files not found in the script's directory.${NC}"
    echo "Please ensure ${PATCH_CORE_FILE}, ${PATCH_LOADER_FILE}, and ${PTH_FILE} are in the same directory as this script."
    exit 1
fi
echo -e "${GREEN}Source files found.${NC}"

# --- 3. Find the active Python environment's site-packages directory ---
echo "Detecting active Python environment..."
# Prefer 'python3' if available, otherwise fall back to 'python'
PYTHON_EXEC=$(command -v python3 || command -v python)

if [ -z "$PYTHON_EXEC" ]; then
    echo -e "${RED}Error: Could not find 'python' or 'python3' in your PATH.${NC}"
    echo "Please activate your target Python environment first."
    exit 1
fi
echo "Using Python executable: ${PYTHON_EXEC}"

# Get the first site-packages path. The python command is robust against empty results.
SITE_PACKAGES_DIR=$($PYTHON_EXEC -c "import site; print(site.getsitepackages()[0] if site.getsitepackages() else '')" | tail -n 1)

if [ -z "$SITE_PACKAGES_DIR" ] || [ ! -d "$SITE_PACKAGES_DIR" ]; then
    echo -e "${RED}Error: Could not determine a valid site-packages directory.${NC}"
    echo "Is your Python environment correctly configured?"
    exit 1
fi
echo "Target site-packages directory: ${SITE_PACKAGES_DIR}"

# --- 4. Check for write permissions ---
if [ ! -w "$SITE_PACKAGES_DIR" ]; then
    echo -e "${RED}Error: No write permission for ${SITE_PACKAGES_DIR}.${NC}"
    echo "Please run this script with sufficient permissions (e.g., using 'sudo ./install_patch.sh' if installing to a system-wide python)."
    exit 1
fi

# --- 5. Copy the files ---
echo "Copying patch files..."
cp -v "${SCRIPT_DIR}/${PATCH_CORE_FILE}" "${SITE_PACKAGES_DIR}/"
cp -v "${SCRIPT_DIR}/${PATCH_LOADER_FILE}" "${SITE_PACKAGES_DIR}/"
cp -v "${SCRIPT_DIR}/${PTH_FILE}" "${SITE_PACKAGES_DIR}/"

echo -e "\n${GREEN}SGLang patch installed successfully!${NC}"
echo "The patch is now active for the Python environment at ${PYTHON_EXEC}."
echo "You can now run 'python -m sglang.launch_server ...' directly."

```

uninstall\_sglang\_hook\_patch.py:

```python
#!/bin/bash

# uninstall_patch.sh: A script to uninstall the SGLang non-intrusive patch.

# --- Configuration ---
PATCH_CORE_FILE="sglang_weight_hook_patch_core.py"
PATCH_LOADER_FILE="sglang_patch_loader.py"
PTH_FILE="sglang_injector.pth"

# --- Style Definitions ---
GREEN='\033[0;32m'
RED='\033[0;31m'
YELLOW='\033[1;33m'
NC='\033[0m' # No Color

set -e

echo -e "${YELLOW}Starting SGLang Patch Uninstallation...${NC}"

# --- 1. Find the active Python environment's site-packages directory ---
echo "Detecting active Python environment..."
PYTHON_EXEC=$(command -v python3 || command -v python)

if [ -z "$PYTHON_EXEC" ]; then
    echo -e "${RED}Error: Could not find 'python' or 'python3' in your PATH.${NC}"
    exit 1
fi
echo "Using Python executable: ${PYTHON_EXEC}"

SITE_PACKAGES_DIR=$($PYTHON_EXEC -c "import site; print(site.getsitepackages()[0] if site.getsitepackages() else '')" | tail -n 1)

if [ -z "$SITE_PACKAGES_DIR" ] || [ ! -d "$SITE_PACKAGES_DIR" ]; then
    echo -e "${RED}Error: Could not determine a valid site-packages directory.${NC}"
    exit 1
fi
echo "Target site-packages directory: ${SITE_PACKAGES_DIR}"

# --- 2. Check for write permissions ---
if [ ! -w "$SITE_PACKAGES_DIR" ]; then
    echo -e "${RED}Error: No write permission for ${SITE_PACKAGES_DIR}.${NC}"
    echo "Please run this script with sufficient permissions (e.g., using 'sudo ./uninstall_patch.sh')."
    exit 1
fi

# --- 3. Remove the files ---
echo "Removing patch files..."
# Use "rm -f" to avoid errors if a file is already missing
rm -vf "${SITE_PACKAGES_DIR}/${PATCH_CORE_FILE}"
rm -vf "${SITE_PACKAGES_DIR}/${PATCH_LOADER_FILE}"
rm -vf "${SITE_PACKAGES_DIR}/${PTH_FILE}"

echo -e "\n${GREEN}SGLang patch uninstalled successfully!${NC}"

```