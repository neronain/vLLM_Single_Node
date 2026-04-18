````markdown
# การใช้งาน vLLM บน DGX Spark

ในที่นี้จะเป็นแนวทางการใช้งาน **vLLM** บน **DGX Spark** ที่สรุปจากประสบการณ์ของฉันเพื่อให้ง่ายต่อการเข้าใจ โดยอ้างอิงจากแหล่งข้อมูลต่อไปนี้:
- [NVIDIA vLLM on DGX Spark](https://build.nvidia.com/spark/vllm/instructions)
- [Hugging Face](https://huggingface.co/)

vLLM คือ Inference Engine ที่รองรับการประมวลผลขนาดใหญ่โดยใช้ Ray เพื่อจัดการคลัสเตอร์ของเครื่องและ GPU อย่างมีประสิทธิภาพ. โดยระบบนี้สามารถใช้งานได้ทั้งในโหมด **single-node** หรือ **multi-node dual Spark**.

## 1. การติดตั้ง

### 1.1 **เตรียมสภาพแวดล้อม**

- ติดตั้ง **Docker** และ **NVIDIA Container Toolkit**.
- ดาวน์โหลด **vLLM Docker Image**:
  ```bash
  docker pull nvcr.io/nvidia/vllm:25.11-py3
````

### 1.2 **การตั้งค่า Spark Cluster**

หากคุณต้องการตั้งค่า **multi-node**:

* เชื่อมต่อสองเครื่องให้พร้อมใช้ผ่าน **QSFP** (200Gb/s) ในการเชื่อมต่อ **Ray** cluster.

![Stacked Spark Setup](images/stacked_spark_setup.png)

### 1.3 **เริ่มต้น Docker Container**

* รัน **vLLM** container:

  ```bash
  docker run --gpus all -it --rm -v ~/.cache/huggingface:/mnt/models -p 8000:8000 --name vllm-container nvcr.io/nvidia/vllm:25.11-py3
  ```

## 2. การใช้งาน vLLM

### 2.1 **เริ่มต้นโมเดล**

* รัน **Llama-3.3-70B-Instruct** ใน **vLLM**:

  ```bash
  docker exec vllm-container \
    vllm serve meta-llama/Llama-3.3-70B-Instruct --tensor-parallel-size 2 --max_model_len 2048
  ```

### 2.2 **ทดสอบการใช้งาน API**

* ทดสอบด้วยคำสั่ง `curl`:

  ```bash
  curl -X POST http://localhost:8000/v1/chat/completions \
    -H "Content-Type: application/json" \
    -d '{
      "model": "meta-llama/Llama-3.3-70B-Instruct",
      "messages": [
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "What is the capital of France?"}
      ],
      "max_tokens": 64,
      "temperature": 0.7
    }'
  ```

## 3. การตั้งค่าคลัสเตอร์ Multi-Node Dual Spark

### 3.1 **ตั้งค่าเครื่องแรก (Head Node)**

* ใช้คำสั่งเพื่อเริ่มต้น **Ray** และ **vLLM** บนเครื่องแรก:

  ```bash
  export MN_IF_NAME=enp1s0f1np1
  export VLLM_HOST_IP=$(ip -4 addr show $MN_IF_NAME | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
  bash run_cluster.sh $VLLM_IMAGE $VLLM_HOST_IP --head ~/.cache/huggingface \
    -e VLLM_HOST_IP=$VLLM_HOST_IP -e UCX_NET_DEVICES=$MN_IF_NAME -e NCCL_SOCKET_IFNAME=$MN_IF_NAME
  ```

### 3.2 **เครื่องที่สอง (Worker Node)**

* บนเครื่องที่สอง:

  ```bash
  export MN_IF_NAME=enp1s0f1np1
  export HEAD_NODE_IP=<IP ของ Node 1>
  bash run_cluster.sh $VLLM_IMAGE $HEAD_NODE_IP --worker ~/.cache/huggingface \
    -e VLLM_HOST_IP=$MN_IF_NAME -e NCCL_SOCKET_IFNAME=$MN_IF_NAME
  ```

### 3.3 **ตรวจสอบสถานะของ Ray Cluster**

* ตรวจสอบสถานะของคลัสเตอร์ Ray:

  ```bash
  docker exec $VLLM_CONTAINER ray status
  ```

## 4. ปัญหาที่พบและการแก้ไข

### 4.1 **ปัญหาการคอมไพล์ `tokenizers`**

* หากพบปัญหาเกี่ยวกับการคอมไพล์ Rust, ให้ติดตั้ง **Rust** และ **Cargo** ด้วยคำสั่ง:

  ```bash
  curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh
  ```

### 4.2 **การดาวน์โหลดโมเดลซ้ำทุกครั้ง**

* กำหนดเส้นทางการเก็บโมเดลด้วยตัวแปร `MODEL_CACHE_DIR`:

  ```bash
  export MODEL_CACHE_DIR=/path/to/your/cache/dir
  ```

![Model Cache Example](images/model_cache_example.png)

## 5. อ้างอิง

* ข้อมูลจาก [NVIDIA vLLM Instructions](https://build.nvidia.com/spark/vllm/instructions)
* โมเดลจาก [Hugging Face](https://huggingface.co/)

---
