# Cloudburst - Instant Instance Operation

My second open source project! 🚀

[中文版](./README_CN.md)

> ⚠️ **Important Update**: We now recommend using **[CloudBurst Fargate](https://github.com/preangelleo/cloudburst-fargate)** for new deployments. CloudBurst Fargate offers:
> - 🚀 Serverless architecture (no servers to manage)
> - 💰 Pay-per-second billing (more cost-efficient)
> - ⚡ Auto-scaling capabilities
> - 🔧 Zero maintenance overhead
>
> This EC2-based version is maintained for users who need specific instance control or have existing EC2 workflows.

## What is this?

A Python framework that **automatically manages AWS EC2 instances** for on-demand video generation tasks. 

**Core Value**: When your application needs to generate videos (using our [Video Generation API](https://github.com/preangelleo/video-generation-docker)), this framework:
- 🚀 Spins up an EC2 instance with the video generation Docker container
- 🎬 Processes your video generation requests in parallel
- 📥 Downloads the completed videos
- 🛑 Immediately terminates the instance to save costs
- 💰 You only pay for the actual processing time (minutes, not months!)

**Perfect for**: Production applications that need to generate videos as part of their workflow but don't want to maintain 24/7 GPU instances.

## 🔗 Built on Video Generation API

This project leverages my first open source project: [**Video Generation API**](https://github.com/preangelleo/video-generation-docker)

### The Perfect Combination:
- **[Video Generation API](https://github.com/preangelleo/video-generation-docker)**: The core Docker image that actually generates the videos
- **CloudBurst (This Project)**: Automates AWS deployment to run the API on-demand

### Two Deployment Options:
| Option | Best For | Cost Model | Setup |
|--------|----------|------------|-------|
| **Video Generation API** | Frequent usage, always-on service | Pay monthly (~$500) | Run Docker container 24/7 |
| **CloudBurst** | Occasional usage, batch processing | Pay per use (~$0.20/batch) | Auto create/destroy instances |

CloudBurst automatically pulls and deploys the Video Generation API Docker image, giving you the same powerful video generation capabilities with 96% cost savings!

## 🎯 The Problem It Solves

Running cloud instances 24/7 is expensive. This framework:
- ✅ Creates instance only when needed
- ✅ Runs your task (e.g., video generation)
- ✅ Downloads results
- ✅ Terminates instance automatically
- ✅ Shows exact cost (e.g., $0.018 for 15 videos)

**You pay by the minute, not by the month!**

## 📋 Prerequisites

Before you start, make sure you have:

### 1. AWS Account with Required Permissions
- **AWS Access Key ID** and **AWS Secret Access Key**
- IAM permissions for:
  - EC2 (create, terminate, describe instances)
  - VPC (security groups, subnets)
  - EC2 Pricing API access

### 2. AWS Resources (Must Create in AWS Console First)
- **EC2 Key Pair** - for SSH access to instances (create in EC2 → Key Pairs)
- **Security Group** - with port 5000 open for API access (create in EC2 → Security Groups)
- **VPC Subnet ID** - where instances will be launched (find in VPC → Subnets)

### 3. Local Requirements
- Python 3.7+
- pip (Python package manager)
- Internet connection for AWS API calls

### 4. AWS Credentials Setup

The framework uses boto3, which automatically finds AWS credentials in this order:

1. **Environment variables** (recommended for production)
```bash
export AWS_ACCESS_KEY_ID=your-access-key
export AWS_SECRET_ACCESS_KEY=your-secret-key
export AWS_DEFAULT_REGION=us-east-1
```

2. **AWS CLI configuration** (if you have AWS CLI installed)
```bash
aws configure
```

3. **AWS credentials file** (automatically created by AWS CLI)
```
~/.aws/credentials
```

**Note**: You don't need to create any files manually. Just set the environment variables or use AWS CLI.

## ⚡ Critical: Download Behavior

**NEW**: Videos are now downloaded **immediately** after each scene is generated for maximum safety!

| Setting | Behavior | When to Use |
|---------|----------|-------------|
| `auto_terminate=True` | Process & Download each scene → Terminate | Production batches, automated workflows |
| `auto_terminate=False` (default) | Process & Download each scene → Keep alive | Development, debugging, retry failed downloads |

**Key Points**:
- ✅ **Immediate downloads**: Each video is saved to disk right after generation
- ✅ **Interrupt-safe**: If process crashes or you Ctrl+C, you keep all downloaded videos
- ✅ Downloaded files are in `result['downloaded_files']` and `result['download_dir']`
- ✅ No more lost work due to download failures or interruptions
- 📁 Files saved to: `{RESULTS_DIR}/batch_YYYYMMDD_HHMMSS/scene_name.mp4`

### Output Directory Configuration

All processing functions now support a `saving_dir` parameter with the same 3-tier priority:
1. **User-provided `saving_dir`** (highest priority)
2. **`RESULTS_DIR` from environment/.env**
3. **Default: `./cloudburst_results/`**

**Single batch processing (`execute_batch`)**:
```python
result = operation.execute_batch(
    scenes=scenes,
    saving_dir="/path/to/my/videos"  # Optional: specify where to save
)
# Files saved to: {saving_dir}/batch_YYYYMMDD_HHMMSS/
```

**Parallel processing (`execute_parallel_batches`)**:
```python
result = execute_parallel_batches(
    scenes=scenes,
    saving_dir="/path/to/my/videos"  # Optional: specify where to save
)
# Files saved to: {saving_dir}/batch_N_timestamp/
```

**Folder scanning (`scan_and_test_folder`)**:
```python
result = scan_and_test_folder(
    folder_path="./scenes",
    saving_dir="/path/to/my/videos"  # Optional: specify where to save
)
```

## 🚀 Quick Start

### 1. Install

```bash
git clone https://github.com/preangelleo/cloudburst
cd cloudburst
pip install -r requirements.txt
```

### 2. Configure

Create `.env` file:
```env
# AWS Settings
AWS_REGION=us-east-1
AWS_KEY_PAIR_NAME=your-keypair-name
AWS_SECURITY_GROUP_ID=sg-xxxxxxxxx
AWS_SUBNET_ID=subnet-xxxxxxxxx

# AWS Credentials (Required)
AWS_ACCESS_KEY_ID=your-aws-access-key-id
AWS_SECRET_ACCESS_KEY=your-aws-secret-access-key

# Optional: specific AMI or instance type
AWS_INSTANCE_TYPE=c5.2xlarge
DOCKER_IMAGE=betashow/video-generation-api:latest
```

### 3. Use It

```python
from instant_instance_operation_v2 import InstantInstanceOperationV2

# Initialize
operation = InstantInstanceOperationV2()

# Create instance, run task, terminate
instance_id, instance_ip = operation.create_instance()

# Deploy your Docker container
operation.deploy_and_run_docker(
    instance_ip=instance_ip,
    docker_image="your-docker-image:latest"
)

# Call your API
response = operation.call_api(
    api_url=f"http://{instance_ip}:5000/your-endpoint",
    request_data={"your": "data"}
)

# Download results
operation.download_file(
    instance_ip=instance_ip,
    remote_path="/results/output.zip",
    local_path="./output.zip"
)

# Terminate and get cost
total_cost = operation.terminate_instance()
print(f"Total cost: ${total_cost:.4f}")
```

## 📊 Real Examples - Video Generation

### Method 1: Automatic Folder Scanning

**Required folder structure**:
```
video_scenes/
├── images/
│   ├── scene_001_chinese.png
│   └── scene_002_chinese.png
├── audio/
│   ├── scene_001_chinese.mp3
│   ├── scene_002_chinese.mp3
│   ├── scene_001_chinese.srt (optional)
│   └── scene_002_chinese.srt (optional)
```

**Note**: Files must follow naming pattern `scene_XXX_chinese.*` or `scene_XXX_english.*`

```python
from instant_instance_operation_v2 import scan_and_test_folder

# Note: scan_and_test_folder uses auto_terminate=True by default!
result = scan_and_test_folder(
    folder_path="./video_scenes/",
    language="chinese",
    enable_zoom=True
)

print(f"✅ Generated {result['successful_scenes']} videos")
print(f"💰 Total cost: ${result['cost_usd']:.4f}")

# Files are automatically downloaded!
if result.get('downloaded_files'):
    print(f"📥 Videos saved to: {result['output_directory']}")
    for file_path in result['downloaded_files']:
        print(f"   🎬 {file_path}")
```

### 🎬 Output Examples

Here's what you can expect from the generated videos:

**English Example**:
[![English Video Example](https://img.youtube.com/vi/JiWsyuyw1ao/maxresdefault.jpg)](https://www.youtube.com/watch?v=JiWsyuyw1ao)

**Chinese Example**:
[![Chinese Video Example](https://img.youtube.com/vi/eJtuwBV_7qY/maxresdefault.jpg)](https://www.youtube.com/watch?v=eJtuwBV_7qY)

**Video Features Demonstrated**:
- ✅ Professional subtitles with background
- ✅ Smooth zoom effects (Ken Burns effect)
- ✅ Synchronized audio and visuals
- ✅ High-quality video output (1080p)

Both examples show the "Full Featured" mode with subtitles and effects enabled.

### Method 2: Custom Scene List (More Flexible)

```python
from instant_instance_operation_v2 import InstantInstanceOperationV2

# Initialize
operation = InstantInstanceOperationV2()

# Define your scenes with any file paths
scenes = [
    {
        "scene_name": "intro_video",
        "input_image": "/path/to/intro.png",
        "input_audio": "/path/to/intro.mp3",
        "subtitle": "/path/to/intro.srt"  # Optional, can be None
    },
    {
        "scene_name": "main_content",
        "input_image": "/path/to/main.png", 
        "input_audio": "/path/to/main.mp3",
        "subtitle": None  # No subtitle for this scene
    }
]

# Process scenes
result = operation.execute_batch(
    scenes=scenes,
    language="english",
    enable_zoom=True,
    auto_terminate=False  # Keep alive for manual download
)

print(f"✅ Processed {result['successful_scenes']} videos")
print(f"💰 Processing cost: ${result['cost_usd']:.4f}")

# Videos already downloaded during processing!
print(f"📥 Downloaded {result['download_count']} videos to: {result['download_dir']}")

# Optional: Terminate instance manually when done
if result.get('instance_id'):
    operation.terminate_instance(result['instance_id'])
    print("✅ Instance terminated")

# Output example:
# ✅ Processed 2 videos
# 💰 Processing cost: $0.0187
# 📥 Downloaded 2 videos
# 💰 Final cost: $0.0195
```

### Method 3: Parallel Processing for Large Batches

Process 100+ scenes across multiple instances simultaneously:

```python
from instant_instance_operation_v2 import execute_parallel_batches

# Process 100 scenes using 10 instances
# Note: execute_parallel_batches uses auto_terminate=True by default!
result = execute_parallel_batches(
    scenes=my_100_scenes,       # Your list of 100+ scenes
    scenes_per_batch=10,        # Preferred scenes per instance
    language="english",
    enable_zoom=True,
    max_parallel_instances=10,  # Launch up to 10 instances at once
    min_scenes_per_batch=5,     # Minimum 5 scenes per instance (avoid startup waste)
    saving_dir="./my_videos"    # Optional: Where to save videos (default: ./cloudburst_results/)
)

# Return structure includes efficiency metrics:
print(f"✅ Processed {result['successful_scenes']}/{result['total_scenes']} scenes")
print(f"💰 Total cost: ${result['total_cost_usd']:.4f}")
print(f"⏱️  Parallel time: {result['parallel_time']:.1f}s")
print(f"🚀 Speedup: {result['efficiency']['speedup_factor']:.1f}x faster!")
print(f"📥 Downloaded: {len(result['downloaded_files'])} videos")

# Access downloaded videos
for file_info in result['downloaded_files']:
    print(f"Batch {file_info['batch_id']}: {file_info['file_path']}")

# Output example:
# ✅ Processed 98/100 scenes  
# 💰 Total cost: $2.1534
# ⏱️  Parallel time: 265.3s
# 🚀 Speedup: 9.4x faster!
# 📥 Downloaded: 98 videos
# 📁 Files saved in: /tmp/batch_*/ directories
```

**Smart Distribution Algorithm:**
- Total scenes ≤ preferred × max_instances: Uses preferred scenes_per_batch
- Total scenes > preferred × max_instances: Evenly distributes across all instances
- Automatically adjusts to ensure each instance processes ≥ min_scenes_per_batch

**Distribution Examples:**
- 50 scenes, batch=10, min=5 → 5 instances × 10 scenes
- 120 scenes, batch=10, min=5 → 10 instances × 12 scenes (even distribution)
- 24 scenes, batch=10, min=8 → 3 instances × 8 scenes (avoid waste)

**Key Benefits:**
- **10x faster** than sequential processing
- **Same total cost** (you pay for compute time either way)
- **Smart allocation** - optimizes instance usage based on minimum scenes
- **Automatic downloads** - all videos saved locally before instances terminate (auto_terminate=True by default)
- **Triple-layer safety** - guaranteed instance cleanup even on errors
- **Results sorted by scene_name** for easy processing

**Safety Features:**
- ✅ Automatic instance termination after downloads
- ✅ Immediate cleanup on batch failures  
- ✅ Emergency cleanup in case of unexpected errors
- ✅ No instances left running = no surprise bills

**Perfect for:**
- Processing entire video courses (50-200 scenes)
- Bulk content generation for multiple clients
- Time-sensitive batch jobs that need to complete quickly
- Production workloads requiring cost control

## 💡 Key Features

### 1. **Smart Instance Management**
- Automatic fallback to alternative instance types
- Handles AWS quota limits gracefully
- Supports spot instances (coming soon)

### 2. **Real-time Cost Tracking**
- Uses AWS Pricing API for accurate costs
- Tracks runtime down to the second
- Returns cost as simple float for easy billing

### 3. **Built for Automation**
- Clean Python dictionary return format
- Ready for database storage
- Perfect for CI/CD pipelines

### 4. **Production Ready**
- Comprehensive error handling
- Automatic cleanup on failure
- Detailed progress logging

## 📈 Use Cases

- **🎬 Video/Audio Processing** - Process media files on GPU instances
- **🤖 ML Model Training** - Train models without 24/7 GPU costs
- **📊 Data Processing** - Run big data jobs on demand
- **🧪 Testing & CI/CD** - Spin up test environments automatically
- **🎨 Rendering** - 3D rendering, image processing

## 🛠️ Advanced Configuration

### Download Behavior Examples

**Immediate Downloads (NEW - Default behavior):**
```python
# Videos download immediately as they're generated
result = operation.execute_batch(
    scenes=scenes,
    auto_terminate=True  # Each scene: Generate → Download → Next → Terminate
)

# All videos already downloaded during processing!
print(f"✅ Downloaded {result['download_count']} videos")
print(f"📁 Saved to: {result['download_dir']}")
for file_path in result['downloaded_files']:
    print(f"  → {file_path}")
```

**Keep Instance Alive (For debugging or retry):**
```python
# Process and download, but keep instance for inspection
result = operation.execute_batch(
    scenes=scenes,
    auto_terminate=False  # Generate & Download all → Keep alive
)

# Videos already downloaded! Instance still running for:
# - Debugging failed scenes
# - Re-downloading specific files  
# - Running additional batches

# Remember to terminate when done:
if result.get('instance_id'):
    operation.terminate_instance(result['instance_id'])
```

### Instance Priority (Automatic Fallback)
```python
# Configure in __init__ method
self.instance_configs = [
    {
        "priority": 1,
        "instance_type": "c5.2xlarge",   # CPU optimized
        "description": "Compute optimized - 8 vCPU, 16GB RAM",
        "hourly_cost": "$0.475/hour"     # ~$0.008/minute
    },
    {
        "priority": 2, 
        "instance_type": "m5.xlarge",    # Memory optimized
        "description": "Memory optimized - 4 vCPU, 16GB RAM",
        "hourly_cost": "$0.192/hour"     # ~$0.003/minute
    },
    {
        "priority": 3,
        "instance_type": "g4dn.xlarge",  # GPU for ML/video
        "description": "NVIDIA T4 - 4 vCPU, 16GB RAM, 16GB GPU",
        "hourly_cost": "$0.526/hour"     # ~$0.009/minute
    }
]
```

### 💰 AWS Instance Pricing Reference
*Last updated: August 6, 2025 20:23 UTC | Region: us-east-1 (N. Virginia) | Linux On-Demand*

| Instance Type | vCPU | RAM | GPU | Hourly Cost | Per Minute | 25-min Job |
|--------------|------|-----|-----|-------------|------------|------------|
| **c5.2xlarge** ⭐ | 8 | 16GB | - | $0.475 | $0.0079 | $0.20 |
| **m5.xlarge** | 4 | 16GB | - | $0.192 | $0.0032 | $0.08 |
| **g4dn.xlarge** | 4 | 16GB | T4 16GB | $0.526 | $0.0088 | $0.22 |
| t3.xlarge | 4 | 16GB | - | $0.234 | $0.0039 | $0.10 |
| t3.large | 2 | 8GB | - | $0.083 | $0.0014 | $0.03 |
| c5.xlarge | 4 | 8GB | - | $0.170 | $0.0028 | $0.07 |
| m5.large | 2 | 8GB | - | $0.164 | $0.0027 | $0.07 |
| c5.4xlarge | 16 | 32GB | - | $6.680 | $0.1113 | $2.78 |
| g4dn.2xlarge | 8 | 32GB | T4 16GB | $0.752 | $0.0125 | $0.31 |

⭐ = Default/Recommended instance type

**Note**: Prices are for Linux On-Demand instances. Spot instances can be 50-90% cheaper. Prices may vary by region.

### Return Format for Developers
```python
{
    "success": True,
    "cost_usd": 0.0187,              # Direct float for billing
    "total_time": 385.2,             # Seconds
    "successful_scenes": 15,         # For batch operations
    "batch_results": [               # Individual results
        {
            "scene_name": "scene_001",
            "success": True,
            "download_url": "http://...",
            "processing_time": 18.5
        }
    ]
}
```

## 📋 Requirements

- Python 3.7+
- AWS Account with EC2 permissions
- boto3, requests, python-dotenv

## 🖥️ Real Terminal Output Example

Here's exactly what you'll see when processing a video with CloudBurst:

### Key Metrics from This Example:
- **Scene Details**: Single 27-second scene with Chinese audio and subtitles
- **Total Time**: 155.9 seconds (2.6 minutes) from start to finish
- **Processing Breakdown**:
  - Instance creation & boot: ~10 seconds
  - Docker container ready: ~75 seconds
  - Video generation: ~80 seconds
  - File download: ~5 seconds
  - Instance termination: ~2 seconds
- **Cost**: $0.021 (about 2 cents for a 27-second video)
- **Output**: 7.2MB MP4 file with zoom effects and Chinese subtitles

```
~/coding % python3 test_cloudburst_auto_download.py
⚠️  WARNING: This will create an AWS EC2 instance and incur charges!
   Estimated cost: ~$0.01-0.02 for a single scene

✨ NEW: auto_terminate=True now downloads files automatically!

Do you want to continue? (yes/no): yes
🚀 Testing CloudBurst with automatic download
============================================================
Scene: scene_024_chinese
Image: scene_024_chinese.png
Audio: scene_024_chinese.mp3
Subtitle: scene_024_chinese.srt
============================================================
⏱️  [08:44:00.763] +  0.00s - === BATCH INSTANT OPERATION START (1 scenes) ===
⏱️  [08:44:00.763] +  0.00s - Starting smart AWS instance creation
⏱️  [08:44:00.763] +  0.00s - 🔍 Checking instance availability before creation...
⏱️  [08:44:01.565] +  0.80s - ✅ Selected instance c5.2xlarge is available in 5 zones
⏱️  [08:44:02.537] +  1.77s - Instance pricing: $0.4750/hour (c5.2xlarge)
⏱️  [08:44:02.537] +  1.77s - 🚀 Attempting to create instance: c5.2xlarge (attempt 1/3)
⏱️  [08:44:04.222] +  3.46s - ✅ Instance created successfully: i-0xxxxxxxxxxxxxxxxx
⏱️  [08:44:04.222] +  3.46s - Waiting for instance to be running...
⏱️  [08:44:10.212] +  9.45s - Instance running at IP: 3.xxx.xxx.xxx
⏱️  [08:44:10.212] +  9.45s - Waiting for Docker API to be ready...
⏱️  [08:44:13.404] + 12.64s - Still waiting for API... attempt 1/120
⏱️  [08:45:05.386] + 64.62s - Still waiting for API... attempt 11/120
⏱️  [08:45:16.021] + 75.26s - Docker API is ready and healthy

⏱️  [08:45:16.022] + 75.26s - 📥 AUTO-DOWNLOAD ENABLED: Files will be downloaded automatically before termination

⏱️  [08:45:16.022] + 75.26s - === Processing Scene 1/1 ===
⏱️  [08:45:16.022] + 75.26s - Processing scene: scene_024_chinese
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Files encoded (with subtitles)
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Adding subtitle to request (length: 1680 chars)
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Using unified API endpoint (expected scenario: full_featured)
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Request has effects: ['zoom_in', 'zoom_out'] (enable_zoom=True)
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Request has subtitle: True (length=1680 chars)
⏱️  [08:45:16.041] + 75.28s - Scene scene_024_chinese: Request language: chinese
⏱️  [08:45:16.042] + 75.28s - Scene scene_024_chinese: Debug request saved to /path/to/project/Temps/debug_request_scene_024_chinese.json
⏱️  [08:45:16.042] + 75.28s - Scene scene_024_chinese: Sending request to /create_video_onestep...
⏱️  [08:46:36.664] +155.90s - Scene scene_024_chinese: Completed - xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx (7.17MB) in 80.6s
⏱️  [08:46:36.664] +155.90s - Scene scene_024_chinese: API detected scenario: full_featured
⏱️  [08:45:36.230] + 95.47s - 📥 Downloading scene_024_chinese immediately...
⏱️  [08:45:41.285] +100.52s - ✅ Downloaded scene_024_chinese: 7.17MB → output/batch_20240807_084516/scene_024_chinese.mp4
⏱️  [08:46:36.665] +155.90s - === BATCH PROCESSING COMPLETED: 1/1 scenes successful in 155.90s ===
⏱️  [08:46:36.665] +155.90s - ✅ Downloaded 1 videos to: output/batch_20240807_084516
⏱️  [08:46:36.665] +155.90s - Current estimated cost: $0.020336 (runtime: 2.57min)
⏱️  [08:46:36.665] +155.90s - ⚠️  Keeping instance alive for potential retry downloads...
⏱️  [08:46:41.726] +160.96s - Terminating instance: i-0xxxxxxxxxxxxxxxxx
⏱️  [08:46:42.958] +162.19s - Instance termination initiated
⏱️  [08:46:42.959] +162.20s - Downloaded 1 files to: /path/to/project/Temps/instant_test_results/batch_20250807_084636

============================================================
📊 CLOUDBURST RESULTS
============================================================
✅ Success: 1/1 scenes processed
💰 Final cost: $0.0210
⏱️  Total time: 155.9 seconds
🖥️  Instance type: c5.2xlarge

📥 Downloaded 1 files automatically!
📁 Output directory: /path/to/project/Temps/instant_test_results/batch_20250807_084636
   🎬 /path/to/project/Temps/instant_test_results/batch_20250807_084636/scene_024_chinese_20250807_084641.mp4 (7.2 MB)

✅ Test completed!
```

### What This Tells You:
1. **Detailed Timing**: Every step is timestamped so you know exactly where time is spent
2. **Smart Instance Selection**: Automatically checks availability across AWS zones
3. **Real-time Progress**: See when Docker is loading, when processing starts, and download progress
4. **Cost Transparency**: Shows hourly rate ($0.475/hour) and final cost ($0.021)
5. **Automatic Downloads**: Files are saved locally before instance termination
6. **Clean Shutdown**: Instance is properly terminated to avoid ongoing charges

### Performance Insights:
- **Setup Overhead**: ~75 seconds for AWS instance + Docker (one-time cost per batch)
- **Processing Speed**: ~3 seconds per second of video (27-second audio → 80 seconds processing)
- **Cost Efficiency**: $0.021 for one video, but if you process 100 videos in the same batch, the setup cost is amortized

This example shows the "full_featured" scenario with both subtitles and zoom effects, which takes the longest to process but produces the highest quality output.

## 🚨 AWS Account Suspension/Quota Limits

If using CloudBurst leads to AWS account suspension or quota limitations, here's what to do:

### Common Triggers:
- Frequent instance creation and termination (CloudBurst's core functionality)
- AWS security algorithms may flag this as suspicious activity
- Potential API key exposure concerns

### Resolution Guide:
For a detailed walkthrough of how to resolve AWS account limitations and communicate with AWS Support effectively, see:
- 📖 **[How We Turned an AWS Security Scare into a Learning Opportunity](https://sumatman.ai/how-we-turned-an-aws-security-scare-into-a-learning-opportunity/)**

This guide covers:
- What triggers AWS security alerts
- How to communicate with AWS Support
- Best practices to prevent future issues
- Our 6-hour resolution experience

### Prevention Tips:
- Use AWS IAM roles with minimal required permissions
- Monitor your CloudBurst usage patterns
- Consider implementing rate limiting between batch runs
- Keep your AWS credentials secure and rotate them regularly

## 🤝 Contributing

This is my second open source project! Feel free to:
- Report bugs
- Suggest features  
- Submit PRs
- Star if you find it useful! ⭐

## 📊 Real Production Benchmarks

### 🚀 Latest Production Test: 55 Scenes Parallel Processing (August 7, 2025)

**Real-world test results from RunPod production environment:**

```
============================================================
🚀 PARALLEL BATCH PROCESSING COMPLETE
============================================================
✅ Successful scenes: 55/55
❌ Failed scenes: 0
💰 Total cost: $0.7184
⏱️  Parallel time: 588.4s (saved 4861.9s)
🚀 Speedup: 9.26x faster
📥 Downloaded files: 55 videos
📁 Files saved in 3 directories
```

**Key Metrics:**
- **Workload**: 55 educational story scenes
- **Success Rate**: 100% (55/55)
- **Parallel Processing Time**: 9.8 minutes (588.4 seconds)
- **Sequential Time (if not parallel)**: 91 minutes (5450.3 seconds)
- **Time Saved**: 81.2 minutes (4861.9 seconds)
- **Speedup Factor**: 9.26x faster
- **Total Cost**: $0.72 USD
- **Cost per Scene**: $0.013 USD

### Performance Analysis

Based on actual production testing with AWS EC2 c5.2xlarge instances:

#### Parallel Processing Distribution (55 scenes)
- **Batch 1**: 5 scenes → Completed in ~9 minutes
- **Batch 2**: 12 scenes → Completed in ~9 minutes  
- **Batch 3**: 38 scenes → Completed in ~9 minutes
- **Total**: 3 instances running simultaneously

#### Video Generation Performance (Historical)
- **15 scenes test**: ~25 minutes, $0.20 USD
- **55 scenes test**: ~9.8 minutes (parallel), $0.72 USD
- **Instance Setup Time**: ~2 minutes per instance
- **Success Rate**: 100% across all tests

### Scenario Performance
| Scenario | Videos | Avg Time/Video | Total Time | File Size |
|----------|--------|----------------|------------|-----------|
| Baseline | 15 | 6.5s | 2.2 min | ~0.8MB each |
| Subtitles Only | 15 | 6.7s | 2.4 min | ~0.8MB each |
| Effects Only | 15 | 34.0s | 9.3 min | ~3.5MB each |
| Full Featured | 15 | 34.5s | 9.6 min | ~3.5MB each |

### 💡 Cost Comparison
- **Traditional 24/7 GPU Instance**: ~$500/month
- **CloudBurst (100 batches/month)**: ~$20/month
- **Savings**: 96% cost reduction!

**Real "Plug and Play" Experience**: From zero to 55 videos in under 10 minutes for just $0.72!

## 🆚 CloudBurst vs CloudBurst Fargate

| Feature | CloudBurst (EC2) | CloudBurst Fargate |
|---------|------------------|-------------------|
| **Architecture** | Instance-based | Serverless |
| **Server Management** | Manual (start/stop) | Automatic |
| **Billing** | Per minute | Per second |
| **Scaling** | Manual | Auto-scaling |
| **Startup Time** | 1-2 minutes | 30-60 seconds |
| **Maintenance** | Required | Zero |
| **Best For** | - Specific instance needs<br>- Existing EC2 workflows<br>- Custom configurations | - Most use cases<br>- Production workloads<br>- Cost optimization |

**👉 We recommend [CloudBurst Fargate](https://github.com/preangelleo/cloudburst-fargate) for new deployments.**

## 📄 License

MIT License

---

**Stop paying for idle cloud instances - use them only when you need them!**