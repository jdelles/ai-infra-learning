# ai-infra-learning

Going deeper on AI infrastructure — the parts of model serving, MLOps, and GPU systems I want to understand at a lower level than day-to-day work requires. Notes, experiments, and benchmarks as I work through it.

---

## Roadmap

**MLOps**
- [ ] ML pipelines: feature stores, training orchestration, model registry
- [ ] Experiment tracking (MLflow, W&B)
- [ ] CI/CD for ML: testing models, data validation, deployment gates
- [ ] Monitoring: drift detection, logging predictions, alerting

**Model Serving**
- [ ] Serving patterns: batch vs. real-time, synchronous vs. async
- [ ] ONNX and model optimization
- [ ] Triton Inference Server deep dive
- [ ] vLLM: architecture, continuous batching, PagedAttention
- [ ] Benchmarking: latency, throughput, cost

**Kubernetes & Cloud-Native ML**
- [ ] KServe / Seldon for model serving on K8s
- [ ] GPU node management: taints, tolerations, resource quotas
- [ ] Kubeflow Pipelines
- [ ] Ray Serve and distributed inference

**GPU & Distributed Systems**
- [ ] CUDA fundamentals
- [ ] Mixed precision training (FP16, BF16)
- [ ] Distributed training: DDP, FSDP, tensor parallelism

---

## Notes

Organized by topic in `notes/` — mlops, model-serving, kubernetes, gpu-systems, distributed-training.

## Experiments

Hands-on exploration in `experiments/`. Setup notes, benchmarks, and takeaways.

---

**Resources:** [Made With ML](https://madewithml.com) · [Chip Huyen](https://huyenchip.com) · [vLLM docs](https://docs.vllm.ai) · *Designing ML Systems* — Huyen
