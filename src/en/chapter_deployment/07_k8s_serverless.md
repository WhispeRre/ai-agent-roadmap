# 19.7 Kubernetes Orchestration and Serverless GPU

> **Goal**: Learn how to orchestrate a complete Agent service stack with Kubernetes, use Serverless GPU platforms such as Modal and RunPod, and understand autoscaling strategies for GPU workloads.

---

## Why Kubernetes for Agent Systems?

A production Agent system is usually not a single service. It contains API servers, workers, vector databases, caches, model-serving backends, observability components, and task queues.

Kubernetes helps manage:

- service discovery;
- rolling updates;
- autoscaling;
- health checks;
- resource isolation;
- secrets and configuration;
- GPU scheduling;
- multi-service orchestration.

---

## Agent Service Architecture on Kubernetes

```text
Ingress / API Gateway
        ↓
Agent API Service
        ↓
Task Queue ───────────────┐
        ↓                  │
Agent Worker Pods          │
        ↓                  │
Tool Services / RAG Index  │
        ↓                  │
Model Gateway ─────────────┘
        ↓
GPU Model Serving Pods
```

Common Kubernetes components:

| Component | Role |
|----------|------|
| Deployment | Runs API and worker replicas |
| Service | Stable internal network endpoint |
| Ingress | External HTTP routing |
| ConfigMap | Non-secret configuration |
| Secret | API keys and credentials |
| HPA | CPU / memory / custom metric autoscaling |
| Node pool | Separate CPU and GPU workloads |

---

## Minimal Agent API Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: agent-api
spec:
  replicas: 3
  selector:
    matchLabels:
      app: agent-api
  template:
    metadata:
      labels:
        app: agent-api
    spec:
      containers:
        - name: agent-api
          image: agent-learning/agent-api:latest
          ports:
            - containerPort: 8000
          envFrom:
            - secretRef:
                name: agent-secrets
          resources:
            requests:
              cpu: "500m"
              memory: "1Gi"
            limits:
              cpu: "2"
              memory: "4Gi"
---
apiVersion: v1
kind: Service
metadata:
  name: agent-api
spec:
  selector:
    app: agent-api
  ports:
    - port: 80
      targetPort: 8000
```

This is only the API layer. Long-running Agent work should usually be moved to worker pods through a queue.

---

## GPU Workload Scheduling

GPU model-serving pods require explicit GPU resources:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: llm-serving
spec:
  replicas: 1
  selector:
    matchLabels:
      app: llm-serving
  template:
    metadata:
      labels:
        app: llm-serving
    spec:
      nodeSelector:
        accelerator: nvidia
      containers:
        - name: vllm
          image: vllm/vllm-openai:latest
          args:
            - --model
            - meta-llama/Llama-3.1-8B-Instruct
          resources:
            limits:
              nvidia.com/gpu: 1
```

In production, use separate node pools for CPU and GPU workloads, and protect GPU capacity with quotas and autoscaling rules.

---

## Autoscaling Strategy

For Agent workloads, CPU usage alone is often a poor scaling signal. Better signals include:

- request queue length;
- average waiting time;
- tokens generated per second;
- GPU utilization;
- number of active conversations;
- model-serving queue depth.

```text
Queue length increases
      ↓
Scale Agent worker pods
      ↓
Model queue increases
      ↓
Scale GPU serving replicas or route to hosted fallback
```

For Kubernetes, combine HPA with custom metrics from Prometheus or queue systems.

---

## Serverless GPU

Serverless GPU platforms are useful when workloads are bursty or experimental.

| Platform | Strength | Typical Use |
|---------|----------|-------------|
| Modal | Developer-friendly Python workflows | Batch inference, experiments, small services |
| RunPod | Flexible GPU instances and serverless endpoints | Model serving, image/video workloads |
| Replicate | Simple hosted model APIs | Product prototypes and public model inference |

Serverless GPU advantages:

- no cluster management;
- scale to zero;
- fast experimentation;
- pay-per-use pricing.

Limitations:

- cold start latency;
- less control over networking and storage;
- possible vendor lock-in;
- harder to optimize custom serving stacks.

---

## Hybrid Deployment Pattern

Many teams use a hybrid approach:

```text
Stable traffic → Kubernetes services
Burst traffic  → Serverless GPU fallback
Offline jobs   → Serverless batch workers
High-risk tasks → Dedicated controlled environment
```

This combines the reliability of Kubernetes with the elasticity of serverless platforms.

---

## Best Practices

- Keep API services stateless; store conversation state externally.
- Move long Agent tasks to worker queues.
- Separate CPU and GPU node pools.
- Use custom metrics for autoscaling instead of CPU-only scaling.
- Add graceful shutdown so long Agent runs can finish or checkpoint.
- Use serverless GPU for bursty workloads, experiments, and batch jobs.
- Monitor cost per deployment, per model, and per task category.

---

## Chapter Takeaways

- Kubernetes is useful for orchestrating the full Agent service stack, not only model serving.
- GPU workloads require dedicated scheduling, quotas, and monitoring.
- Autoscaling should be based on queue depth, latency, tokens/sec, and GPU metrics.
- Serverless GPU platforms are excellent for bursty or experimental workloads but may introduce cold starts.
- Hybrid deployment often gives the best balance between control and elasticity.

---

*Previous: [19.6 Model Serving for LLM and Agent Systems](./06_model_serving.md)*  
*Next: [19.8 Long-Running Task Queues and Cost Governance](./08_task_queue_cost.md)*