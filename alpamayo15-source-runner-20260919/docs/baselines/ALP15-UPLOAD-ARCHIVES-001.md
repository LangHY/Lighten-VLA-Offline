# ALP15-UPLOAD-ARCHIVES-001：L20 上传归档

## 决策单

- 日期：2026-09-19；决策者、执行者与验收者：GPT-5.6 Sol high。
- 当前里程碑：跑通 NVIDIA 官方原始 Alpamayo 1.5 测试并建立可复现基线。
- 目标：在不重复归档已完成 Cosmos/Qwen 配置包的前提下，为剩余必传资产生成三份可校验归档：源码与运行代码、Alpamayo checkpoint、单 clip 数据；最后生成总上传 SHA-256 清单。
- 输入与边界：官方源码 HEAD `36aeb4c5938cbc2eb2aed33b22434773da4ab639`；模型仅允许 `LICENSE`、`README.md`、`config.json`、`model.safetensors.index.json` 和五个冻结分片；clip 必须保留完整 cache snapshot、内部 blobs、九个相对符号链接和 `dataset-manifest.json`。
- 明确排除：源码中的不可搬运 `alpamayo1.5/.venv`；项目根 `.git`、`.venv`、`.DS_Store`、`.ruff_cache`；模型 `.gitattributes` 与 `configuration.json`；下载缓存和日志；已单独归档的 Cosmos/Qwen 配置目录。
- 产物：每份 `.tar.zst` 及相邻 `.sha256`；总清单包含三份新归档、现有配置归档和现有 Linux/amd64 uv 环境归档。
- 验收标准：归档目标事前不存在；`zstd -t`、相邻 SHA-256 和 `zstd -dc | tar -tf -` 全部通过；源码归档含官方 `.git` 而不含 `.venv`；模型归档成员集合恰好为九项；clip 归档保留九个符号链接；项目测试通过；原始资产不删除。
- 停止条件：可用空间不足 29 GiB、任一输入集合或哈希异常、归档覆盖已有文件、归档流失败或成员集合偏离。执行采用直接 tar→zstd 流，不创建大型中间副本。

## 执行回报

- 模型归档首次在写入前停止：历史决策记录中的 `config.json` SHA-256 只有 63 位，属于无效摘要；未创建模型归档。随后以冻结 Hugging Face revision 的 Git blob 和 ModelScope 固定传输 revision 的 `X-Linked-Etag` 双重核对，本机 3058-byte 文件的正确 SHA-256 为 `824fc3552466aaecb67c896a453667e1e15c5687adbb416ce42a6bda3de1e68e`。相关脚本与文档已修正后再继续。
