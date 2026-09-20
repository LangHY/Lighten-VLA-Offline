# 智行·端析 / Alpamayo 1.5：当前进度

更新时间：2026-09-19 22:15 CST。本文是**状态快照**；进行中的下载或构建须以完成后的退出码和验收记录更新，不能按“已完成”理解。

## 当前结论

官方源码、10B checkpoint 文件、一个冻结测试 clip、已封装的 Cosmos/Qwen 非权重配置、独立离线兼容推理脚本及经禁网重建验证的 Linux/amd64 uv 依赖交付物已在本机准备；**尚未具备可确认的 L20 完全离线端到端推理结果**。最关键的未完成项是资产传输后复验与 L20 实机运行。没有进行剪枝、蒸馏、量化、ONNX 或 TensorRT 工作。

| 项目 | 当前状态 | 已有证据／边界 |
| --- | --- | --- |
| NVIDIA 官方源码 | 已核对 | `alpamayo1.5/` HEAD=`36aeb4c5938cbc2eb2aed33b22434773da4ab639`，工作树 clean；原始 `test_inference.py` SHA256=`dc69646feed09f92defa00a19ae6f2fc2a37a10a70ec35678f87946fbe6fe8e8`，未改动。 |
| Alpamayo 1.5 10B 权重 | 文件已准备 | 本机 `models/` 含五个 safetensors 分片、`config.json` 和索引；此前已按冻结大小／SHA256 校验，本次复查五片与两份 JSON 仍在。**尚未在 L20 加载**，传输后需重验。 |
| 一个测试 clip | 文件已验收 | `/Users/lang/Downloads/alp15-test-clip-delivery-20260917`：5 项 feature、4 项 metadata，冻结 `clip_id=030c760c-ae38-49aa-9ad8-f5650a545d26`、`t0_us=5100000`、`chunk_id=3119`；文件验收见 [ALP15-CLIP-ACCEPT-001.md](ALP15-CLIP-ACCEPT-001.md)。本机已在外联拦截下通过 `physical_ai_av==0.2.0` 读取 egomotion，**尚未完成四路视频与模型端到端测试**。 |
| 离线兼容推理代码 | 已实现、本机测试通过 | [脚本](../../scripts/test_alpamayo15_offline_compat.py)显式使用本地权重、clip、Cosmos/Qwen 配置，模型配置仅在内存中重映射，并阻止 Python socket 外联；项目 7 项本机测试通过。它不等于 NVIDIA 官方原始入口跑通，详见 [ALP15-OFFLINE-INFER-001.md](ALP15-OFFLINE-INFER-001.md)。 |
| Qwen 处理器配置 | 九项已验收并封装 | 冻结 revision `89644892e4d85e24eaac8bacfd4f463576704203` 的九个非权重文件与 HF blob 身份已校验，并以真实文件写入配置交付物；全局缓存保留。 |
| Cosmos 处理器／VLM 配置 | 九项已验收并封装 | ModelScope 传输的九个非权重文件已按大小与 Git blob SHA-1 匹配冻结 Hugging Face revision `a9fae2cf89dc64db96b12860417f0eb403013bb9`，且未发现权重；原 staging 已移入废纸篓。配置交付目录、归档与 SHA-256 见 [ALP15-CONFIG-DELIVERY-001.md](ALP15-CONFIG-DELIVERY-001.md)。**不需要 Cosmos 权重**。 |
| Linux/amd64 uv 环境 | 本机依赖交付物已验收 | `/Users/lang/Downloads/alpamayo15-linux-amd64-uv-20260919` 包含 uv 0.12.15、CPython 3.12.14、uv cache 和验证 venv；另有约 3.76 GiB 的 `.tar.zst` 上传归档及 SHA256。在 CUDA 12.8.1 / Ubuntu 22.04 / amd64 的全新禁网容器中执行 `uv sync --frozen --offline` 退出码 0，安装 107 包；torch、flash-attn、Alpamayo 导入和修正运行时库路径后的 `ldd` 均通过。Mac 无 NVIDIA GPU，L20 CUDA 执行仍未验收。详见 [ALP15-UV-ENV-001.md](ALP15-UV-ENV-001.md)。 |
| L20 官方／兼容推理 | 未执行 | L20 驱动、CUDA、依赖导入、官方入口阻塞记录、完整输出及资源观测均尚无实机验收证据。 |

## Linux/amd64 环境准备结果

原有 `ubuntu:22.04` 实际是 `linux/arm64`，没有用于目标 L20 依赖。另一个本地 `alpamayo:latest`（Ubuntu 22.04 / amd64 / CUDA 12.4.1）缺少 `nvcc`，其中 PyTorch 导入报 `libcusparseLt.so.0` 缺失，也未用作最终基线。

最终使用 NVIDIA 官方 `nvidia/cuda:12.8.1-devel-ubuntu22.04` 的 Linux/amd64 镜像，digest=`sha256:a99a1860ba8e2916e5c3e73b72ec4c4301653a84586e05bfc9a2aa2d58027e97`。在 `--network none` 新容器中，以导出的 uv cache 和 uv 管理 Python 完成全新环境同步；后续禁网导入验证退出码 0。首次裸 `ldd` 因未加入 PyTorch 的 `torch/lib` 以退出码 3 停止；保留失败日志后按实际运行时库路径复验，结果为 `ldd_missing=none`。验证 venv 的 Python 是指向 `/offline/uv-python` 的绝对链接，因此仅作为证据；L20 应从交付缓存重新创建环境。

## 下一步与验收门槛

1. 向 L20 传输源码、模型、clip、`/Users/lang/Downloads/alp15-configs-delivery-20260919.tar.zst` 及 `/Users/lang/Downloads/alpamayo15-linux-amd64-uv-20260919`；在 L20 先校验归档与全文件 SHA-256，再依据交付 README 断网重建 venv，并验证架构、驱动/CUDA、Python、核心导入与 `torch.cuda.is_available()`。
2. 按 [AGENTS.md](../../AGENTS.md) 先记录 NVIDIA 官方原始入口在断网环境下的实际结果；若确有已记录的阻塞，再运行已批准的离线兼容入口。保存完整日志、退出码、输出形状、CoC、minADE、内存／显存与耗时，由 Sol 复核。**兼容入口成功不能命名为官方原始入口成功**。

本文件不包含 token、密码、SSH 地址、GPU UUID 或带签名下载 URL；完整过程输出以当前任务工具调用记录及后续单独的构建日志为准。
