# Alpamayo 1.5 Linux/amd64 uv 离线环境交付物

本目录用于在无外网的 Linux/amd64 L20 主机上，依据 NVIDIA 官方 Alpamayo 1.5 源码中的冻结 `uv.lock` 重建 Python 环境。它不是 macOS 环境，也不是可独立搬运后直接执行的通用 `.venv`。

## 内容

- `bin/uv`：Linux x86_64 的 uv 0.12.15。
- `uv-python/`：uv 管理的 CPython 3.12.14 Linux x86_64 运行时。
- `uv-cache/`：已填充的 Linux x86_64 依赖缓存，包含 torch 2.8.0+cu128 和本地构建的 flash-attn 2.8.3 wheel。
- `verify-venv/`：在禁网验证容器中创建的验证环境。其 Python 绝对链接指向 `/offline/uv-python/...`，只用于证明构建结果；不要把它直接当作 L20 原生环境。
- `manifests/`：冻结 `uv.lock`、`pyproject.toml` 和本次验证记录。
- `logs/`：断网同步和导入检查日志。

## L20 原生重建

先把本目录与冻结官方源码传到 L20。以下路径必须替换为 L20 上的实际绝对路径：

```bash
export ALP15_DEPS=/srv/alpamayo/alpamayo15-linux-amd64-uv-20260919
export ALP15_SOURCE=/srv/alpamayo/alpamayo1.5
export UV_CACHE_DIR="$ALP15_DEPS/uv-cache"
export UV_PYTHON_INSTALL_DIR="$ALP15_DEPS/uv-python"
export UV_PROJECT_ENVIRONMENT="$ALP15_SOURCE/.venv"

"$ALP15_DEPS/bin/uv" sync \
  --project "$ALP15_SOURCE" \
  --frozen \
  --offline
```

然后验证锁文件和关键导入：

```bash
shasum -a 256 "$ALP15_SOURCE/uv.lock"
"$ALP15_SOURCE/.venv/bin/python" - <<'PY'
import torch
import flash_attn
import physical_ai_av
import transformers
from alpamayo1_5.models.alpamayo1_5 import Alpamayo1_5

print("torch", torch.__version__, "CUDA build", torch.version.cuda)
print("CUDA available", torch.cuda.is_available())
print("flash_attn", flash_attn.__version__)
print("transformers", transformers.__version__)
print("Alpamayo", Alpamayo1_5.__name__)
PY
```

冻结 `uv.lock` 的期望 SHA256：

```text
6abf4ccba882e0b7849880ccaef794cbeecc36a43a2a12007e6b30632ae4e8b1
```

## 已完成的本机验证

在 `nvidia/cuda:12.8.1-devel-ubuntu22.04` 的 Linux/amd64 容器中，以 `--network none`、全新 `verify-venv/` 和本目录缓存执行 `uv sync --frozen --offline`，退出码为 0，共安装 107 个锁定包。随后在另一个禁网容器中完成 torch、flash-attn 和 Alpamayo 类导入；为裸 `ldd` 加入 venv 的 `torch/lib` 后，flash-attn 扩展无缺失动态库。

Mac Docker 不具备 NVIDIA GPU，因此这里没有验证 CUDA 实际执行。只有在 L20 上 `torch.cuda.is_available()` 为真、完成官方入口或批准的兼容入口并保存退出码、显存与输出后，才能宣称对应推理基线成功。

