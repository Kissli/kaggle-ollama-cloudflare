# Kaggle Ollama + Cloudflare Tunnel

在 Kaggle 免费 GPU（双 T4，共 32GB 显存）上部署 Ollama 大模型服务，通过 Cloudflare Quick Tunnel 对外提供 OpenAI 兼容接口，供本地 VS Code 的 Cline / Roo Code 等 AI 编程插件零成本调用。

针对 Kaggle 4 核 CPU 与双 Tesla T4 GPU 环境进行了完整的**硬件级加速优化**（CUDA 驱动库防回退、4 线程 OpenMP/MKL 绑定、Flash Attention 2、显存永久常驻与预热），支持 **GitHub Actions 网页端一键启动/关机**、**Kaggle API 命令行自动化** 以及 **远程急停关机（Remote Kill Switch）**，杜绝浪费每周宝贵的 30 小时 GPU 额度。

---

## 硬件规格与优化策略

- **环境**：Kaggle Notebook，Accelerator 选 `GPU T4 x2`
- **GPU 资源**：2 张 NVIDIA Tesla T4，每张 16GB，共 32GB 显存（双卡张量并行）
- **CPU 资源**：4 核 vCPU，29GB 内存
- **GPU 配额**：每周 30 小时免费额度，单次会话最长 12 小时（默认保活 4 小时）
- **默认模型**：`qwen3:8b`（约 5.2GB，预热后显存毫秒级响应）
- **上下文窗口**：已实测调优至 **32,768 (32k)**，兼顾多轮代码长上下文与显存安全（彻底解决 Ollama 默认 2048 词导致的提示词静默截断问题）

### 硬件与性能优化亮点
1. **CUDA 驱动库显式链接**：注入 `/usr/lib64-nvidia` 等动态链接库路径，防止 Ollama 启动时因驱动检测失败而**静默回退至 CPU 纯软算（1~2 tok/s）**。
2. **CPU 4 线程深度绑定**：显式注入 `OMP_NUM_THREADS=4`、`MKL_NUM_THREADS=4`，优化 Prompt Ingestion 阶段的 CPU 多核计算与内存数据吞吐。
3. **Flash Attention 2 加速**：开启 Turing 架构显存优化，大幅降低长上下文显存占用。
4. **显存预热（VRAM Pre-Warm）**：启动后通过本地 API 立即预热锁定显存，后续 VS Code 发起首个请求零延迟，避免插件超时。
5. **显存永久驻留 (`OLLAMA_KEEP_ALIVE=-1`)**：解决 Ollama 默认 5 分钟闲置即卸载权重的问题。

---

## 前置要求

1. **Kaggle 账号手机验证**：Kaggle 规定必须完成手机号绑定验证，才会在 Notebook 右侧设置栏解锁 `Internet`（联网权限）与 `GPU T4 x2` 加速卡。
2. **VS Code 插件**：安装 [Cline](https://marketplace.visualstudio.com/items?itemName=saoudrizwan.claude-dev) 或 [Roo Code](https://marketplace.visualstudio.com/items?itemName=RooVeterinaryInc.roo-cline)。

---

## 多种部署运行方式对比

| 部署方式 | 算力提供方 | 特点 | 适用场景 |
| :--- | :--- | :--- | :--- |
| **方式一：GitHub Actions 一键调度（推荐）** | Kaggle 双 T4 GPU | 本地零依赖，直接在 GitHub 网页点按钮启动与急停关机 | 最省心、全平台通用 |
| **方式二：本地终端 Kaggle CLI** | Kaggle 双 T4 GPU | 本地一行命令提交与远程急停关机，无浏览器依赖 | 熟悉命令行与自动化脚本的开发者 |
| **方式三：Kaggle 网页端直接运行** | Kaggle 双 T4 GPU | 传统 Jupyter Notebook 逐行交互执行 | 首次体验或排查调试 |
| **方式四：Google Colab 备用运行** | Colab 单张 T4 GPU | 免费单卡 15GB 显存，无需配置 API Token | Kaggle 每周 30h 额度耗尽时的临时替补 |

> **⚠️ 关于“直接在 GitHub 上跑大模型”的说明**：
> GitHub Actions 官方免费提供的 Runner 只有 **2 核 CPU 与 7GB 内存，没有独立 GPU**。如果在 GitHub Runner 上直接跑 8B 大模型，推理速度仅为 0.5~1 词/秒（极慢且不可用），且 GitHub 服务条款（ToS）严格禁止将 Actions 长期用作对外代理/隧道（易封号）。
> **最佳实践**：把 GitHub Actions 当作**云端总控台**，利用它自动调用 Kaggle API，将运算安全地运行在 Kaggle 免费的 32GB 双 T4 GPU 上！

---

### 方式一：GitHub Actions 网页端一键调度（推荐）

只需在 GitHub 网页上点一下按钮即可启动，无需在本地电脑配置 Python 环境。

#### 1. 配置 GitHub 仓库 Secrets
在你的 GitHub 仓库页面，进入 **Settings** -> **Secrets and variables** -> **Actions**，点击 **New repository secret**，添加以下两个密钥：
- `KAGGLE_USERNAME`：kiki5858
- `KAGGLE_KEY`：export KAGGLE_API_TOKEN=KGAT_310d05ce27500534eb5918dbd1b3b593
- `NTFY_TOPIC`（可选）：ntfy_topicmy-secret-ollama-channel

#### 2. 一键启动云端 GPU
1. 点击仓库上方的 **Actions** 标签页。
2. 在左侧工作流列表中选择 **Deploy Ollama on Kaggle**。
3. 点击右侧的 **Run workflow** 下拉按钮：
   - **选择操作**：选择 `start`
   - **自定义 ntfy 频道**（可选）：填入你习惯的频道名称（如留空则自动生成随机频道）
4. 点击绿色的 **Run workflow** 开始运行。GitHub 会在 10 秒内自动完成环境校验并将代码提交至 Kaggle 双 T4 GPU 节点启动！

#### 3. 接收公网隧道 URL
打开手机 ntfy App 或电脑浏览器访问 `https://ntfy.sh/<你的频道名>`，云端服务就绪后会收到推送消息，包含可直接使用的 `https://<随机>.trycloudflare.com/v1` 地址。

#### 4. 一键关机保存配额
当你写完代码时，再次回到 GitHub **Actions** -> **Deploy Ollama on Kaggle** -> **Run workflow**，操作选择 `stop` 并点击运行，GitHub 会立刻向云端发送急停关机信号，**秒级停止扣除 Kaggle 每周 GPU 配额**！

---

### 方式二：本地终端 Kaggle CLI 命令行一键部署

适合本地习惯使用命令行的开发者。

#### 1. 配置本地 Kaggle CLI
```bash
pip install kaggle
mkdir -p ~/.kaggle
cp /path/to/kaggle.json ~/.kaggle/
chmod 600 ~/.kaggle/kaggle.json
```

#### 2. 修改元数据用户名
打开项目根目录下的 `kernel-metadata.json`，将 `id` 中的 `YOUR_KAGGLE_USERNAME` 替换为你的 Kaggle 用户名：
```json
{
  "id": "你的Kaggle用户名/kaggle-ollama-cloudflare",
  "title": "Kaggle Ollama Cloudflare",
  "code_file": "kaggle-ollama.ipynb",
  "language": "python",
  "kernel_type": "notebook",
  "is_private": "true",
  "enable_gpu": "true",
  "enable_internet": "true",
  "machine_shape": "NvidiaTeslaT4"
}
```

#### 3. 推送启动与关机
```bash
# 启动云端双 T4 GPU 服务
kaggle kernels push

# 本地工作完成后，远程一键急停停机（保存配额）
curl -d "SHUTDOWN_GPU" https://ntfy.sh/<你的-NTFY_TOPIC>
```

---

### 方式三：Kaggle 网页端交互式运行

1. 打开 [Kaggle](https://www.kaggle.com/) -> 创建新 Notebook（或点击 File -> Import Notebook 上传 `kaggle-ollama.ipynb`）。
2. **关键侧边栏设置**：
   - **Accelerator**：选择 `GPU T4 x2`
   - **Internet**：切换为 `Internet On`（开启联网）
3. 按顺序从上往下运行 Notebook 单元格：
   - **Cell 0**：硬件检查（检查 2 块 T4 GPU 与 4 核心 CPU）
   - **Cell 1**：环境清理与 Ollama 安装
   - **Cell 2**：注入 4 核 CPU 与双 GPU 优化环境变量，启动 Ollama 并进行健康探测
   - **Cell 3**：验证 Ollama 本地接口
   - **Cell 4**：下载 `qwen3:8b` 模型
   - **Cell 5**：预热并锁定显存（VRAM Pre-Warm），执行本地推理冒烟测试
   - **Cell 6**：安装 Cloudflare Tunnel 客户端（`cloudflared`）
   - **Cell 7**：启动隧道，捕获生成的 `https://<子域名>.trycloudflare.com` 地址并推送通知
   - **Cell 8**：公网端点与 SSE 流式推理（Server-Sent Events）连通性验证，自动输出 Cline 配置 JSON
   - **Cell 9**：4 小时保活心跳循环（自带故障 3 次崩溃退出防护，并监听 `SHUTDOWN_GPU` 远程急停信号）
   - **Cell 10**：会话清理单元格（释放进程与临时文件）

---

### 方式四：Google Colab 备用运行方式

当你的 Kaggle 每周 30 小时配额耗尽时，可将 `kaggle-ollama.ipynb` 导入 [Google Colab](https://colab.research.google.com/) 运行：
1. 打开 Colab，点击“文件” -> “上传笔记本”，上传 `kaggle-ollama.ipynb`。
2. 菜单栏选择“代码执行程序” -> “更改运行时类型” -> 硬件加速器选择 **T4 GPU**。
3. 从上往下执行各单元格即可（Colab 免费提供单张 T4 GPU 15GB 显存，对于 `qwen3:8b` 同样可以全量装入显存运行）。

---

## VS Code Cline 插件配置

当获取到公网地址后，打开 VS Code 进行配置：

1. 打开 Cline 设置（点击 Cline 侧边栏右上角的齿轮 ⚙️）。
2. 在 **API Provider** 下拉框选择 **OpenAI Compatible**。
3. 按如下参数填写（或参考 `cline-config.json`）：
   - **Base URL**：`https://<你的子域名>.trycloudflare.com/v1`
   - **API Key**：`none`（任意字符均可）
   - **Model ID**：`qwen3:8b`
4. 展开下方 **Model Configuration**：
   - **Context Window Size**：`32768`
   - **Max Tokens**：`32768`
   - **Max Output Tokens**：`8192`

也可以直接参考根目录下的配置文件模板 `cline-config.json`：
```json
{
  "apiKey": "none",
  "endpoint": "https://<YOUR-TUNNEL-SUBDOMAIN>.trycloudflare.com/v1",
  "model": {
    "contextWindow": 32768,
    "id": "qwen3:8b",
    "maxOutputTokens": 8192,
    "maxTokens": 32768,
    "name": "qwen3:8b"
  },
  "name": "ollama",
  "vendor": "ollama"
}
```

---

## 常见问题排查 (FAQ)

1. **为什么过去推理速度极慢（只有 1~2 tok/s）？**
   - 这是因为之前的脚本缺少 `/usr/lib64-nvidia` 等库路径，Ollama 在容器中未能正确握手 GPU 驱动，静默回退到了 CPU 软算。新版本在启动前注入了完整的驱动动态链接库路径和 `CUDA_VISIBLE_DEVICES=0,1`，推理速度正常为 40~60 tok/s。
2. **为什么 Cline 发送稍大代码块后会胡言乱语？**
   - Ollama 原生默认的上下文仅为 2048 词（约 1500 字），旧配置为 Cline 声明了 128k，导致 Ollama 自动从头部截断了 Cline 核心的 System Prompt 与工具调用规则。新版本在启动时与预热时统一固定为 32,768 上下文，彻底解决截断问题。
3. **如何防止配额耗尽？**
   - 如果使用 GitHub Actions 启动，直接在 Actions 页面选择 `stop` 运行即可一键关机。
   - 如果在本地，在终端执行 `curl -d "SHUTDOWN_GPU" https://ntfy.sh/<topic>` 即可远程一键停机。
   - 即使不手动停机，Notebook 内建了异常探测机制，若隧道或服务连续 3 次探测失败会自动退出，防止静默空烧配额。
