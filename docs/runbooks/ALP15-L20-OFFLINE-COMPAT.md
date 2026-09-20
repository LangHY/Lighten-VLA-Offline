# Alpamayo 1.5 L20 离线兼容运行手册

本文把 [BF16 基准报告](ALP15-L20-BF16-BASELINE.pdf) 中因 PDF 排版被拆行的命令整理为可复制版本。适用服务器根目录为 `/home/tuqiang/lang/Lighten_VLA`。

## 1. 命名边界

当前已验证的是：

> Alpamayo 1.5 10B 在 NVIDIA L20 上的完全离线兼容性端到端推理与 BF16 性能基准。

不要写成“NVIDIA 官方原始入口完全离线跑通”。当前成功链路使用 `scripts/test_alpamayo15_offline_compat.py` 和自定义基准脚本，对本地模型路径、Cosmos/Qwen 配置及 `physical_ai_av` 离线行为做了兼容处理。

## 2. 固定资产

服务器项目目录应包含：

- `alpamayo1.5/`：官方源码，HEAD `36aeb4c5938cbc2eb2aed33b22434773da4ab639`；
- `models/`：五个 checkpoint 分片、索引、`config.json`、LICENSE 和 README；
- `alp15-test-clip-delivery-20260917/`：冻结测试 clip；
- `alp15-configs-delivery-20260919/`：Cosmos/Qwen 各九个非权重配置及来源清单；
- `alpamayo15-linux-amd64-uv-20260919/`：离线 uv、Python 与依赖缓存。

固定测试输入：

```text
clip_id=030c760c-ae38-49aa-9ad8-f5650a545d26
t0_us=5100000
chunk_id=3119
seed=42
top_p=0.98
temperature=0.6
num_traj_samples=1
max_generation_length=256
dtype=BF16
```

## 3. 首次离线重建环境

```bash
cd /home/tuqiang/lang/Lighten_VLA

export ALP15_DEPS="$PWD/alpamayo15-linux-amd64-uv-20260919"
export ALP15_SOURCE="$PWD/alpamayo1.5"
export UV_CACHE_DIR="$ALP15_DEPS/uv-cache"
export UV_PYTHON_INSTALL_DIR="$ALP15_DEPS/uv-python"
export UV_PROJECT_ENVIRONMENT="$ALP15_SOURCE/.venv"

sha256sum "$ALP15_SOURCE/uv.lock"
"$ALP15_DEPS/bin/uv" sync \
  --project "$ALP15_SOURCE" \
  --frozen \
  --offline
```

冻结 `uv.lock` 的期望 SHA-256：

```text
6abf4ccba882e0b7849880ccaef794cbeecc36a43a2a12007e6b30632ae4e8b1
```

## 4. 激活和验证环境

```bash
cd /home/tuqiang/lang/Lighten_VLA
source alpamayo1.5/.venv/bin/activate

python - <<'PY'
import torch
import flash_attn
import physical_ai_av
import transformers
from alpamayo1_5.models.alpamayo1_5 import Alpamayo1_5

print("torch:", torch.__version__)
print("torch CUDA build:", torch.version.cuda)
print("CUDA available:", torch.cuda.is_available())
if torch.cuda.is_available():
    print("GPU:", torch.cuda.get_device_name(0))
    print("BF16:", torch.cuda.is_bf16_supported())
print("flash_attn:", flash_attn.__version__)
print("physical_ai_av:", physical_ai_av.__name__)
print("transformers:", transformers.__version__)
print("Alpamayo:", Alpamayo1_5.__name__)
PY
```

已验证组合为 Python 3.12.14、PyTorch 2.8.0+cu128、FlashAttention 2.8.3、Transformers 4.57.1、`physical-ai-av==0.2.0` 和 NVIDIA L20。

## 5. 资产预检

```bash
cd /home/tuqiang/lang/Lighten_VLA

python -B scripts/test_alpamayo15_offline_compat.py \
  --model-dir "$PWD/models" \
  --dataset-dir "$PWD/alp15-test-clip-delivery-20260917" \
  --cosmos-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Cosmos-Reason2-8B" \
  --qwen-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Qwen3-VL-2B-Instruct" \
  --config-manifest-dir "$PWD/alp15-configs-delivery-20260919/manifests" \
  --preflight-only
```

只有 checkpoint、索引、clip、配置清单、冻结 revision、`physical-ai-av` 版本及外联阻断全部通过后，才进入正式推理。

## 6. 离线兼容端到端推理

```bash
cd /home/tuqiang/lang/Lighten_VLA
mkdir -p logs
set -o pipefail

python -B scripts/test_alpamayo15_offline_compat.py \
  --model-dir "$PWD/models" \
  --dataset-dir "$PWD/alp15-test-clip-delivery-20260917" \
  --cosmos-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Cosmos-Reason2-8B" \
  --qwen-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Qwen3-VL-2B-Instruct" \
  --config-manifest-dir "$PWD/alp15-configs-delivery-20260919/manifests" \
  2>&1 | tee "logs/offline_compat_e2e_$(date +%Y%m%d_%H%M%S).log"

exit_code=${PIPESTATUS[0]}
echo "exit_code=$exit_code"
test "$exit_code" -eq 0
```

预期关键输出：

```text
minADE = 0.371870219707489 m
pred_xyz shape = (1, 1, 1, 64, 3)
pred_rot shape = (1, 1, 1, 64, 3, 3)
```

## 7. BF16 性能基准

运行前确认 `scripts/benchmark_alpamayo15_offline.py` 存在，并核对其 SHA-256。2026-09-20 结果记录中的摘要为：

```text
861ff3d8a60c420681fd5eeb069a405d5159bf4dc4a6743fe84a3855937593b0
```

当前本地工作区尚未保存该脚本，不能在本机重算摘要；在脚本同步并核验前不要把下面命令作为本地可复现入口。

```bash
cd /home/tuqiang/lang/Lighten_VLA
mkdir -p logs
set -o pipefail

python -B scripts/benchmark_alpamayo15_offline.py \
  --source-dir "$PWD/alpamayo1.5" \
  --model-dir "$PWD/models" \
  --dataset-dir "$PWD/alp15-test-clip-delivery-20260917" \
  --cosmos-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Cosmos-Reason2-8B" \
  --qwen-config-dir "$PWD/alp15-configs-delivery-20260919/dependencies/configs/Qwen3-VL-2B-Instruct" \
  --config-manifest-dir "$PWD/alp15-configs-delivery-20260919/manifests" \
  --warmup-runs 1 \
  --runs 5 \
  2>&1 | tee "logs/alp15_l20_baseline_console_$(date +%Y%m%d_%H%M%S).log"

exit_code=${PIPESTATUS[0]}
echo "benchmark_exit_code=$exit_code"
test "$exit_code" -eq 0
```

结构化结果保存在 [`results/baselines/l20-bf16/2026-09-20`](../../results/baselines/l20-bf16/2026-09-20/README.md)。

## 8. 运行时资源观测

另开终端观察：

```bash
watch -n 1 nvidia-smi
```

如需保存采样：

```bash
mkdir -p logs
nvidia-smi \
  --query-gpu=timestamp,memory.used,memory.free,utilization.gpu,power.draw \
  --format=csv \
  -l 1 \
  > "logs/gpu_monitor_$(date +%Y%m%d_%H%M%S).csv"
```

## 9. 每次实验必须保留

- 实际命令和退出码；
- 完整控制台日志；
- GPU/驱动/CUDA/Python/依赖版本；
- 官方源码 HEAD、工作树状态、checkpoint 与数据标识；
- 输入参数、CoC、minADE 和输出形状；
- 模型加载、冷启动、稳态、VLM、Diffusion 延迟；
- allocated/reserved/device memory 和 GPU 监控记录；
- 结构化 JSON 结果及 SHA-256。

后续实验应固定 clip、`t0_us`、seed、采样参数、计时方法和 minADE 口径，确保剪枝、量化或 Token 剪枝结果可以与 BF16 基准直接比较。
