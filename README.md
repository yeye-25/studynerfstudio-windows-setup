<img width="1265" height="1401" alt="image" src="https://github.com/user-attachments/assets/2221967f-6cc5-4c4c-adde-39912dfc7d7b" />
# Day 1: Windows 下 Nerfstudio + 3D Gaussian Splatting 底层环境搭建实录

## 📌 项目背景
在 Windows 环境下配置 `nerfstudio` 及 `splatfacto` (3D Gaussian Splatting) 训练环境，主要解决由于新旧版本不兼容导致的 C++ 编译崩溃问题。

## 💻 环境信息
*   操作系统：Windows 10/11
*   显卡：RTX 4060 Laptop GPU (8GB)
*   CUDA Toolkit：12.1
*   PyTorch：2.5.1+cu121
*   Python：3.9 (Miniconda3)

## 🛠️ 核心踩坑与解决方案

### 1. 彻底清理旧版 CUDA 11.8
*   **操作**：除了控制面板卸载，必须手动删除物理残留目录 `C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.8`。
*   **注意**：坚决保留显卡驱动、NVIDIA 控制面板，防止黑屏。

### 2. 编译 `tiny-cuda-nn` 报 C1128 错误
*   **报错**：`fatal error C1128: 节数超过对象文件格式限制: 请使用 /bigobj 进行编译`
*   **解决**：设置环境变量，让 nvcc 传递参数给编译器：
    ```cmd
    set NVCC_PREPEND_FLAGS=-Xcompiler /bigobj
    pip install . --no-build-isolation
3. 编译 gsplat 遇到地狱级报错 (cudafe++ 崩溃)
报错：fatal error C1189: -- unsupported Microsoft Visual Studio version! 及 nvcc error: 'cudafe++' died with status 0xC0000409

原因：安装了过新的 VS（如 VS 2026 / MSVC 14.52），超出 CUDA 12.1 的支持上限。强行加 -allow-unsupported-compiler 也无效。

解决：

彻底卸载最新的 VS。
下载安装 Visual Studio 2022 生成工具 (Build Tools)。
关键：在安装器的“单个组件”中，勾选旧版编译器 MSVC v143 - VS 2022 C++ x64/x86 生成工具 (v14.39-17.9)。
将 14.39 版本的 cl.exe 路径加入系统环境变量 Path 的最顶端。

4. pip 构建隔离导致找不到 torch
解决：编译 CUDA 扩展时必须加上 --no-build-isolation。
pip install gsplat==1.4.0 --no-build-isolation --no-cache-dir
🚀 Day 1 成果
成功跑通官方自带的 apple 数据集，并在 Viewer 中看到了清晰的 3D 模型，成功导出 .ply 文件。

***

### 📄 今天总结（11月15日）：数据处理、COLMAP 排雷与自建数据集
# Day 2: COLMAP 依赖配置与自建数据集排坑实录

## 📌 项目背景
在成功跑通官方测试集后，尝试使用自己拍摄的照片（`test` 文件夹）训练 3D 模型，重点解决数据处理工具链（FFmpeg、COLMAP）的兼容性问题。

## 🛠️ 核心踩坑与解决方案

### 1. 缺失 FFmpeg
*   **报错**：`Could not find ffmpeg. Please install ffmpeg.`
*   **解决**：用 winget 下载时常因网络超时失败，用 Conda 安装常遇到 GBK 编码报错。**最稳方案**：直接去官网下载 `ffmpeg-release-essentials.zip`，解压后将 `bin` 文件夹路径加到系统环境变量 `Path` 中。

### 2. COLMAP 版本地狱（flann vs faiss 冲突）
*   **报错**：`Failed to read faiss index...` 或 `unrecognised option '--SiftExtraction.use_gpu'`
*   **原因**：通过 Conda 安装的 COLMAP 3.13 太新。2025年5月后，COLMAP 底层从 `flann` 切换到 `faiss`，且修改了参数名，导致 Nerfstudio 1.1.5 无法兼容。
*   **解决**：
    1. **代码级绕过**：修改 Nerfstudio 源码 `colmap_utils.py` 中的参数名。
    2. **穷举匹配**：在命令后加 `--matching-method exhaustive` 绕过词汇树匹配。
    3. **硬核方案**：手动敲 COLMAP 命令行，逐行提取特征、匹配、建立稀疏重建。

### 3. 照片重叠度低导致重建失败
*   **现象**：使用 62 张照片，COLMAP 疯狂报 `Discarding reconstruction due to bad initial pair`，最终只找出了 21 张照片的相机位姿（匹配率仅 33.87%）。
*   **原因**：拍摄时走动太快，相邻照片没有共同特征（重叠度太低），且部分照片可能曝光过度或模糊。
*   **解决**：先用成功算出的 21 张图片模型，通过 `--skip-colmap` 手动导入 Nerfstudio 跑通流程。
    ns-process-data images --data "原图路径" --output-dir "输出路径" --skip-colmap --colmap-model-path "COLMAP生成的稀疏模型路径"
💡 拍摄心得（极其重要）
慢走、密拍：保证相邻两张照片有 70% 以上的画面重叠。

光线充足、不反光：避免物体表面反光导致特征提取失败。

稳：千万别手抖拍糊，模糊的照片是 COLMAP 的天敌。

