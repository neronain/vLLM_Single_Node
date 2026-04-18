โมเดล **`Gemma-4-26B-A4B-it-Uncensored-NVFP4`** ทดสอบ Model Security Research 

* **ปลดล็อกข้อจำกัดด้วย Uncensored:** นี่คือจุดแข็งที่สุดสำหรับสาย Security ครับ โมเดลปกติ (รวมถึง Qwen หรือ Llama ตัวมาตรฐาน) จะมีการฝังระบบ Safety Guardrails ไว้ ถ้าคุณป้อน Log ที่มีลักษณะของการโจมตี, ขอให้วิเคราะห์ Malware, หรือขอให้เขียน Script สำรวจช่องโหว่ (Red Teaming) โมเดลปกติมักจะปฏิเสธการตอบ (Refusal) แต่โมเดล Uncensored จะข้ามข้อจำกัดนี้และยอมวิเคราะห์ข้อมูลให้คุณอย่างตรงไปตรงมา
* **ขนาด 26B (Sweet Spot):** เป็นขนาดที่ฉลาดพอที่จะเข้าใจตรรกะของ Code และ Network Protocol ที่ซับซ้อน แต่ก็ไม่ได้ใหญ่จนประมวลผลช้าเกินไป
* **เทคโนโลยี NVFP4 (NVIDIA FP4):** การบีบอัดระดับ 4-bit แบบ Native ของ NVIDIA ถือว่าล้ำหน้ามาก มันจะรีดประสิทธิภาพของ GPU บน DGX Spark ได้สูงสุด โมเดลระดับ 26B ปกติอาจจะกิน VRAM ราวๆ 50-60GB แต่พอเป็น NVFP4 มันจะใช้ VRAM ลดลงเหลือเพียง **~14-16GB เท่านั้น** ทำให้คุณสามารถโยน Context ยาวๆ (เช่น Log ไฟล์ขนาดใหญ่) เข้าไปวิเคราะห์ได้โดยที่ VRAM ยังเหลือเฟือ

---

### ⚙️ วิธีอัปเดต Script เพื่อรันโมเดลนี้

คุณสามารถใช้ Script `start_vllm.sh` ตัวเดิมที่ผมเขียนให้ได้เลยครับ แค่เปลี่ยนตัวแปรชื่อโมเดล และเพิ่มพารามิเตอร์เพื่อให้ vLLM รู้จักการบีบอัดแบบ NVFP4 (โดยปกติ vLLM จะ Auto-detect ได้ แต่ระบุไว้จะชัวร์กว่าครับ)

เข้าไปแก้ไฟล์ `start_vllm.sh` ตามนี้ครับ:

```bash
export HF_MODEL_HANDLE="AEON-7/Gemma-4-26B-A4B-it-Uncensored-NVFP4"
```

และในส่วนของคำสั่งรัน `vllm serve` ด้านล่าง ให้เพิ่มพารามิเตอร์เหล่านี้เข้าไปครับ:

```bash
    vllm serve ${HF_MODEL_HANDLE} \
    --host 0.0.0.0 \
    --port 8000 \
    --max-model-len 32768 \
    --trust-remote-code
```
*(หมายเหตุ: `--trust-remote-code` มักจะจำเป็นสำหรับโมเดล Custom หรือโมเดลที่มีการทำ Quantization รูปแบบใหม่ๆ เพื่อให้ vLLM โหลด Custom Operations ขึ้นมาทำงานได้)*

---

### ⚠️ ข้อควรระวังในการใช้งาน (Security Note)
เนื่องจากโมเดลนี้เป็น **Uncensored** มันจะไม่มีจริยธรรมมาคอยเบรก หากคุณให้มันช่วยเขียนโค้ดเพื่อทดสอบช่องโหว่ (PoC Exploit) มันจะเขียนให้จริงๆ ซึ่งโค้ดนั้นสามารถทำงานและสร้างความเสียหายได้จริง การที่เรารันทุกอย่างใน Docker Container ถือเป็นการจำกัดขอบเขต (Sandboxing) ที่ดีมากในระดับหนึ่งแล้วครับ

เปิดไฟล์ `start_gemma_sec.sh` ขึ้นมาแก้จุดเดิมอีกครั้ง:
```bash
nano start_gemma_sec.sh
```

แล้วแก้ตรงบรรทัด `-c` ให้เป็นแบบนี้ครับ (เพิ่ม `--upgrade mistral_common` เข้าไป):

```bash
# รัน Docker Container พร้อมตั้งค่าพารามิเตอร์สำหรับ NVFP4 และสั่งอัปเดตไลบรารี
docker run --gpus all \
    -v ~/.cache/huggingface:/root/.cache/huggingface \
    --env "HUGGING_FACE_HUB_TOKEN=$HF_TOKEN" \
    -p 8000:8000 \
    --ipc=host \
    --entrypoint bash \
    nvcr.io/nvidia/vllm:${LATEST_VLLM_VERSION} \
    -c "pip install --upgrade mistral_common git+https://github.com/huggingface/transformers.git && vllm serve ${HF_MODEL_HANDLE} --host 0.0.0.0 --port 8000 --trust-remote-code --max-model-len 8192 --gpu-memory-utilization 0.9"
```

เซฟไฟล์แล้วรันสคริปต์ `./start_gemma_sec.sh` ใหม่อีกรอบได้เลยครับ

