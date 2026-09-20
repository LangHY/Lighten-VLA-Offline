# 基线执行记录

当前整体状态见 [ALP15-PROGRESS-20260917.md](ALP15-PROGRESS-20260917.md)；其中严格区分已验证资产、进行中的环境准备和未完成的 L20 实机门禁。

## L20 候选基线服务器只读盘点

将仓库中的脚本复制到已获授权的 L20 候选基线服务器后运行。通过 Sol 决策后，该服务器即为官方测试执行服务器。Mac 仅用于代码和记录准备，不运行官方模型测试。最短命令如下；不设置目录变量时，只收集主机环境信息。

```bash
bash audit_l20_readiness.sh
```

如已知且允许披露对应路径，可在同一命令中限定盘点范围：

```bash
CODE_DIR=/srv/alpamayo1.5 MODEL_DIR=/srv/models/Alpamayo-1.5-10B DATA_DIR=/srv/alpamayo-data bash audit_l20_readiness.sh /tmp/alp15_l20_audit
```

脚本只读取服务器状态，并只在传入的日志目录（或当前目录自动生成的时间戳目录）写入 `readiness.log`。它不会联网、安装软件、下载文件、加载模型、改动系统配置或读取目录之外的模型/数据文件。

## 回传要求

回传以下内容，供 Sol 决策是否允许进入官方原始测试阶段：

- `readiness.log` 完整文件，以及实际运行命令、退出码与生成时间；
- 使用的服务器类型（L20）、操作系统、驱动、CUDA、Python、PyTorch、可见 GPU 与 BF16 查询结果；
- 若设置了目录变量：源码 Git remote/HEAD/status、模型文件总大小与五个指定官方 safetensors 的校验结果、数据目录清单；
- 任一缺失命令、错误或资源不足现象的完整原始输出。

绝对不要发送 token、密码、私钥、SSH 配置、GPU UUID、主机 IP，或包含这些信息的原始配置文件。若 Git remote URL 中含认证信息，先人工脱敏后再回传。

## 离线资产准备（L20 无外网）

离线包的来源、版本、执行顺序、Linux/amd64 依赖边界及验收口径见 [ALP15-OFFLINE-ASSET-001.md](ALP15-OFFLINE-ASSET-001.md)。准备脚本默认只做 dry-run；只有显式 `--execute` 才会下载或写入资产：

```bash
bash scripts/prepare_alpamayo15_offline_bundle.sh --bundle-root /transfer/alpamayo15-offline-bundle
```

准备机完成实际下载后，必须在不运行模型的前提下做只读验收：

```bash
bash scripts/verify_alpamayo15_offline_bundle.sh --bundle-root /transfer/alpamayo15-offline-bundle
```

当前默认使用已冻结的 Hugging Face 模型与数据集来源；模型 ModelScope 传输源的受限规则见 [ALP15-OFFLINE-ASSET-001.md](ALP15-OFFLINE-ASSET-001.md)。不要向任何脚本参数传入 token。

已拉取冻结官方仓库后，独立离线兼容入口的差异、资产缺口与 L20 预检/运行口径见 [ALP15-OFFLINE-INFER-001.md](ALP15-OFFLINE-INFER-001.md)。它不修改或替代 NVIDIA 官方原始入口。

Cosmos 九个非权重配置的 ModelScope 传输、冻结 Hugging Face blob 等价性校验与来源清单见 [ALP15-COSMOS-CONFIG-001.md](ALP15-COSMOS-CONFIG-001.md)。

Cosmos/Qwen 干净配置目录、上传归档、校验与中间文件清理记录见 [ALP15-CONFIG-DELIVERY-001.md](ALP15-CONFIG-DELIVERY-001.md)。
