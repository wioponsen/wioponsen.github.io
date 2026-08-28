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

Step 1: Install Build Tools

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

Step 2: Clone and Compile with CUDA

```shell
# 1. Clone the repository
git clone https://github.com/ggml-org/llama.cpp.git
cd llama.cpp

# 2. Configure the build with CUDA enabled
cmake -B build -DGGML_CUDA=ON -DGGML_CUDA_F16=ON

# 3. Compile using all available CPU cores
cmake --build build --config Release -j $(nproc)

```

Step 3: Run the Model using GPU Acceleration

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

Step 4: systemd demae （可选）

```shell
sudo nano /etc/systemd/system/llama.service
```

写入：
```ini
[Unit]
Description=Llama.cpp API Server
After=network.target

[Service]
Type=simple
User=your_username
WorkingDirectory=/home/your_username/llama.cpp
ExecStart=/home/your_username/llama.cpp/build/bin/llama-server -m /home/your_username/llama.cpp/models/DeepSeek-R1-Distill-Qwen-8B-Q4_K_M.gguf --host 0.0.0.0 --port 8080 -ngl 99
Restart=on-failure

[Install]
WantedBy=multi-user.target

```

启动并设置开机自启：
```shell
sudo systemctl daemon-reload
sudo systemctl start llama
sudo systemctl enable llama
```

Step 5: Install and configure LiteLLM to support Anthropic API

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
      api_base: http://0.0.0.0:8082/v1
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


Run LiteLLM and tell it to accept Anthropic-formatted requests:
```shell
litellm --config config.yaml --port 4000
```

至此，就可以在浏览器ui进行控制, 进入 http://{ip}:4000/ui/ , 就能生成token，选择model


{% endraw %}



