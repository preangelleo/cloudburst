# CloudBurst - 按需云端视频生成框架

我的第二个开源项目！🚀

[English Version](./README.md)

> ⚠️ **重要更新**：我们现在推荐新部署使用 **[CloudBurst Fargate](https://github.com/preangelleo/cloudburst-fargate)**。CloudBurst Fargate 提供：
> - 🚀 无服务器架构（无需管理服务器）
> - 💰 按秒计费（更加经济高效）
> - ⚡ 自动伸缩能力
> - 🔧 零维护开销
>
> 这个基于 EC2 的版本为需要特定实例控制或有现有 EC2 工作流的用户保留。

## 这是什么？

一个 **自动管理 AWS EC2 实例** 的 Python 框架，专门用于按需视频生成任务。

**核心价值**：当你的应用需要生成视频时（使用我们的 [视频生成 API](https://github.com/preangelleo/video-generation-docker)），这个框架会：
- 🚀 启动一个带有视频生成 Docker 容器的 EC2 实例
- 🎬 并行处理你的视频生成请求
- 📥 下载完成的视频
- 🛑 立即终止实例以节省成本
- 💰 你只需为实际处理时间付费（按分钟计费，而不是按月！）

**适用场景**：需要在工作流程中生成视频，但不想维护 24/7 GPU 实例的生产应用。

## 🔗 基于视频生成 API 构建

本项目基于我的第一个开源项目：[**视频生成 API**](https://github.com/preangelleo/video-generation-docker)

### 完美组合：
- **[视频生成 API](https://github.com/preangelleo/video-generation-docker)**：实际生成视频的核心 Docker 镜像
- **CloudBurst（本项目）**：自动化 AWS 部署，按需运行 API

### 两种部署选择：
| 选项 | 适用场景 | 成本模式 | 设置方式 |
|------|---------|---------|----------|
| **视频生成 API** | 频繁使用，常驻服务 | 按月付费（~$500） | 24/7 运行 Docker 容器 |
| **CloudBurst** | 偶尔使用，批量处理 | 按次付费（~$0.20/批） | 自动创建/销毁实例 |

CloudBurst 自动拉取并部署视频生成 API Docker 镜像，让您以 96% 的成本节省获得同样强大的视频生成能力！

## 🎯 解决的问题

24/7 运行云实例太贵了。这个框架：
- ✅ 仅在需要时创建实例
- ✅ 运行你的任务（如视频生成）
- ✅ 下载结果
- ✅ 自动终止实例
- ✅ 显示准确成本（例如：15个视频仅需 $0.018）

**按分钟付费，而不是按月！**

## 📋 前提条件

开始之前，请确保您已准备好：

### 1. AWS 账户及必需权限
- **AWS Access Key ID** 和 **AWS Secret Access Key**
- IAM 权限包括：
  - EC2（创建、终止、查询实例）
  - VPC（安全组、子网）
  - EC2 Pricing API 访问权限

### 2. AWS 资源（必须先在 AWS 控制台创建）
- **EC2 密钥对** - 用于 SSH 访问实例（在 EC2 → 密钥对 中创建）
- **安全组** - 开放 5000 端口供 API 访问（在 EC2 → 安全组 中创建）
- **VPC 子网 ID** - 实例将在此子网启动（在 VPC → 子网 中查找）

### 3. 本地环境要求
- Python 3.7+
- pip（Python 包管理器）
- 互联网连接（用于 AWS API 调用）

### 4. AWS 凭证设置

本框架使用 boto3，它会自动按以下顺序查找 AWS 凭证：

1. **环境变量**（推荐用于生产环境）
```bash
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=us-east-1
```

2. **AWS CLI 配置**（如果您已安装 AWS CLI）
```bash
aws configure
```

3. **AWS 凭证文件**（由 AWS CLI 自动创建）
```
~/.aws/credentials
```

**注意**：您无需手动创建任何文件。只需设置环境变量或使用 AWS CLI 即可。

## ⚡ 重要：下载行为说明

**重要**：理解 `auto_terminate` 参数对成功下载文件至关重要：

| 设置 | 行为 | 使用场景 |
|------|------|----------|
| `auto_terminate=True` | 处理 → 自动下载 → 终止 | 生产批处理、CI/CD 流水线 |
| `auto_terminate=False`（默认） | 处理 → 保持运行 | 开发、调试、手动控制 |

**关键点**：
- ✅ 使用 `auto_terminate=True`：文件在终止前自动下载到 `output/` 目录
- ✅ 使用 `auto_terminate=False`：你必须手动调用 `download_batch_results()` 后再终止
- ✅ 使用自动下载时，下载的文件路径在 `result['downloaded_files']` 中
- ⚠️ 默认值为 `False`，防止意外终止而丢失文件

### 输出目录配置

所有处理函数现在都支持 `saving_dir` 参数，使用相同的三级优先级：
1. **用户提供的 `saving_dir`**（最高优先级）
2. **环境变量/.env 中的 `RESULTS_DIR`**
3. **默认值：`./cloudburst_results/`**

**单批次处理（`execute_batch`）**：
```python
result = operation.execute_batch(
    scenes=scenes,
    saving_dir="/path/to/my/videos"  # 可选：指定保存位置
)
# 文件保存到：{saving_dir}/batch_YYYYMMDD_HHMMSS/
```

**并行处理（`execute_parallel_batches`）**：
```python
result = execute_parallel_batches(
    scenes=scenes,
    saving_dir="/path/to/my/videos"  # 可选：指定保存位置
)
# 文件保存到：{saving_dir}/batch_N_timestamp/
```

**文件夹扫描（`scan_and_test_folder`）**：
```python
result = scan_and_test_folder(
    folder_path="./scenes",
    saving_dir="/path/to/my/videos"  # 可选：指定保存位置
)
```

## 🚀 快速开始

### 1. 安装

```bash
git clone https://github.com/preangelleo/cloudburst
cd cloudburst
pip install -r requirements.txt
```

### 2. 配置

创建 `.env` 文件：
```env
# AWS 设置
AWS_REGION=us-east-1
AWS_KEY_PAIR_NAME=your-keypair-name
AWS_SECURITY_GROUP_ID=sg-xxxxxxxxx
AWS_SUBNET_ID=subnet-xxxxxxxxx

# AWS 凭证（必需）
AWS_ACCESS_KEY_ID=your-aws-access-key-id
AWS_SECRET_ACCESS_KEY=your-aws-secret-access-key

# 可选：指定 AMI 或实例类型
AWS_INSTANCE_TYPE=c5.2xlarge
DOCKER_IMAGE=betashow/video-generation-api:latest
```

### 3. 使用示例

#### 方法 1：自动文件夹扫描

**所需文件夹结构**：
```
video_scenes/
├── images/
│   ├── scene_001_chinese.png
│   └── scene_002_chinese.png
├── audio/
│   ├── scene_001_chinese.mp3
│   ├── scene_002_chinese.mp3
│   ├── scene_001_chinese.srt (可选)
│   └── scene_002_chinese.srt (可选)
```

**注意**：文件必须遵循命名模式 `scene_XXX_chinese.*` 或 `scene_XXX_english.*`

```python
from instant_instance_operation_v2 import scan_and_test_folder

# 注意：scan_and_test_folder 默认使用 auto_terminate=True！
result = scan_and_test_folder(
    folder_path="./video_scenes/",
    language="chinese",
    enable_zoom=True
)

print(f"✅ 生成了 {result['successful_scenes']} 个视频")
print(f"💰 总成本：${result['cost_usd']:.4f}")

# 文件已自动下载！
if result.get('downloaded_files'):
    print(f"📥 视频保存到：{result['output_directory']}")
    for file_path in result['downloaded_files']:
        print(f"   🎬 {file_path}")
```

### 🎬 输出效果示例

以下是生成视频的实际效果：

[![视频生成示例](https://img.youtube.com/vi/eJtuwBV_7qY/maxresdefault.jpg)](https://www.youtube.com/watch?v=eJtuwBV_7qY)

**视频特性展示**：
- ✅ 专业中文字幕（带背景框）
- ✅ 流畅的缩放特效（Ken Burns 效果）
- ✅ 音画同步
- ✅ 高质量视频输出（1080p）

此示例展示了"完整功能"模式，同时启用了字幕和特效。

#### 方法 2：自定义场景列表（更灵活）

```python
from instant_instance_operation_v2 import InstantInstanceOperationV2

# 初始化
operation = InstantInstanceOperationV2()

# 定义场景（可使用任意文件路径）
scenes = [
    {
        "scene_name": "intro_video",
        "input_image": "/path/to/intro.png",
        "input_audio": "/path/to/intro.mp3",
        "subtitle": "/path/to/intro.srt"  # 可选，可为 None
    },
    {
        "scene_name": "main_content",
        "input_image": "/path/to/main.png", 
        "input_audio": "/path/to/main.mp3",
        "subtitle": None  # 此场景无字幕
    }
]

# 处理场景
result = operation.execute_batch(
    scenes=scenes,
    language="chinese",
    enable_zoom=True,
    auto_terminate=False  # 保持运行，需手动下载
)

print(f"✅ 处理了 {result['successful_scenes']} 个视频")
print(f"💰 处理成本：${result['cost_usd']:.4f}")

# 当 auto_terminate=False 时需要手动下载
if result['success'] and result.get('instance_id'):
    download_result = operation.download_batch_results(
        batch_results=result['batch_results'],
        output_dir="./output",
        instance_id=result['instance_id']
    )
    print(f"📥 下载了 {download_result['download_count']} 个视频")
    print(f"💰 最终成本：${download_result['final_cost_usd']:.4f}")

# 输出示例：
# ✅ 处理了 2 个视频
# 💰 处理成本：$0.0187
# 📥 下载了 2 个视频
# 💰 最终成本：$0.0195
```

### 方法 3：大批量并行处理

同时在多个实例上处理 100+ 个场景：

```python
from instant_instance_operation_v2 import execute_parallel_batches

# 使用 10 个实例处理 100 个场景
# 注意：execute_parallel_batches 默认使用 auto_terminate=True！
result = execute_parallel_batches(
    scenes=my_100_scenes,       # 你的 100+ 个场景列表
    scenes_per_batch=10,        # 每个实例期望处理的场景数
    language="chinese",
    enable_zoom=True,
    max_parallel_instances=10,  # 同时启动最多 10 个实例
    min_scenes_per_batch=5,     # 每个实例最少处理 5 个场景（避免启动成本浪费）
    saving_dir="./my_videos"    # 可选：指定保存视频的目录（默认：./cloudburst_results/）
)

# 返回结果包含效率指标：
print(f"✅ 处理完成 {result['successful_scenes']}/{result['total_scenes']} 个场景")
print(f"💰 总成本：${result['total_cost_usd']:.4f}")
print(f"⏱️  并行时间：{result['parallel_time']:.1f}秒")
print(f"🚀 加速比：{result['efficiency']['speedup_factor']:.1f}倍速度！")
print(f"📥 已下载：{len(result['downloaded_files'])} 个视频")

# 访问下载的视频
for file_info in result['downloaded_files']:
    print(f"批次 {file_info['batch_id']}: {file_info['file_path']}")

# 输出示例：
# ✅ 处理完成 98/100 个场景
# 💰 总成本：$2.1534
# ⏱️  并行时间：265.3秒
# 🚀 加速比：9.4倍速度！
# 📥 已下载：98 个视频
# 📁 文件保存在：/tmp/batch_*/ 目录中
```

**智能分配算法：**
- 总场景数 ≤ 期望值 × 最大实例数：使用期望的 scenes_per_batch
- 总场景数 > 期望值 × 最大实例数：均匀分配到所有实例
- 自动调整以确保每个实例至少处理 min_scenes_per_batch 个场景

**分配示例：**
- 50 场景，batch=10，min=5 → 5 个实例 × 10 场景
- 120 场景，batch=10，min=5 → 10 个实例 × 12 场景（均匀分配）
- 24 场景，batch=10，min=8 → 3 个实例 × 8 场景（避免浪费）

**核心优势：**
- **速度提升 10 倍** 相比顺序处理
- **成本相同**（无论如何都要支付计算时间）
- **智能分配** - 根据最小场景数自动优化实例使用
- **自动下载** - 所有视频在实例终止前自动保存到本地（默认 auto_terminate=True）
- **三重安全保障** - 即使出错也保证实例清理
- **结果按场景名排序** 便于后续处理

**安全特性：**
- ✅ 下载完成后自动终止实例
- ✅ 批处理失败时立即清理
- ✅ 意外错误时紧急清理机制
- ✅ 确保无实例遗留 = 无意外账单

**适用场景：**
- 处理完整的视频课程（50-200 个场景）
- 为多个客户批量生成内容
- 需要快速完成的时间敏感批处理任务
- 需要成本控制的生产工作负载

## 📊 真实案例

生成 15 个带特效的视频：
- ✅ 成功生成 15 个视频
- 💰 总成本：$0.0187
- ⏱️ 总时间：385.2 秒

**对比传统方式**：
- 24/7 GPU 实例：~$500/月
- 使用 CloudBurst：~$0.02/批次

## 💡 主要特性

### 1. **智能实例管理**
- 自动回退到备用实例类型
- 优雅处理 AWS 配额限制
- 支持竞价实例（即将推出）

### 2. **实时成本追踪**
- 使用 AWS Pricing API 获取准确成本
- 精确到秒的运行时间追踪
- 返回简单的浮点数便于计费

### 3. **为自动化而生**
- 干净的 Python 字典返回格式
- 为数据库存储准备就绪
- 完美适配 CI/CD 流水线

### 4. **生产就绪**
- 全面的错误处理
- 失败时自动清理
- 详细的进度日志

## 📈 使用场景

- **🎬 视频/音频处理** - 在 GPU 实例上处理媒体文件
- **🤖 机器学习模型训练** - 无需 24/7 GPU 成本即可训练模型
- **📊 数据处理** - 按需运行大数据任务
- **🧪 测试与 CI/CD** - 自动创建测试环境
- **🎨 渲染** - 3D 渲染、图像处理

## 🛠️ 高级配置

### 下载行为示例

**自动下载（生产环境推荐）：**
```python
# 文件在终止前自动下载
result = operation.execute_batch(
    scenes=scenes,
    auto_terminate=True  # 处理 → 下载 → 终止
)

# 直接访问已下载的文件
for file_path in result['downloaded_files']:
    print(f"视频已保存：{file_path}")
```

**手动下载（用于开发/调试）：**
```python
# 保持实例运行以便检查
result = operation.execute_batch(
    scenes=scenes,
    auto_terminate=False  # 处理 → 保持运行
)

# 准备好后手动下载
if result['success']:
    download_result = operation.download_batch_results(
        batch_results=result['batch_results'],
        output_dir="./output",
        instance_id=result['instance_id']
    )
    
    # 然后手动终止
    operation.terminate_instance()
```

### 实例优先级（自动回退）
```python
# 在 __init__ 方法中配置
self.instance_configs = [
    {
        "priority": 1,
        "instance_type": "c5.2xlarge",   # CPU 优化型
        "description": "计算优化 - 8 vCPU, 16GB RAM",
        "hourly_cost": "$0.475/小时"     # ~$0.008/分钟
    },
    {
        "priority": 2, 
        "instance_type": "m5.xlarge",    # 内存优化型
        "description": "内存优化 - 4 vCPU, 16GB RAM",
        "hourly_cost": "$0.192/小时"     # ~$0.003/分钟
    },
    {
        "priority": 3,
        "instance_type": "g4dn.xlarge",  # GPU 用于 ML/视频
        "description": "NVIDIA T4 - 4 vCPU, 16GB RAM, 16GB GPU",
        "hourly_cost": "$0.526/小时"     # ~$0.009/分钟
    }
]
```

### 💰 AWS 实例价格参考
*最后更新：2025年8月6日 20:23 UTC | 区域：us-east-1（北弗吉尼亚）| Linux 按需实例*

| 实例类型 | vCPU | 内存 | GPU | 每小时费用 | 每分钟 | 25分钟任务 |
|---------|------|------|-----|-----------|--------|-----------|
| **c5.2xlarge** ⭐ | 8 | 16GB | - | $0.475 | $0.0079 | $0.20 |
| **m5.xlarge** | 4 | 16GB | - | $0.192 | $0.0032 | $0.08 |
| **g4dn.xlarge** | 4 | 16GB | T4 16GB | $0.526 | $0.0088 | $0.22 |
| t3.xlarge | 4 | 16GB | - | $0.234 | $0.0039 | $0.10 |
| t3.large | 2 | 8GB | - | $0.083 | $0.0014 | $0.03 |
| c5.xlarge | 4 | 8GB | - | $0.170 | $0.0028 | $0.07 |
| m5.large | 2 | 8GB | - | $0.164 | $0.0027 | $0.07 |
| c5.4xlarge | 16 | 32GB | - | $6.680 | $0.1113 | $2.78 |
| g4dn.2xlarge | 8 | 32GB | T4 16GB | $0.752 | $0.0125 | $0.31 |

⭐ = 默认/推荐实例类型

**注意**：价格为 Linux 按需实例价格。竞价实例可便宜 50-90%。价格可能因区域而异。


## 📋 系统要求

- Python 3.7+
- AWS 账户（需要 EC2 权限）
- boto3, requests, python-dotenv

## 🚨 AWS 账号限制/配额问题处理

如果使用 CloudBurst 导致 AWS 账号被限制或配额受限，请参考以下处理方法：

### 常见触发原因：
- 频繁创建和终止实例（CloudBurst 的核心功能）
- AWS 安全算法可能将此标记为可疑活动
- 可能的 API 密钥泄露担忧

### 解决指南：
有关如何解决 AWS 账号限制问题以及如何与 AWS 支持团队有效沟通的详细步骤，请参阅：
- 📖 **[我们如何将一次 AWS 安全警报转化为学习机会](https://laogege.org/aws-security/)**

该指南涵盖：
- 什么会触发 AWS 安全警报
- 如何与 AWS 支持团队沟通
- 预防未来问题的最佳实践
- 我们 6 小时的解决经历

### 预防建议：
- 使用具有最小必需权限的 AWS IAM 角色
- 监控您的 CloudBurst 使用模式
- 考虑在批处理运行之间实施速率限制
- 保持 AWS 凭证安全并定期轮换

## 🤝 贡献

这是我的第二个开源项目！欢迎：
- 报告 bug
- 建议新功能
- 提交 PR
- 如果觉得有用，请给个 Star！⭐

## 📊 真实生产环境基准测试

### 🚀 最新生产测试：55 场景并行处理（2025年8月7日）

**来自 RunPod 生产环境的真实测试结果：**

```
============================================================
🚀 并行批处理完成
============================================================
✅ 成功场景：55/55
❌ 失败场景：0
💰 总成本：$0.7184
⏱️  并行时间：588.4秒（节省 4861.9秒）
🚀 加速比：9.26倍速度
📥 下载文件：55 个视频
📁 文件保存在 3 个目录中
```

**关键指标：**
- **工作负载**：55 个教育故事场景
- **成功率**：100%（55/55）
- **并行处理时间**：9.8 分钟（588.4 秒）
- **顺序处理时间（如果不并行）**：91 分钟（5450.3 秒）
- **节省时间**：81.2 分钟（4861.9 秒）
- **加速倍数**：9.26 倍
- **总成本**：$0.72 美元
- **每场景成本**：$0.013 美元

### 性能分析

基于 AWS EC2 c5.2xlarge 实例的实际生产测试：

#### 并行处理分配（55 场景）
- **批次 1**：5 场景 → 约 9 分钟完成
- **批次 2**：12 场景 → 约 9 分钟完成  
- **批次 3**：38 场景 → 约 9 分钟完成
- **总计**：3 个实例同时运行

#### 视频生成性能（历史数据）
- **15 场景测试**：~25 分钟，$0.20 美元
- **55 场景测试**：~9.8 分钟（并行），$0.72 美元
- **实例启动时间**：每个实例 ~2 分钟
- **成功率**：所有测试均达 100%

### 场景性能对比
| 场景类型 | 视频数 | 平均时间/视频 | 总时间 | 文件大小 |
|----------|--------|---------------|--------|----------|
| 基础版 | 15 | 6.5秒 | 2.2 分钟 | ~0.8MB/个 |
| 仅字幕 | 15 | 6.7秒 | 2.4 分钟 | ~0.8MB/个 |
| 仅特效 | 15 | 34.0秒 | 9.3 分钟 | ~3.5MB/个 |
| 完整版 | 15 | 34.5秒 | 9.6 分钟 | ~3.5MB/个 |

### 💡 成本对比
- **传统 24/7 GPU 实例**：~$500/月
- **CloudBurst（100 批次/月）**：~$20/月
- **节省**：96% 成本降低！

**真正的"即插即用"体验**：从零到 55 个视频，仅需不到 10 分钟，只花 $0.72！

## 🆚 CloudBurst vs CloudBurst Fargate

| 特性 | CloudBurst (EC2) | CloudBurst Fargate |
|------|------------------|-------------------|
| **架构** | 基于实例 | 无服务器 |
| **服务器管理** | 手动（启动/停止） | 自动 |
| **计费方式** | 按分钟 | 按秒 |
| **伸缩性** | 手动 | 自动伸缩 |
| **启动时间** | 1-2 分钟 | 30-60 秒 |
| **维护** | 需要 | 零维护 |
| **适用于** | - 特定实例需求<br>- 现有 EC2 工作流<br>- 自定义配置 | - 大多数使用场景<br>- 生产工作负载<br>- 成本优化 |

**👉 我们推荐新部署使用 [CloudBurst Fargate](https://github.com/preangelleo/cloudburst-fargate)。**

## 📄 许可证

MIT License

---

**停止为闲置的云实例付费 - 仅在需要时使用它们！**

🎉 感谢使用 CloudBurst！如有问题，欢迎在 [Issues](https://github.com/preangelleo/cloudburst/issues) 中提出。