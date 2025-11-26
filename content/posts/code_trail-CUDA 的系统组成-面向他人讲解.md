---
title: "CUDA 的系统组成-面向他人讲解"
date: 2025-11-01
tags:
  - 
publish: true  
---

## 很棒的资料
1. https://jamesakl.com/posts/cuda-ontology/ 详细系统讲解了CUDA整体

## 从顶往下：GPU 软件栈的层次结构

可以把 CUDA 系统理解成**从硬件到应用的四层结构**：

```bash
┌──────────────────────────┐
│ 你的深度学习程序 (PyTorch / TensorFlow) │ ← 应用层
├──────────────────────────┤
│ CUDA Toolkit / cuDNN / cuBLAS / 驱动API │ ← 开发与加速库层
├──────────────────────────┤
│ NVIDIA 驱动 (Driver) │ ← 驱动层
├──────────────────────────┤
│ GPU 硬件 (2080 Ti 等) │ ← 硬件层
└──────────────────────────┘
```

**“我的 Python 程序通过 PyTorch 调用 CUDA Runtime API，**
**CUDA Runtime 依赖 NVIDIA 驱动与硬件通信，**
**驱动再控制显卡完成底层计算。”**

**“CUDA 生态是一整套 GPU 加速栈：**
**从最底层的硬件到驱动，再到 Toolkit 和加速库，最后被深度学习框架调用。**
**驱动连接硬件，Toolkit 提供编译和库接口，**
**框架调用这些接口完成并行计算。”**

| **层级**     | **名称**                                 | **作用**                                     | **举例或说明**                             |
| ------------ | ---------------------------------------- | -------------------------------------------- | ------------------------------------------ |
| **硬件层**   | GPU 芯片                                 | 负责并行计算（核心算力来源）                 | RTX 2080 Ti, RTX 3070, A100                |
| **驱动层**   | NVIDIA Driver                            | 操作系统和 GPU 之间的桥梁，负责调度 GPU 资源 | 比如 Driver Version: 535.230.02            |
| **运行库层** | CUDA Runtime / cuDNN / cuBLAS / cuFFT 等 | NVIDIA 提供的各种 GPU 加速库                 | PyTorch 调用 cuDNN 做卷积、cuBLAS 做矩阵乘 |
| **工具包层** | CUDA Toolkit                             | 包含编译器 nvcc、头文件、开发工具            | CUDA 11.3, 12.1 等                         |
| **应用层**   | 深度学习框架 / 程序                      | 你的 Python 代码、PyTorch、TensorFlow 等     | torch.cuda.is_available()                  |

1. 驱动必须 ≥ CUDA Toolkit 要求的最低版本。驱动新版可以兼容老 CUDA。

2. 当用 conda install pytorch cudatoolkit=11.3 时，**这套 CUDA runtime 是内置在 PyTorch 包里的**，不会依赖系统路径

   - 所以哪怕系统没装 /usr/local/cuda，PyTorch 依然能跑。
   - torch.version.cuda 看到的版本即 PyTorch 编译时自带的 CUDA 版本。

3. 排查命令

4. | **检查命令**                                        | **作用**                            |
   | --------------------------------------------------- | ----------------------------------- |
   | nvidia-smi                                          | 驱动+GPU检测                        |
   | nvcc -V                                             | CUDA Toolkit 版本（如果系统安装了） |
   | python -c "import torch; print(torch.version.cuda)" | PyTorch 自带 CUDA 版本              |
   | torch.cuda.is_available()                           | 是否能用 GPU                        |

