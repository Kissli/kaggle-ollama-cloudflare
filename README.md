# Kaggle Ollama + Cloudflare Tunnel

在 Kaggle 免费 GPU 上跑 Ollama 大模型，通过 Cloudflare Tunnel 对外提供 OpenAI 兼容接口，给 VS Code 的 Cline 插件用。

## 规格

- Kaggle Notebook，Accelerator 选 GPU T4 x2
- GPU：2 张 NVIDIA T4，每张 16GB，共 32GB（两张分开的卡，不是合成一张）
- CPU：4 核，内存 29GB
- 每周 GPU 额度 30 小时，单次最长 12 小时
- Python：Kaggle 预装，不用手动装 CUDA
- 模型：qwen3:8b（约 5.2GB）

## 运行顺序

在 Kaggle Notebook 里按 `kaggle-ollama.ipynb` 的单元格顺序跑：

1. 检查 GPU：`nvidia-smi`
2. 安装 Ollama，验证版本
3. 后台启动 Ollama，等 5 秒
4. 验证 Ollama 服务
5. 拉模型 `ollama pull qwen3:8b`，检查列表
6. 验证 OpenAI 兼容接口 `/v1/models`
7. 本地推理测试
8. 安装 cloudflared，验证版本
9. 启动隧道，保持该单元格运行，复制输出中的 `https://<随机>.trycloudflare.com` 地址
10. 公网测试，把 `<your-tunnel>` 换成自己的隧道子域名

## 注意

- Kaggle Notebook 要开联网，选 GPU
- 隧道地址每次重启都会变，`<your-tunnel>` 要换成自己的，不要编假地址
- 公网端点没有密码保护，不要公开分享
- Cline 配置见 `cline-config.json`，endpoint 换成自己的隧道地址
