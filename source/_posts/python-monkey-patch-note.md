---
title: Python环境下自动注入动态补丁方案
date: 2025-07-03 10:32:38
tags:
    - Python
    - SGLang
categories:
    - Tech Notes
cover: "imgs/tech_note/monkey-patch-python.webp"
excerpt: "使用 Monkey Patch 以及模块检测的方法，实现对 Python 项目的无感知代码注入。"
comment: false
---

## site-packages

> A Python installation has a `site-packages` directory inside the module directory. This directory is where user installed packages are dropped. A `.pth` file in this directory is maintained which contains paths to the directories where the extra packages are installed.

一般来说，用户创建的python虚拟环境就对应一个site-packages，这个目录存放该环境下安装的包（包含pip install的以及用户手动安装的等），里面会存放很多packages等，也会有一些.pth文件，这些.pth文件会在所有python进程启动的时候优先检查，一般作用是优先导入所需的包，比如：

```python
import _virtualenv
```

同样在同目录下有一个\_virtualenv.py文件，存放实际需要导入的代码。

`.pth` 文件生效的机制是：Python解释器在**启动时**，会扫描`site-packages`目录，找到`.pth`文件并执行里面的代码。（注意对该环境下每个进程都会生效）

++可以利用这个机制，在site-packages目录下放入我们自己实现的.pth文件和.py文件，来达到自动注入的效果++。

用下面的命令可以查找当前环境的site-packages目录位置：

```python
python -c "import site; print(site.getsitepackages()[0])"
```

## Python monkey patch 动态补丁

> **Monkey patching** in Python refers to the practice of dynamically modifying or extending code at runtime typically replacing or adding new functionalities to existing modules, classes or methods without altering their original source code.

简单说就是，对于某些项目，想利用里面的hook或者加入自己的逻辑但又不想直接修改源码，可以用monkey patching这种动态补丁方式，来向项目里注入自己的逻辑（只在执行时生效）。

基础使用可以参考这篇文章：

[https://www.tutorialspoint.com/python/python\_monkey\_patching.htm](https://www.tutorialspoint.com/python/python_monkey_patching.htm)

具体使用时，一些详细的使用细节以及实现方式可以直接问ai。

## Import-based lazy patching

在注入的时候，为了避免对全局环境造成影响，可以对patch进行一下拆分，单独实现一个loader，在loader里添加module检测，只有当指定的特定module在被当前进程导入的时候，才去实际加载补丁（延迟加载）。

以向sglang里打补丁为例，可以实现这样一个loader：

```python
# sglang_patch_loader.py

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

注意，这里的target\_module可以不是我们需要打补丁的module，它只是作为一个“哨兵”存在，检测到它被导入的时候去实际加载补丁，所以可以根据实际情况选择监控哪个module。补丁本身可以是向我们所需的任何位置打。

## 总结

基本上就是利用上面几个机制，来给针对的项目自动注入逻辑补丁，实现自己的功能。

我习惯的文件组织方式是实现为：

*   xxx\_patch\_injector.pth：给site-packages用的入口文件，用于import loader

*   xxx\_patch\_loader.py：加载控制器，里面通过module detect控制补丁的注入逻辑

*   xxx\_patch\_core.py：实际补丁，里面实现我们想要注入的逻辑，可以不止一个文件

只要确保injector放在当前使用的python环境的site-packages目录下，而loader和core可以被injector加载即可。（保险的话可以将几个文件都放在site-packages目录下）
