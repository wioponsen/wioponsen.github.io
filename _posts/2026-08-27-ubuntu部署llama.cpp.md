---
title: 'ubuntu部署llama.cpp'
date: 2026-08-27 14:11:00 +0000
author: wioponsen
categories: [blog]
tags: [blog]
math: true
mermaid: true
---
{% raw %}

## Step 1: Install Build Tools

```shell
sudo apt update && sudo apt upgrade -y
sudo apt install -y build-essential cmake git wget python3-pip

wget https://developer.download.nvidia.com/compute/cuda/repos/wsl-ubuntu/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt-get update
sudo apt-get -y install cuda-toolkit-12-6

sudo apt install -y libssl-dev
# 多卡装nccl
sudo apt install -y libnccl2 libnccl-dev

# 如果有环境错误， 就删除旧的，有错误的，重新装toolkit
sudo apt purge -y nvidia-cuda-toolkit
sudo apt autoremove -y

```

## Step 2: Clone and Compile with CUDA

```shell
# 1. Clone the repository
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# 2. Configure the build with CUDA enabled
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON

# 3. Compile using all available CPU cores
cmake --build build --config Release -j $(nproc)

```

## Step 3: Run the Model using GPU Acceleration

```shell

#  终端与大模型进行交互
./build/bin/llama-cli -m models/your-model-q4_k_m.gguf \
    -p "You are a helpful programming assistant." \
    -cnv \
    -ngl 99

# 
./build/bin/llama-server -m models/DeepSeek-R1-Distill-Qwen-8B-Q4_K_M.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    --alias deepseek-r1 \
    --ctx-size 4096 \
    -ngl 99 \

    --split-mode row \
    --main-gpu 0  
```
--host 0.0.0.0: 允许局域网内的其他电脑（比如你的 Mac mini 或笔记本）访问这台 Ubuntu 服务器。
--ctx-size 4096: 设置模型的上下文窗口大小。
--alias deepseek-r1: 设置模型别名， 后续id和名称都可以使用这个别名
启动后，该服务会完全兼容 OpenAI 的 API 格式，接口地址为 http://你的Ubuntu_IP:8080/v1。

获取模型列表信息, 里面有id信息
```shell
curl ip:port/v1/models
```

## Step 4: Install and configure LiteLLM to support Anthropic API

```shell
pip install litellm[proxy]
pip install httpx[socks] --break-system-packages
pip install prisma --break-system-packages
```

自己安装 PostgreSQL（本地/服务器），并启动服务
```shell
sudo apt update
sudo apt install postgresql postgresql-contrib
sudo systemctl start postgresql
sudo systemctl enable postgresql

#创建用户
sudo -u postgres psql
# 内部进行设置：其中llmproxy是用户，数据库是litellm
CREATE USER llmproxy WITH PASSWORD 'password';
CREATE DATABASE litellm OWNER llmproxy;
GRANT ALL PRIVILEGES ON DATABASE litellm TO llmproxy;
\q

#可以得到数据库链接，默认端口是5432
DATABASE_URL="postgresql://llmproxy:password@localhost:5432/litellm"

#测试数据库
# 本地默认端口
pg_isready
# 指定主机和端口
pg_isready -h localhost -p 5432
```


Create a configuration file named config.yaml to map llama.cpp as an Anthropic-compatible endpoint:
https://docs.litellm.com.cn/docs/learn/gateway_quickstart
```yaml
# config.yaml
model_list:
  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://0.0.0.0:8080/v1
      api_key: "none"
      custom_llm_provider: openai

general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
  database_url: "postgresql://{user}:{pswd}@localhost:5432/litellm"
  store_model_in_db: true
```

配置账户， 或者直接export到环境，默认账户是 admin / {UI_PASSWORD}
```ini
# .env
UI_USERNAME=admin
UI_PASSWORD=pswd
LITELLM_MASTER_KEY=sk-1234
```

如果有多个模型需要支持，或者有其他厂商的模型需要统一中转，可以继续增加模型配置，而且可以设置fallback（失败自动切换）规则
```yaml
model_list:
  - model_name: claude-sonnet
    litellm_params:
      model: anthropic/claude-sonnet-4-5
      api_key: os.environ/ANTHROPIC_API_KEY

  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://127.0.0.1:8080/v1
      api_key: "none"

# 方式 A：写在 litellm_settings
litellm_settings:
  num_retries: 2                    # 每个模型先重试几次
  request_timeout: 60               # 超时秒数
  fallbacks:
    - claude-sonnet: ["qwen2.5-coder-7b"]   # Claude 挂了 → 本地 qwen, 允许多级fallback，可以在中括号中逐个顺序填写

# 或方式 B：写在 router_settings（效果类似）
#router_settings:
#  fallbacks:
#    - claude-sonnet: ["qwen2.5-coder-7b"]
```


Run LiteLLM and tell it to accept Anthropic-formatted requests:
```shell
litellm --config config.yaml --port 4000
```

至此，就可以在浏览器ui进行控制, 进入 http://{ip}:4000/ui/ , 就能生成token，选择model


## Step 5: One-Click Start Script

编写脚本启动服务：run_llama.sh
```sh
#!/bin/bash

#1. llama.cpp 配置
LLAMA_CMD="/path/to/llama-server" # llama-server 或 ./llama-cli 的绝对路径
MODEL_PATH="/path/to/qwen2.5-coder-7b-instruct-q6_k.gguf" # GGUF 模型文件的绝对路径
MODEL_ALIAS="qwen2.5-coder-7b"
LLAMA_PORT=8082
CONTENT_LENGTH=4096
LITE_LLAMA_PORT=4000

# 2. LiteLLM 配置
LITELLM_DIR="/home/w/works/llama/" # 包含 .env 和 config.yaml 的目录绝对路径
# ============================================

echo "[1/3] 正在启动 llama.cpp..."
# 后台启动 llama.cpp 并将日志保存到指定文件
$LLAMA_CMD -m $MODEL_PATH --host 0.0.0.0 --port $LLAMA_PORT --alias $MODEL_ALIAS --ctx-size $CONTENT_LENGTH -ngl 99 > ~/llama_server.log 2>&1 &
LLAMA_PID=$!

echo "[2/3] 等待 llama.cpp 端口 ($LLAMA_PORT) 就绪..."
# 循环检测端口，直到 llama.cpp 彻底启动成功
while ! nc -z localhost $LLAMA_PORT; do   
  sleep 1
done
echo "llama.cpp 已成功就绪 (PID: $LLAMA_PID)！"

echo "[3/3] 正在启动 LiteLLM Proxy..."
# 进入 LiteLLM 所在目录（确保能读到 .env 和 config.yaml）
cd $LITELLM_DIR

# 启动 LiteLLM（如果是 pip 安装的使用下面这行）
litellm --config config.yaml --port $LITE_LLAMA_PORT > ~/litellm_proxy.log 2>&1 &

# 如果你是用 Docker Compose 启动 LiteLLM，请注释掉上面那行，改用下面这行：
# docker-compose up -d

echo "LiteLLM 已在后台启动！"
echo "你可以通过 'tail -f ~/litellm_proxy.log' 查看 LiteLLM 日志。"
echo "你可以通过 'tail -f ~/llama_server.log' 查看 llama.cpp 日志。"
```

写入：
```ini
[Unit]
Description=Llama.cpp and LiteLLM Auto Start Service
After=network.target

[Service]
Type=forking
# 替换为你的实际 Linux 用户名
User=your_username 
# 换成你刚才创建的脚本的绝对路径
ExecStart=/home/your_username/run_llama.sh
# 如果服务崩溃，自动重启
Restart=on-failure
# 给予脚本足够的时间运行（特别是加载大模型时）
TimeoutStartSec=300

[Install]
WantedBy=multi-user.target
```

启动并设置开机自启：
```shell
# 1. 重新加载 Systemd 配置
sudo systemctl daemon-reload
# 2. 启用开机自启
sudo systemctl enable ai-services.service
# 3. 立即手动启动测试
sudo systemctl start ai-services.service
```

## Step 6: others
多卡服务器，跑模型有两种情况：
1. 模型参数小，每个卡都能跑完整示例
2. 模型参数大，多张卡组合起来才能跑

*单卡装不下的大模型：*
 模型切分到多卡， 使用1个 llama-server 实例，使用参数 `--split-mode layer`， 并发小
 *每卡一份完整模型：*
 每张卡一个 llama-server，不同端口，每张卡一个独立的 llama.cpp 实例 + LiteLLM 做负载均衡

以四张卡，每张卡单独运行一份模型为例：

1. 每张卡起一个 llama.cpp 实例
```shell
# GPU 0 → 端口 8080
CUDA_VISIBLE_DEVICES=0 llama-server \
  -m /path/to/model.gguf \
  -ngl 99 \
  -c 8192 \
  --parallel 4 \
  --host 0.0.0.0 --port 8080 \
  --alias qwen2.5-coder-7b

# GPU 1 → 8081
CUDA_VISIBLE_DEVICES=1 llama-server \
  -m /path/to/model.gguf \
  -ngl 99 -c 8192 --parallel 4 \
  --host 0.0.0.0 --port 8081 \
  --alias qwen2.5-coder-7b

# GPU 2 → 8082
CUDA_VISIBLE_DEVICES=2 llama-server ... --port 8082 ...

# GPU 3 → 8083
CUDA_VISIBLE_DEVICES=3 llama-server ... --port 8083 ...
```

- CUDA_VISIBLE_DEVICES=N：把进程绑到第 N 张卡  
- --parallel 4（或 -np 4）：单实例同时处理的会话数（slot）；越大越占显存（主要是 KV cache）  
- -c：总上下文，会在多个 slot 之间分；例如 -c 32768 --parallel 4 ≈ 每路约 8k

2. 用 LiteLLM 统一成一个模型名（负载均衡）

config.yaml
```shell
model_list:
  # 同一个 model_name，多个 api_base → 自动负载均衡
  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://127.0.0.1:8080/v1
      api_key: "none"
      rpm: 60          # 可选，用于加权调度

  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://127.0.0.1:8081/v1
      api_key: "none"
      rpm: 60

  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://127.0.0.1:8082/v1
      api_key: "none"
      rpm: 60

  - model_name: qwen2.5-coder-7b
    litellm_params:
      model: openai/qwen2.5-coder-7b
      api_base: http://127.0.0.1:8083/v1
      api_key: "none"
      rpm: 60

router_settings:
  routing_strategy: least-busy   # 或 simple-shuffle
  # least-busy：优先发给当前在途请求少的实例，适合并发
  # simple-shuffle：随机/加权，简单好用

general_settings:
  master_key: sk-你的主密钥
```


{% endraw %}



