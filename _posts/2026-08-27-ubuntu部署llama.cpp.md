---
title: 'ubuntu部署llama.cpp'
date: 2026-08-27 14:11:00 +0000
author: wioponsen
categories: [blog]
tags: [blog]
math: true
mermaid: true
---

Step 1: Install Build Tools

```shell
sudo apt update
sudo apt install -y build-essential cmake git
```

Step 2: Clone and Compile with CUDA

```shell
# 1. Clone the repository
git clone https://github.com
cd llama.cpp

# 2. Configure the build with CUDA enabled
cmake -B build -DGGML_CUDA=ON

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
    --ctx-size 4096 \
    -ngl 99


```
--host 0.0.0.0: 允许局域网内的其他电脑（比如你的 Mac mini 或笔记本）访问这台 Ubuntu 服务器。
--ctx-size 4096: 设置模型的上下文窗口大小。
启动后，该服务会完全兼容 OpenAI 的 API 格式，接口地址为 http://你的Ubuntu_IP:8080/v1。

Step 4: systemd demae

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


