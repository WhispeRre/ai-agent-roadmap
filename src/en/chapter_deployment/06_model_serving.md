# 19.6 Model Serving for LLM and Agent Systems

> **Goal**: Master the three major model-serving frameworks (vLLM, SGLang, and TGI), understand mainstream quantization options (GPTQ, AWQ, GGUF), and design routing strategies that balance cost, latency, and quality.

---

## Why Model Serving Matters

Calling hosted APIs is the fastest way to build an Agent prototype. But as traffic grows, teams often need self-hosted or dedicated model serving for:

- lower latency;
- predictable cost;
- private data constraints;
- custom fine-tuned models;
- control over batching and GPU utilization;
- fallback when external providers are unavailable.

Model serving is not just "run a model on a GPU". It is an engineering system that handles token streaming, batching, caching, quantization, scaling, and monitoring.

---

## Serving Architecture

```text
Client / Agent Runtime
        ↓
API Gateway
        ↓
Model Router
        ↓
Serving Backend
  ├── vLLM
  ├── SGLang
  └── TGI
        ↓
GPU Workers
        ↓
Metrics / Logs / Traces
```

For Agent workloads, model serving must support long contexts, tool-call formatting, streaming, and sometimes high request burstiness.

---

## Framework Comparison

| Framework | Strength | Best For |
|----------|----------|----------|
| vLLM | High-throughput serving with PagedAttention | General LLM inference and OpenAI-compatible APIs |
| SGLang | Fast structured generation and complex prompting | Agent workflows, function calling, multi-step programs |
| TGI | Mature Hugging Face ecosystem integration | Hugging Face model deployment and enterprise serving |

### vLLM

vLLM is widely used because it provides high throughput and an OpenAI-compatible server.

```bash
python -m vllm.entrypoints.openai.api_server \
  --model meta-llama/Llama-3.1-8B-Instruct \
  --host 0.0.0.0 \
  --port 8000
```

The OpenAI-compatible interface allows many Agent frameworks to switch from hosted APIs to self-hosted inference with minimal code changes.

### SGLang

SGLang is optimized for structured generation, multi-call workflows, and efficient inference programs. It is useful when Agent behavior has repeated prompt patterns or structured tool-call outputs.

### TGI

Text Generation Inference (TGI) integrates well with Hugging Face models and deployment workflows. It is often chosen by teams already using the Hugging Face ecosystem.

---

## Quantization Options

Quantization reduces memory usage and can lower serving cost.

| Method | Typical Use | Trade-off |
|-------|-------------|-----------|
| GPTQ | Post-training quantization for GPU inference | Good compression, setup complexity |
| AWQ | Activation-aware weight quantization | Strong quality retention for 4-bit models |
| GGUF | llama.cpp ecosystem | Excellent for CPU / edge / local inference |
| FP8 | Modern GPU serving | Strong throughput on supported hardware |

Rule of thumb:

- use full precision or FP8 for high-quality production serving;
- use AWQ / GPTQ when GPU memory is constrained;
- use GGUF for local, edge, or CPU-friendly deployments.

---

## Model Routing

A production Agent rarely needs one model for all requests.

```python
class ServingRouter:
    def route(self, request: dict) -> str:
        task_type = request.get("task_type", "chat")
        input_tokens = request.get("input_tokens", 0)
        risk = request.get("risk", "low")

        if risk == "high":
            return "gpt-4.1-or-strong-internal-model"
        if task_type in {"classification", "rewrite", "extract"}:
            return "small-fast-model"
        if input_tokens > 16_000:
            return "long-context-model"
        return "general-agent-model"
```

Routing should consider:

- task complexity;
- input length;
- tool-use requirements;
- latency target;
- user tier;
- risk level;
- fallback policy.

---

## Serving Metrics

Track metrics at both infrastructure and Agent levels:

| Metric | Why It Matters |
|-------|----------------|
| Tokens per second | Throughput and user experience |
| Time to first token | Streaming responsiveness |
| Queue wait time | Batch pressure and capacity shortage |
| GPU utilization | Cost efficiency |
| KV cache hit / memory usage | Long-context performance |
| Error rate | Reliability |
| Cost per request | Business sustainability |
| Quality score | Routing and model upgrade safety |

---

## Best Practices

- Start with hosted APIs unless there is a clear need for self-hosting.
- Use OpenAI-compatible endpoints to simplify integration with Agent frameworks.
- Benchmark with your real prompts and context lengths, not only synthetic tests.
- Monitor time to first token separately from total latency.
- Use quantization only after measuring quality impact.
- Add fallback to hosted providers or alternative models for outages.
- Treat model serving as part of the Agent runtime, not an isolated infrastructure component.

---

## Chapter Takeaways

- Model serving provides cost, latency, privacy, and customization control for production Agents.
- vLLM, SGLang, and TGI are the main serving options, each with different strengths.
- Quantization reduces cost but must be evaluated for quality impact.
- Model routing is essential for balancing cost, quality, latency, and risk.
- Production serving requires monitoring of throughput, latency, GPU utilization, errors, and quality.

---

*Previous: [19.5 Practice: Production Agent Service](./05_practice_production_agent.md)*  
*Next: [19.7 Kubernetes Orchestration and Serverless GPU](./07_k8s_serverless.md)*