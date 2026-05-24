---
title: Day 12 — Hạ tầng Cloud & Deployment
sidebar_position: 12
---

# Day 12 — Hạ tầng Cloud & Deployment

> **Mục tiêu**: Chọn hạ tầng cloud phù hợp cho AI app, hiểu serverless vs containers, deploy bằng Docker + Kubernetes, scale & secure đúng cách.

## 1. Lựa chọn hạ tầng — Cấp độ cơ bản

### 1.1 Trục quyết định

```
                  Compute
                     │
        ┌────────────┼────────────┐
    Serverless    Container      VM/Bare-metal
    Vercel        ECS, Fly        EC2, GPU instance
    Lambda        Cloud Run       on-prem
    Modal         K8s             specialized
        │           │               │
    1-5 req/s   100+ req/s      train/fine-tune
    nhỏ gọn     full control    cần GPU
```

### 1.2 Cloud providers cho AI

| Provider | Mạnh ở |
|----------|--------|
| **AWS** | Bedrock (Claude + nhiều model), SageMaker, đầy đủ nhất |
| **GCP** | Vertex AI (Gemini), TPU, BigQuery + Vector |
| **Azure** | OpenAI service exclusive, enterprise-friendly |
| **Anthropic API direct** | Claude, đơn giản nhất |
| **OpenAI API direct** | GPT models |
| **Replicate / Modal** | Run open-source model dễ |
| **RunPod / Lambda Labs** | GPU rẻ cho fine-tune |
| **Fly.io / Railway** | Easy container deploy |

### 1.3 Nguyên tắc chọn

- **MVP**: dùng API direct (Anthropic/OpenAI) + Vercel/Railway. Đừng tự host GPU.
- **Enterprise**: AWS Bedrock / Azure OpenAI vì compliance, VPC, SSO.
- **Cần on-prem**: Llama/Mistral + vLLM trên GPU server.
- **Cost-sensitive scale**: optimize prompt cache, fine-tune nhỏ thay vì gọi API.

### Bài tập 12.1

Cho app: chatbot nội bộ công ty, 500 nhân viên, ~50 query/giờ peak, data nhạy cảm phải on VN. Đề xuất hạ tầng. Vì sao?

---

## 2. Deployment cơ bản với Docker — Cấp độ trung cấp

### 2.1 Dockerfile cho FastAPI + Anthropic

```dockerfile
# Dockerfile
FROM python:3.12-slim

WORKDIR /app

# Cài đặt deps trước (cache layer)
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy code
COPY . .

# Non-root user
RUN useradd -m appuser
USER appuser

ENV PORT=8000
EXPOSE 8000

CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```python
# main.py
from fastapi import FastAPI
from pydantic import BaseModel
from anthropic import Anthropic

app = FastAPI()
client = Anthropic()

class Q(BaseModel):
    question: str

@app.post("/chat")
def chat(q: Q):
    resp = client.messages.create(
        model="claude-sonnet-4-6",
        max_tokens=1024,
        messages=[{"role": "user", "content": q.question}],
    )
    return {"answer": resp.content[0].text}

@app.get("/health")
def health():
    return {"status": "ok"}
```

Build & run:

```bash
docker build -t my-ai-app .
docker run -p 8000:8000 -e ANTHROPIC_API_KEY=$KEY my-ai-app
```

### 2.2 Docker Compose (dev local)

```yaml
# docker-compose.yml
services:
  app:
    build: .
    ports: ["8000:8000"]
    env_file: .env
    depends_on: [chroma, redis]

  chroma:
    image: chromadb/chroma:latest
    volumes: ["./chroma_data:/chroma/chroma"]
    ports: ["8001:8000"]

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
```

### 2.3 Deploy lên Fly.io (đơn giản nhất)

```bash
fly launch
fly secrets set ANTHROPIC_API_KEY=sk-...
fly deploy
```

5 phút có URL HTTPS public.

### 2.4 Deploy lên AWS ECS / Cloud Run

```bash
# Cloud Run (GCP)
gcloud run deploy my-ai-app \
  --image gcr.io/my-project/my-ai-app \
  --region asia-southeast1 \
  --allow-unauthenticated \
  --set-env-vars ANTHROPIC_API_KEY=...
```

### Bài tập 12.2

1. Dockerize FastAPI app phần 2.1. Run local OK?
2. Deploy lên Fly.io hoặc Cloud Run. Đo cold start latency.
3. Push image lên Docker Hub / GHCR.

---

## 3. Production Deployment — Cấp độ nâng cao

### 3.1 Kubernetes pattern

Cho app có scale lớn, K8s là chuẩn de facto.

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ai-app
spec:
  replicas: 3
  selector:
    matchLabels: {app: ai-app}
  template:
    metadata:
      labels: {app: ai-app}
    spec:
      containers:
      - name: api
        image: ghcr.io/me/ai-app:v1.2.0
        ports: [{containerPort: 8000}]
        env:
        - name: ANTHROPIC_API_KEY
          valueFrom:
            secretKeyRef: {name: anthropic, key: api-key}
        resources:
          requests: {cpu: 200m, memory: 512Mi}
          limits:   {cpu: 1000m, memory: 1Gi}
        readinessProbe:
          httpGet: {path: /health, port: 8000}
          periodSeconds: 5
        livenessProbe:
          httpGet: {path: /health, port: 8000}
          initialDelaySeconds: 30
---
apiVersion: v1
kind: Service
metadata: {name: ai-app}
spec:
  selector: {app: ai-app}
  ports: [{port: 80, targetPort: 8000}]
  type: ClusterIP
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata: {name: ai-app}
spec:
  scaleTargetRef: {kind: Deployment, name: ai-app}
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource: {name: cpu, target: {type: Utilization, averageUtilization: 70}}
```

### 3.2 Self-host LLM với vLLM

Nếu cần on-prem hoặc cost optimization cao:

```bash
# GPU server (A100, H100, hoặc consumer 4090)
pip install vllm

vllm serve meta-llama/Llama-3.1-8B-Instruct \
  --tensor-parallel-size 1 \
  --max-model-len 8192 \
  --port 8000
```

API tương thích OpenAI:

```python
from openai import OpenAI
client = OpenAI(base_url="http://gpu-server:8000/v1", api_key="dummy")
```

vLLM dùng **PagedAttention** → throughput cao gấp 10x so với HuggingFace Transformers thông thường.

### 3.3 Architecture pattern điển hình

```
                         ┌─────────────────┐
                         │ CDN / Cloudfront│
                         └────────┬────────┘
                                  │
                         ┌────────▼────────┐
                         │ ALB / Ingress    │
                         └────────┬────────┘
                                  │
                ┌─────────────────┼─────────────────┐
                ▼                 ▼                 ▼
          ┌──────────┐      ┌──────────┐     ┌──────────┐
          │ API pod  │      │ API pod  │     │ API pod  │
          └─────┬────┘      └─────┬────┘     └─────┬────┘
                │                 │                 │
                └────────┬────────┴─────────────────┘
                         ▼
        ┌────────────────┼────────────────┐
        ▼                ▼                ▼
   ┌─────────┐     ┌──────────┐    ┌──────────┐
   │ Redis   │     │ Vector DB│    │ LLM API  │
   │ (cache) │     │ (Qdrant) │    │(Anthropic│
   └─────────┘     └──────────┘    │  /vLLM)  │
                                   └──────────┘
```

### 3.4 Caching & Rate Limit

```python
import redis
import hashlib, json

r = redis.from_url("redis://...")

def cached_chat(question: str, ttl=3600):
    key = "chat:" + hashlib.sha256(question.encode()).hexdigest()
    if cached := r.get(key):
        return json.loads(cached)
    resp = llm_call(question)
    r.setex(key, ttl, json.dumps(resp))
    return resp

# Rate limit: 60 req / phút / user
def rate_limit(user_id):
    key = f"rl:{user_id}:{time.time() // 60:.0f}"
    n = r.incr(key)
    r.expire(key, 60)
    if n > 60:
        raise HTTPException(429, "Too many requests")
```

### 3.5 Streaming responses

LLM trả output từng token → cải thiện perceived latency.

```python
from fastapi.responses import StreamingResponse

@app.post("/chat/stream")
def chat_stream(q: Q):
    def gen():
        with client.messages.stream(
            model="claude-sonnet-4-6",
            max_tokens=1024,
            messages=[{"role": "user", "content": q.question}],
        ) as stream:
            for text in stream.text_stream:
                yield text
    return StreamingResponse(gen(), media_type="text/event-stream")
```

### Bài tập 12.3

1. Viết Dockerfile + K8s manifest cho app. Deploy lên cluster (k3s, kind, hoặc cloud).
2. Thử vLLM với 1 model nhỏ (Llama-3.2-3B). Đo throughput.
3. Add Redis cache, đo cache hit rate sau 100 query.
4. Implement streaming endpoint. So sánh perceived latency vs non-stream.

---

## 4. Security

| Concern | Cách xử lý |
|---------|-----------|
| **API key** | Secret manager (AWS, Vault), không commit |
| **HTTPS** | Always-on, TLS 1.3 |
| **Input size** | Limit max body size (1MB), max tokens |
| **DoS** | Rate limit, WAF, CDN |
| **Data residency** | Choose region phù hợp (VN/SG cho VN user) |
| **VPC** | API LLM qua PrivateLink/private endpoint nếu sensitive |
| **Logs PII** | Mask trước log |

## 5. Multi-region & Failover

Cho app cần SLA cao:

```
Primary region (Singapore)
   ├── Active traffic
   └── Failover to backup if down

Backup region (Tokyo)
   ├── Standby (read-only)
   └── Promoted on disaster

Anthropic API
   └── Multi-region replication, no action needed
```

DNS failover qua Route 53 / Cloudflare.

## 6. CI/CD cho AI App

```yaml
# .github/workflows/deploy.yml
name: deploy
on:
  push: {branches: [main]}
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: pip install -r requirements.txt
      - run: pytest                       # unit test
      - run: python eval/run_eval.py      # LLM eval gate
  deploy:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - run: docker build -t my-app:${{github.sha}} .
      - run: docker push ghcr.io/me/my-app:${{github.sha}}
      - run: kubectl set image deploy/ai-app api=ghcr.io/me/my-app:${{github.sha}}
```

**Quan trọng**: thêm bước **eval gate** — nếu accuracy giảm > X%, block deploy.

---

## 7. Tóm tắt

- MVP: API + Vercel/Railway. Production: Docker + K8s hoặc Cloud Run.
- Self-host LLM với **vLLM** khi cần on-prem hoặc cost optimization.
- Architecture: API + Redis cache + Vector DB + LLM provider.
- Cache, rate limit, stream → trải nghiệm tốt.
- CI/CD có **eval gate** để chặn regression.
- Security: HTTPS, secret manager, rate limit, data residency.

## 8. Bài tập tổng hợp

1. **Full stack**: Dockerize FastAPI + ChromaDB + Redis. Deploy lên 1 cloud miễn phí (Fly.io / Cloud Run free tier). URL public.
2. **K8s exercise**: Tạo cluster local (k3s/kind). Deploy với 3 replica, HPA, Service. Curl test.
3. **vLLM**: Thuê 1 GPU spot (RunPod $0.4/h), serve Llama-3.2-3B với vLLM. Benchmark throughput.
4. **Cost calc**: Cho app 100K request/tháng, mỗi request ~2K input + 500 output token. Tính chi phí cho: (a) Claude Sonnet, (b) GPT-4.1 mini, (c) self-host vLLM Llama-3.1-8B. So sánh.
5. **Đọc**:
   - [vLLM docs](https://docs.vllm.ai/)
   - [AWS Bedrock](https://aws.amazon.com/bedrock/)
   - [Cloud Run + Anthropic guide](https://cloud.google.com/blog)

---

> Hết Day 12. Mai sẽ học **Monitoring & LLMOps** — phần đảm bảo hệ thống không "lén lút" hư.
