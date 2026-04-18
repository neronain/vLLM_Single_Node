# vLLM_Single_Node
Install and use vLLM on DGX Spark

Configure Docker permissions
To easily manage containers without sudo, you must be in the docker group. If you choose to skip this step, you will need to run Docker commands with sudo.

Open a new terminal and test Docker access. In the terminal, run:
```bash
docker ps
```

If you see a permission denied error (something like permission denied while trying to connect to the Docker daemon socket), add your user to the docker group so that you don't need to run the command with sudo .

```bash
sudo usermod -aG docker $USER
newgrp docker
```

Pull vLLM container image
Find the latest container build from https://catalog.ngc.nvidia.com/orgs/nvidia/containers/vllm

```bash
export LATEST_VLLM_VERSION=<latest_container_version>
# example
# export LATEST_VLLM_VERSION=26.02-py3

export HF_MODEL_HANDLE=<HF_HANDLE>
# example
# export HF_MODEL_HANDLE=openai/gpt-oss-20b

docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}
```

ตัวอย่างคำสั่งที่ใช้งาน สำหรับ DGX Spark
```bash
# Set the variables using actual values (no angle brackets)
export LATEST_VLLM_VERSION="26.02-py3"
export HF_MODEL_HANDLE="openai/gpt-oss-20b"

# Pull the container
docker pull nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION}
```

ส่วนผมใช้วิธีการสร้างไฟล์ script สำหรับรันเลย จะได้ ไม่พลาด
วิธีสร้างและใช้งาน Script
1. สร้างไฟล์ script:
พิมพ์คำสั่งนี้ใน Terminal เพื่อสร้างและเปิดไฟล์ใหม่ (เช่นตั้งชื่อว่า start_vllm.sh):
```bash
nano start_vllm.sh
```
2. Copy โค้ดด้านล่างนี้ไปวางในไฟล์:
```bash
#!/bin/bash

# ==========================================
# vLLM Server Configuration Script
# ==========================================

# กำหนดเวอร์ชันของ Container และชื่อ Model
export LATEST_VLLM_VERSION="26.02-py3"
export HF_MODEL_HANDLE="openai/gpt-oss-20b" # เปลี่ยนเป็นชื่อโมเดลที่คุณต้องการใช้งาน

# ใส่ Hugging Face Token ของคุณที่นี่ (เอาเครื่องหมาย <> ออกด้วยนะครับ)
export HF_TOKEN="your_actual_token_here"

echo "=========================================="
echo "Starting vLLM Server..."
echo "Model: $HF_MODEL_HANDLE"
echo "Container: nvcr.io/nvidia/vllm:$LATEST_VLLM_VERSION"
echo "=========================================="

# รัน Docker Container
docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
    -p 8000:8000 \
    --ipc=host \
    nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
    vllm serve ${HF_MODEL_HANDLE} \
    --host 0.0.0.0 \
    --port 8000
```
3. บันทึกไฟล์:<br>
  กด Ctrl + O (เพื่อเซฟ)<br>
  กด Enter (เพื่อยืนยันชื่อไฟล์)<br>
  กด Ctrl + X (เพื่อออกจาก nano)<br>

4. กำหนดสิทธิ์ให้ไฟล์สามารถรันได้ (Executable):
```bash
chmod +x start_vllm.sh
```

5. สั่งรัน Script:
```bash
./start_vllm.sh
```
---
การใช้งาน vLLM ผ่าน Docker ร่วมกับการ Mount Volume
```bash
-v ~/.cache/huggingface:/root/.cache/huggingface \
```
บรรทัดนี้ทำหน้าที่ "เชื่อมต่อ" (Mount) โฟลเดอร์แคชของเครื่อง DGX Spark (Host) เข้ากับโฟลเดอร์แคชภายใน Container

กระบวนการทำงานจะเป็นแบบนี้ครับ:

รันครั้งแรก: เมื่อคุณรัน Script นี้ครั้งแรก vLLM จะตรวจสอบในโฟลเดอร์ /root/.cache/huggingface (ซึ่งเชื่อมกับ ~/.cache/huggingface ในเครื่องคุณ) ว่ามีไฟล์ของโมเดล openai/gpt-oss-20b (หรือโมเดลที่คุณระบุ) อยู่หรือเปล่า ถ้าไม่มี มันจะเริ่มทำการดาวน์โหลดโมเดลจาก Hugging Face ลงมาเก็บไว้ในโฟลเดอร์นี้

รันครั้งต่อไป: เมื่อคุณปิด Container และรัน Script นี้ใหม่อีกครั้ง vLLM จะเข้าไปเช็คที่โฟลเดอร์เดิม และจะพบว่าไฟล์โมเดลถูกดาวน์โหลดมาเก็บไว้เรียบร้อยแล้ว มันจึง ข้ามขั้นตอนการดาวน์โหลด และทำการโหลดโมเดลเข้าสู่ GPU (VRAM) เพื่อพร้อมใช้งานได้ทันที ซึ่งจะใช้เวลาแค่ไม่กี่วินาทีหรือหลักนาที (ขึ้นอยู่กับขนาดโมเดลและความเร็วของ Disk/GPU)

สรุปคือ:

ดาวน์โหลดเฉพาะครั้งแรกที่เรียกใช้โมเดลนั้นๆ

ครั้งต่อไปจะโหลดจาก Cache ในเครื่อง (Disk) ขึ้นไปบน GPU เลย ประหยัดเวลาและแบนด์วิดท์อินเทอร์เน็ตได้มากครับ<br>

Cleanup and rollback<br>
For container approach (non-destructive):<br>
```bash
docker rm $(docker ps -aq --filter ancestor=nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION})
docker rmi nvcr.io/nvidia/vllm
```

เมื่อหน้าจอฝั่งที่รันเซิร์ฟเวอร์ขึ้นข้อความว่า Uvicorn running on http://0.0.0.0:8000 (หรือคล้ายๆ กัน) แปลว่าเซิร์ฟเวอร์พร้อมรับคำสั่งแล้ว

---
  คุณสามารถเปิด Terminal หน้าต่างใหม่ขึ้นมาอีกอัน (อย่าปิดหน้าต่างที่รันเซิร์ฟเวอร์ค้างไว้นะครับ) แล้วใช้คำสั่งเพื่อทดสอบได้เลย เนื่องจาก vLLM ออกแบบมาให้เข้ากันได้กับโครงสร้างของ OpenAI API (OpenAI-compatible) เราจึงสามารถทดสอบได้ 2 วิธีง่ายๆ ดังนี้ครับ:


1. ทดสอบด้วยคำสั่ง curl (พิมพ์ใน Terminal)<br>
คำสั่งนี้จะเป็นการจำลองการส่งข้อมูลแบบ JSON ไปหาเซิร์ฟเวอร์โดยตรงครับ ก๊อปปี้ไปวางแล้วกด Enter ได้เลย:
```bash
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "openai/gpt-oss-20b",
    "messages": [
      {"role": "system", "content": "You are a helpful cybersecurity expert."},
      {"role": "user", "content": "Hello! Tell me a very short joke about hackers."}
    ],
    "max_tokens": 100,
    "temperature": 0.7
  }'
```

2. ทดสอบด้วยสคริปต์ Python (แนะนำสำหรับการเอาไปต่อยอด)
สำหรับการเอาไปใช้กับงานวิจัย ป.โท หรือระบบ Log Analysis ของคุณในอนาคต คุณสามารถใช้ Library openai มาตรฐานของ Python ต่อเข้ากับ vLLM ในเครื่องตัวเองได้เลยครับ

ถ้าจะเทสด้วยวิธีนี้ ให้ลง library ก่อน (pip install openai) แล้วรันโค้ดนี้ครับ:

```bash
from openai import OpenAI

# ชี้ API URL ไปที่ Localhost ของเครื่องเรา (พอร์ต 8000)
# vLLM ปกติถ้าไม่ตั้งค่าบังคับรหัสผ่าน สามารถใส่ api_key เป็นอะไรก็ได้ เช่น "EMPTY"
client = OpenAI(
    base_url="http://localhost:8000/v1",
    api_key="EMPTY", 
)

response = client.chat.completions.create(
    model="openai/gpt-oss-20b", # เปลี่ยนเป็นชื่อโมเดลที่คุณรันอยู่
    messages=[
        {"role": "system", "content": "You are a security expert."},
        {"role": "user", "content": "Explain what a firewall is in one simple sentence."},
    ]
)

print(response.choices[0].message.content)
```
