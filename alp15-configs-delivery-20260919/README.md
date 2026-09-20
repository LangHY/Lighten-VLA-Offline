# Alpamayo 1.5 离线配置交付物

此交付物只包含 Alpamayo 1.5 离线推理所需的两组非权重配置：

- `nvidia/Cosmos-Reason2-8B` @ `a9fae2cf89dc64db96b12860417f0eb403013bb9`
- `Qwen/Qwen3-VL-2B-Instruct` @ `89644892e4d85e24eaac8bacfd4f463576704203`

每组均严格包含九个文件：`config.json`、`generation_config.json`、`preprocessor_config.json`、`video_preprocessor_config.json`、`tokenizer.json`、`tokenizer_config.json`、`chat_template.json`、`merges.txt`、`vocab.json`。交付物不含任何模型权重。

## L20 解包后校验

在交付目录根部运行：

```bash
shasum -a 256 -c SHA256SUMS
```

若 L20 没有 `shasum`，可用：

```bash
sha256sum -c SHA256SUMS
```

## 离线兼容入口路径

```text
--cosmos-config-dir <交付目录>/dependencies/configs/Cosmos-Reason2-8B
--qwen-config-dir <交付目录>/dependencies/configs/Qwen3-VL-2B-Instruct
--config-manifest-dir <交付目录>/manifests
```

`cosmos-reason2-source.json` 记录 Hugging Face 权威 revision 及 ModelScope 传输来源；两组文件均已按大小、SHA-256 和冻结 revision 的 Git blob 身份验证。该资产验收不代表模型加载或端到端推理通过。
