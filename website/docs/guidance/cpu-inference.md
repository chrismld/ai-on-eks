---
sidebar_label: CPU Inference on EKS
mermaid: true
---

import Admonition from '@theme/Admonition';

# CPU Inference and Orchestration on EKS

## TL;DR

CPU instances are a first-class compute option for a wide range of AI workloads on Amazon EKS. From small language models (SLMs) and classical ML inference to data pipelines and agent orchestration, CPUs offer strong price-performance, broad capacity availability, and familiar Kubernetes scheduling semantics. A deliberate CPU-first strategy lets you build resilient, cost-efficient AI architectures without sacrificing quality where it matters most.

Production AI pipelines in 2026 are heterogeneous. Not every workload needs a GPU — most of the **work** in a real AI pipeline doesn’t require one. Routing, classification, retrieval, embedding, orchestration, and a growing share of language model inference all run effectively on CPU.

AWS Graviton4 has made the economics of this approach compelling. With up to **50% better price-performance** for ML inference compared to equivalent x86 instances, Graviton is the default benchmark for cost-conscious workloads. Add Karpenter’s node consolidation, KEDA’s event-driven scaling, and quantized model serving via ‘llama.cpp’, and you have a production-ready stack that platform teams can operate without deep GPU expertise.

**This guide is for:**

* **Platform engineers** designing multi-tenant EKS clusters for AI workloads.
* **ML practitioners** evaluating inference backends for models under 30B parameters.
* **FinOps** teams looking for concrete cost levers without sacrificing SLOs.


**What you’ll learn:**

* Understand which AI workloads belong on CPUs and where GPUs or Trainium are genuinely necessary.
* How to apply a four-dimensional decision framework to any new workload.
* Two production patterns with CPUs: agentiz SLM pre-filtering and high-density model farms.
* Advanced optimizations: quantization, bin-packing, Spot scheduling, and observability.

**Key caveat:**

<Admonition type="caution" title="Benchmark Before You Commit">
Every recommendation in this guide includes a benchmark reminder. The right instance family (Graviton, x86, GPU, or Trainium) depends on your model, data, and latency budget. Use this guide as an informed starting point, then validate empirically.
</Admonition>

## Why CPUs for AI Workloads?

Architecture conversations on EKS often default to GPU-first thinking. Sometimes that’s appropriate. Often, it isn’t.  The reality is that production AI pipelines are heterogeneous: CPUs anchor the control plane and much of the inference load, while GPUs and accelerators handle the compute-intensive peaks.

Three things make the CPU argument compelling right now: 

**Capacity availability.** GPU instances frequently require capacity reservations weeks in advance. CPU instances (especially Graviton) are broadly available across all AWS regions with no specialized device plugins, no DRA configuration, and no MIG partitioning. When you need to scale fast, CPU is the fastest lever to pull.

**Economics.** AWS Graviton4 delivers up to 50% better price-performance for ML inference compared to equivalent x86 instances. For teams running FinOps reviews or managing multi-tenant clusters, that delta is material. A Graviton migration can deliver 40-60% cost reductions for suitable workloads, with lower energy consumption as a bonus.

**Operational simplicity.** CPU pods use standard Kubernetes scheduling (`requests`, `limits`, node affinity, topology spread). No device plugins, no custom schedulers, no `nvidia.com/gpu` resource types. Teams that want to run AI workloads without deep GPU expertise can get to production faster on CPU.

## CPU vs GPU / Trainium: When to Choose Each

|Factor	|Choose CPU (Graviton / x86)	|Choose GPU/Trainium	|
|---	|---	|---	|
|Model Size	|SLMs 1-8B (quantized); embeddings; classifiers	|8B+ for low-latency online inference; 70B+ in general	|
|Latency SLO	|p95 100-500ms acceptable	|p95 < 50 ms required	|
|Concurrency	|< 100 req/s per endpoint	|> 100 req/s sustained	|
|Workload Type	|Orchestration, retrieval, ETL, batch scoring	|Online inference, fine-tuning, training	|
|Capacity	|Immediate availability, no reservations	|Often requires reserved capacity	|
|Cost Sensitivity	|Graviton delivers best $/token for eligible workloads	|GPU amortizes at high utilization	|
|Team Expertise	|Standard Kubernetes operatons	|Requires GPU operations knowledge	|

<Admonition type="tip" title="Benchmark Reminder">
These thresholds are starting points. Run `llama.cpp` on a c7g or r7g instance with your actual model and traffic pattern. Graviton4 can surprise you, especially for 7B quantized models.
</Admonition>

## Workload Decision Framework

Choosing the right compute for an AI workload comes down to four dimensions:

1. **Model size and precision**: Does quantization keep quality within your acceptable range?
2. **Latency and throughput SLOs**: What are your p50/p95 targets and peak requests rates?
3. **Workload Types:** Online inference, batch scoring, retrieval or orchestration?
4. **Cost and Capacity constraints:** FinOps budget, regional availability, reservation strategy?

The following flowchart captures the decision logic. Start from your model size and follow the path:

```mermaid
flowchart TD
    Start["🤔 New AI Workload"] --> ModelSize{"Model Size?"}

    ModelSize -->|"1–8B (quantized)"| LatencySLM{"p95 Latency SLO?"}
    ModelSize -->|"8–30B"| LatencyMed{"Workload Type?"}
    ModelSize -->|"70B+"| GPU_Large["🟢 GPU / Trainium<br/>Default for online inference"]

    LatencySLM -->|"100–500ms OK"| CPU_SLM["🔵 CPU (Graviton)<br/>llama.cpp Q4_K_M<br/>Best price-performance"]
    LatencySLM -->|"< 50ms required"| Concurrency{"Concurrency?"}

    Concurrency -->|"< 100 req/s"| CPU_SLM
    Concurrency -->|"> 100 req/s"| GPU_SLM["🟢 GPU / Trainium<br/>High-throughput inference"]

    LatencyMed -->|"Batch / async / offline"| CPU_Med["🔵 CPU (Graviton)<br/>Test Q4 quantization"]
    LatencyMed -->|"Online / long context"| GPU_Med["🟢 GPU / Trainium<br/>Online inference"]

    Start --> Orchestration{"Orchestration /<br/>Retrieval / ETL?"}
    Orchestration -->|"Yes"| CPU_Orch["🔵 CPU (Graviton)<br/>Standard K8s scheduling<br/>No device plugins needed"]

    Start --> Training{"Fine-tuning /<br/>Training?"}
    Training -->|"Data prep"| CPU_Prep["🔵 CPU<br/>Pipeline orchestration"]
    Training -->|"Model training"| GPU_Train["🟢 GPU / Trainium<br/>Training & distillation"]

    style Start fill:#f5f5f5,stroke:#616161,color:#000
    style ModelSize fill:#fff3e0,stroke:#e65100,color:#000
    style LatencySLM fill:#e3f2fd,stroke:#1565c0,color:#000
    style LatencyMed fill:#e3f2fd,stroke:#1565c0,color:#000
    style Concurrency fill:#fce4ec,stroke:#c62828,color:#000
    style Orchestration fill:#fff3e0,stroke:#e65100,color:#000
    style Training fill:#fff3e0,stroke:#e65100,color:#000

    style CPU_SLM fill:#c8e6c9,stroke:#2e7d32,color:#000
    style CPU_Med fill:#c8e6c9,stroke:#2e7d32,color:#000
    style CPU_Orch fill:#c8e6c9,stroke:#2e7d32,color:#000
    style CPU_Prep fill:#c8e6c9,stroke:#2e7d32,color:#000

    style GPU_Large fill:#e1bee7,stroke:#7b1fa2,color:#000
    style GPU_SLM fill:#e1bee7,stroke:#7b1fa2,color:#000
    style GPU_Med fill:#e1bee7,stroke:#7b1fa2,color:#000
    style GPU_Train fill:#e1bee7,stroke:#7b1fa2,color:#000
```

Use the table below as your decision matrix. It reflects practical thresholds.

|Workload	|CPU	|GPU / Trainium	|Notes	|
|---	|---	|---	|---	|
|SLMs (1–8B params, quantized) - Phi-3 Mini, Llama 3.2 3B, Qwen2.5 7B	|Default choice. Strong price-perf at 100–500ms latency, moderate QPS. Graviton especially effective.	|When p95 <50ms or concurrency >100 req/s.	|llama.cpp Q4_K_M or Q8_0 on Graviton4 recommended	|
|Medium models (8–30B params) - Llama 3.1 8B, Mistral 7B	|Batch, async, offline scoring. Test Q4 on Graviton4.	|Online inference, long contexts, tight latency.	|Benchmark Q4 on Graviton4. Results can surprise	|
|Large LLMs (70B+ params) - Llama 3.1 70B	|Non-real-time only, heavy quantization	|Default for production online inference	|Even 70B can run on CPU; expect high latency	|
|Classical ML / Embeddings / CV	|High-density serving; bin-pack across nodes.	|Heavy vision or multi-modal at scale.	|TorchServe, Triton on CPU handles thousands of models.	|
|Data pipelines / ETL / Synthetic data	|Ray and Spark on CPU for data prep and feature engineering.	|N/A	|CPUs anchor this entire data prep stage	|
|Agent orchestration / RAG retrieval	|Network-bound services — API gateways, LiteLLM proxy, retrievers, chunkers.	|N/A	|Graviton's networking throughput shines here.	|
|Fine-tuning / Training	|Data prep and pipeline orchestration.	|Model training and distillation.	|Hybrid: CPU prep → GPU train → CPU infer.	|

### Quick Benchmark Workflow

Before committing to an instance family, run a structured benchmark across the main options. The goal is a single comparable number for each: **cost-per-1,000-queries** at your target p95 latency.

**The four steps:**

1. **Deploy one node per family:** a Graviton node (`r7g.4xlarge`), an x86 node (`c6i.4xlarge`), and a GPU node (`g6.xlarge`) in your test cluster. Keep node count at 1 so costs are directly comparable.
2. **Deploy your model on each node** using `llama.cpp `keep  the same quantization format (Q4_K_M is a good default), the same context size, and the same number of threads as vCPUs. Identical configuration is what makes the comparison valid.
3. **Run a load test against each endpoint.** Tools like `hey` can help you start simple by simulating concurrent users giving you a latency breakdown and throughput summary. Repeat with the correct endpoint for each architecture. `hey` prints p50/p95/p99 latency and requests/sec directly to stdout. No export or parsing needed for a quick comparison.
4. **Compare cost-per-1,000-queries** from the exported summaries. For instance, a Graviton `r7g.4xlarge` at ~$0.84/hr handling 180k requests/hr works out to ~$0.005 per 1,000 requests.

**What to look for:** If Graviton meets your p95 SLO, it wins on cost. If it misses by a small margin, try Graviton4 (`r8g` family) before reaching for GPU. SVE2 support in Graviton4 often closes the gap for quantized 7B models. If latency is still too high at your concurrency target, that's the signal to move the workload to GPU.


<Admonition type="info" title="Keep Test Clusters Short-Lived">
Keep the test cluster running until you have results from all three families. Tear it down the same day — these are single-node clusters and the cost is negligible compared to making the wrong architecture decision at scale.
</Admonition>

## Production Patterns

### Pattern 1: Agentic AI -  SLM Pre-Filter on CPU with LLM Escalation

Most agent workflows execute the same narrow patterns repeatedly: classify the request, pick a tool, extract structured data, validate a response. These tasks don't require a 70B parameter model. 

NVIDIA's research on SLMs demonstrates that models under 10B parameters, when specialized for a domain, can match or exceed large LLMs on constrained sub-tasks, while running efficiently on CPU at significantly lower cost and latency. When a model is fine-tuned for a specific domain, its smaller footprint can actually make it *more accurate and cheaper* than invoking a general-purpose LLM.

**The practical pattern:** An SLM on Graviton handles the majority of requests end-to-end. A routing layer — also running on CPU — escalates only genuinely complex cases to a GPU-hosted LLM. 

```mermaid
flowchart LR
    User["👤 User Request"] --> GW

    subgraph CPU_Tier ["🔵 CPU Tier (Graviton)"]
        GW["🌐 API Gateway<br/>LiteLLM Proxy<br/>Auth · Routing · Rate Limiting"]
        Agent["🤖 Agent Orchestrator<br/>Strands / LangChain<br/>Tool Calls · State"]
        SLM["🧠 SLM Inference<br/>llama.cpp<br/>Phi-3 Mini / Llama 3.2 3B"]
        Vector["🔍 Vector Retrieval<br/>OpenSearch 2.17<br/>Graviton Nodes"]

        GW --> Agent
        Agent --> SLM
        Agent --> Vector
    end

    subgraph GPU_Tier ["🟣 GPU / Trainium Tier"]
        LLM["🧠 Large LLM<br/>Complex Synthesis<br/>Long-Context Reasoning"]
    end

    SLM -->|"70–80% handled<br/>end-to-end"| Response["✅ Response"]
    SLM -->|"20–30% complex<br/>cases escalated"| LLM
    LLM --> Response

    style CPU_Tier fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style GPU_Tier fill:#f3e5f5,stroke:#7b1fa2,stroke-width:3px

    style User fill:#f5f5f5,stroke:#616161,color:#000
    style GW fill:#c8e6c9,stroke:#388e3c,color:#000
    style Agent fill:#a5d6a7,stroke:#2e7d32,color:#000
    style SLM fill:#81c784,stroke:#1b5e20,color:#000
    style Vector fill:#c8e6c9,stroke:#388e3c,color:#000
    style LLM fill:#ce93d8,stroke:#6a1b9a,color:#000
    style Response fill:#f5f5f5,stroke:#616161,color:#000
```

**Components running on CPU (Graviton):**

* API gateway / LiteLLM proxy - handles auth, routing, rate limiting
* Agent orchestrator (Strands, LangChain) - manages tool calls and state
* SLM inference service - `llama.cpp` or Ray Serve with Phi-3 Mini or Llama 3.2 3B
* Vector retrieval - OpenSearch 2.17 on Graviton nodes

**Components on GPU/Trainium:**

* Heavyweight LLM for complex synthesis, long-context reasoning

**Why this pattern works:** You cut LLM invocations significantly. In many agentic workflows, 70-80% of requests are classifiable or extractable by an SLM. The routing layer itself is a simple CPU service, and the entire CPU tier scales independently from the GPU tier.

This pattern also fits a fine-tuning lifecycle: collect domain data on CPU nodes, fine-tune on GPU, then deploy the quantized model back to CPU for inference, at substantially lower cost than an LLM endpoint.

### Pattern 2: High-Density CPU Model Farm

Not every AI workload is a language model. Production ML pipelines routinely deploy hundreds or thousands of smaller models: embeddings, recommenders, classifiers, BERT-based scorers, and computer vision models. Individually lightweight, these models become expensive when assigned their own GPU resources.

**The solution:** High-density CPU serving (bin-packing multiple models per node using TorchServe or Triton on Graviton), with Karpenter managing node lifecycle and KEDA scaling on observed load.

```mermaid
flowchart TB
    subgraph Scaling ["⚙️ Scaling & Lifecycle"]
        Karpenter["🔄 Karpenter<br/>Node Consolidation<br/>Spot + On-Demand"]
        KEDA["📈 KEDA<br/>Event-Driven Scaling<br/>Queue Depth · Latency"]
    end

    subgraph CPU_Farm ["🔵 Graviton Node Pool — High-Density Model Farm"]
        direction TB
        subgraph Node1 ["Node 1 (r7g.4xlarge)"]
            M1["Embedding Model A"]
            M2["Classifier B"]
            M3["BERT Scorer C"]
            M4["Recommender D"]
        end
        subgraph Node2 ["Node 2 (r7g.4xlarge)"]
            M5["Embedding Model E"]
            M6["CV Model F"]
            M7["Classifier G"]
            M8["Summarizer H"]
        end
    end

    subgraph RAG_Flow ["🔗 RAG Extension"]
        Chunk["📄 Chunking<br/>CPU"]
        Embed["🔢 Embedding<br/>CPU Farm"]
        Search["🔍 OpenSearch<br/>Retrieval"]
        LLM_Gen["🧠 LLM<br/>GPU — Final Generation"]
    end

    Karpenter --> CPU_Farm
    KEDA --> CPU_Farm
    Chunk --> Embed --> Search --> LLM_Gen

    style Scaling fill:#fff3e0,stroke:#e65100,stroke-width:2px
    style CPU_Farm fill:#e8f5e9,stroke:#2e7d32,stroke-width:3px
    style RAG_Flow fill:#e3f2fd,stroke:#1565c0,stroke-width:2px
    style Node1 fill:#c8e6c9,stroke:#388e3c
    style Node2 fill:#c8e6c9,stroke:#388e3c

    style Karpenter fill:#ffcc80,stroke:#e65100,color:#000
    style KEDA fill:#ffcc80,stroke:#e65100,color:#000
    style M1 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M2 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M3 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M4 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M5 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M6 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M7 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style M8 fill:#a5d6a7,stroke:#2e7d32,color:#000
    style Chunk fill:#90caf9,stroke:#1565c0,color:#000
    style Embed fill:#90caf9,stroke:#1565c0,color:#000
    style Search fill:#90caf9,stroke:#1565c0,color:#000
    style LLM_Gen fill:#ce93d8,stroke:#7b1fa2,color:#000
```

**This pattern extends naturally into RAG architectures:** embedding generation, document chunking, and retrieval from OpenSearch 2.17 all run cost-effectively on CPU nodes, feeding results to a GPU-hosted LLM only for the final generation step. The CPU farm handles the volume; the GPU handles the complexity.

## Optimization Best Practices

With provisioning and scaling in place, a few optimizations make a real difference in production.

### **1. Quantization: the highest-leverage CPU Optimization.** 

Running a 7B model at full BF16 on CPU is impractical; running it at Q4 with llama.cpp on Graviton is viable and cost-effective. 

**Recommended approach:** Build llama.cpp with ARM-optimized BLAS backends, setting n_threads equal to the vCPU count, and selecting Q4_K_M or Q8_0 quantization formats for the best balance of quality and throughput. Graviton4 instances add SVE2 support, which provides a measurable uplift for GEMM-heavy workloads underpinning model inference.

|Quantization	|Quality Impact	|Throughput vs BF16	|Use Case	|
|---	|---	|---	|---	|
|Q4_K_M	|Low (< 1% perplexity delta)	|~4–5x faster	|Production default for SLMs	|
|---	|---	|---	|---	|
|Q8_0	|Negligible	|~2x faster	|Quality-sensitive tasks	|
|Q5_K_M	|Very low	|~3.5x faster	|Balance of quality and speed	|
|BF16	|None	|1x (baseline)	|Avoid on CPU for 7B+ models	|

**Graviton4 and SVE2:** Graviton4 instances (`r8g`, `c8g`) add SVE2 (Scalable Vector Extension 2) support, providing a measurable uplift for GEMM-heavy workloads underpinning model inference. Build `llama.cpp` with `-march=armv9-a+sve2` to take advantage of this.


<Admonition type="tip" title="Benchmark Reminder">
Q4_K_M quality degradation varies by model and task. Always evaluate on your evaluation set before deploying to production.
</Admonition>

### **2. Bin-packing maximizes utilization.** 

For classical ML and embedding models (typically <500MB each), the goal is **maximum pod density per node at stable tail latency**. Two things determine whether you achieve that: accurate resource requests, and controlled threading. Everything else is secondary. Base your `requests` on observed p50–p90 usage under realistic load. Use Goldilocks, VPA recommendations, or Prometheus histograms from a load test, but never guess. Defaults are almost always wrong in both directions.

ML libraries (PyTorch, ONNX Runtime, MKL, OpenBLAS) will spawn as many threads as they can see vCPUs on the node, not the CPUs allocated to the pod. On a dense node with 20 pods, that means every pod tries to spawn 32 threads. The node thrashes on context switching and p99 latency spikes. Fix it explicitly:

```
env:
  - name: OMP_NUM_THREADS
    value: "2"          # match your cpu request (2000m = 2 threads)
  - name: MKL_NUM_THREADS
    value: "2"
  - name: OPENBLAS_NUM_THREADS
    value: "2"
  - name: INTRA_OP_NUM_THREADS    # PyTorch / ONNX Runtime
    value: "2"
  - name: NUM_INTER_THREADS
    value: "1"          # keep inter-op parallelism minimal to avoid nested spawning
```

Set each value equal to or below your CPU request. For pods with 4+ cores, benchmark starting at 2-4 threads. Many small models perform better with fewer threads due to cache efficiency. If you use HPA with many thin pods, 1–2 threads per pod almost always wins.

### **3. Spot + Karpenter Consolidation**

Karpenter's consolidation works exceptionally well for CPU inference workloads because interruption handling is straightforward for stateless inference pods with queue-backed pipelines.

```
# Karpenter disruption config for cost-optimal CPU tier
disruption:
  consolidationPolicy: WhenUnderutilized
  consolidateAfter: 30s
  budgets:
    - nodes: "20%"  # Never consolidate more than 20% of nodes at once
```

### **4. Multi-arch images maximize scheduling flexibility.**

Build for both `arm64` and `amd64` to maximize scheduling flexibility. Karpenter can then choose the best available instance type across Graviton and x86 families without pod scheduling failures.

### 5. Observability: Instrument from Day One

Without observability at the model layer, you're scaling blindly. Expose Prometheus metrics for every inference service and use them to drive both KEDA scaling and operational dashboards.

**Key metrics to instrument:**

|Metric	|Description	|Alerting Threshold	|
|---	|---	|---	|
|`llama_tokens_per_second`	|Throughput per replica	|Alert if < 50% of baseline	|
|---	|---	|---	|
|`llama_pending_requests`	|Queue depth	|Scale trigger at > 5	|
|`llama_request_duration_ms`	|End-to-end latency histogram	|Alert on p95 > SLO	|
|`llama_model_load_time_seconds`	|Cold start time	|Alert if > 30s	|
|`container_memory_working_set_bytes`	|RSS memory per podAlert at 85% of limit	|Alert at 85% of limit	|

## ****Evaluating Model Quality for CPU-First Workloads****

Deploying a quantized SLM on CPU is a cost and latency decision. It only makes sense if the model still produces correct, useful outputs for your workload. This section explains how to validate that.

The tradeoff is clear: smaller models or quantization cut compute cost but can reduce quality. The impact varies, sometimes negligible, sometimes severe. The workloads that shine on CPU (classification, extraction, routing, summarization, embeddings) often retain good quality in the 3B–7B range with proper quantization and prompting.

### What to evaluate

Different workloads degrade in different ways. Here are a few common examples.

|
****Workload****	|
****What may  degrade****	|
****What to  measure****	|
|---	|---	|---	|
|
Intent  or ticket classification	|
Errors  on ambiguous inputs	|
Accuracy,  F1 per class	|
|
Structured  extraction (JSON)	|
Missing  fields or wrong schema	|
Exact  match, schema validity	|
|
RAG  answers	|
Hallucinations  or ignoring context	|
Faithfulness,  answer relevance	|
|
Summarization	|
Missing  facts or poor coverage	|
ROUGE-L,  BERTScore, human review	|
|
Agent  routing	|
Selecting  the wrong tool	|
Tool  accuracy	|
|
Embeddings	|
Worse  retrieval quality	|
Recall@K,  NDCG	|

### A practical evaluation workflow

The goal is to create a **quality check before production**, similar to how you would run a load test before choosing an instance type.

#### **Step 1: Build a small evaluation dataset**

Avoid generic benchmarks like MMLU. They measure general reasoning, not your real task. Instead, create a small dataset from your actual workload. You usually only need:

* 100-300 labeled examples
* a mix of normal inputs and edge cases
* real context for RAG workloads

#### **Step 2: Establish a baseline**

Run the dataset against your trusted model (e.g GPT-4o or a large Llama model). This gives you a **quality baseline**. Your CPU model doesn’t need to beat it, but it should stay reasonably close.

#### **Step 3: Test the CPU model**

Run the same dataset on the smaller or quantized model. For structured tasks, use a deterministic prompt (``temperature: 0``) so results are reproducible.

Example:

```
`python eval_classify.py \
`  `--endpoint http://llama-server:8080/v1/chat/completions \
`  `--model phi-3-mini-4k-instruct \
`  `--eval-set eval_set.jsonl`
```

A simple classification evaluation script might look like this:

```
`accuracy = correct_predictions / total_examples
 print(f"Accuracy: {accuracy:.2%}")`
```

#### **Step 4: Compare against your quality threshold**

Define what “good enough” means **before running the test**.
For example:

```
If SLM accuracy ≥ (baseline − 5 percentage points) → OK for CPU
 If SLM accuracy < (baseline − 5 points) → consider GPU or a larger model
```

The right threshold depends on the task. A support classifier reviewed by humans can tolerate more errors than a system making automatic decisions.

### How to recover quality

If results performs poorly, try these before abandoning CPU:

* **Use a higher-quality quantization format**: Moving from Q4 to Q8 often restores much of the lost quality.
* **Add few-shot examples in the prompt**: Including a few labeled examples can improve results significantly.
* **Fine-tune the model on your domain**: A small model trained on your data can outperform a much larger generic model for specific tasks.
* **Use hybrid routing**: Let the SLM handle simple cases and send difficult inputs to a larger model.

### The key idea

The goal is not to prove that CPUs perform like GPUs. They don’t when running very large models. Instead, we want to identify the parts of your workload where a **small model on CPU is good enough**, and only use larger models when necessary. The important step is to **measure first and decide based on data**.

With your instance family selected from the benchmark workflow and your quality threshold validated against a task-specific eval set, you have everything you need to deploy with confidence. Here's how to put it all together.

## Getting Started

Deploy a quantized SLM on Graviton in under 30 minutes with this checklist.

### Step 1: Create a Graviton EKS Node Group

```
# Using eksctl — add Graviton nodegroup to existing cluster
eksctl create nodegroup \
  --cluster your-cluster \
  --name graviton-inference \
  --node-type r7g.4xlarge \
  --nodes-min 1 \
  --nodes-max 10 \
  --node-labels "tier=cpu-inference" \
  --managed

# Or with Karpenter (preferred for production)
kubectl apply -f - <<EOF
apiVersion: karpenter.sh/v1
kind: NodePool
metadata:
  name: graviton-inference
spec:
  template:
    spec:
      nodeClassRef:
        group: karpenter.k8s.aws
        kind: EC2NodeClass
        name: default
      requirements:
        - key: kubernetes.io/arch
          operator: In
          values: ["arm64"]
        - key: karpenter.sh/capacity-type
          operator: In
          values: ["on-demand"]
        - key: node.kubernetes.io/instance-type
          operator: In
          values: ["r7g.2xlarge", "r7g.4xlarge"]
  limits:
    cpu: "64"
EOF
```

### Step 2: Deploy an SLM with llama.cpp

```
# llama-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: slm-phi3-mini
spec:
  replicas: 1
  selector:
    matchLabels:
      app: slm-phi3-mini
  template:
    metadata:
      labels:
        app: slm-phi3-mini
    spec:
      nodeSelector:
        kubernetes.io/arch: arm64
      initContainers:
        - name: download-model
          image: public.ecr.aws/aws-cli/aws-cli:latest
          command:
            - aws
            - s3
            - cp
            - s3://your-model-bucket/phi-3-mini-q4_k_m.gguf
            - /models/model.gguf
          volumeMounts:
            - name: model-storage
              mountPath: /models
      containers:
        - name: llama-server
          image: your-ecr-repo/llama-server:latest  # multi-arch image
          command:
            - llama-server
            - --model
            - /models/model.gguf
            - --n-threads
            - "8"
            - --ctx-size 
            - "4096" 
            - --batch-size 
            - "512" 
            - --metrics
            - --port
            - "8080"
            - --host
            - "0.0.0.0"
          resources:
            requests:
              cpu: "8"
              memory: "8Gi"
            limits:
              cpu: "8"
              memory: "10Gi"
          env:
            - name: OMP_NUM_THREADS
              value: "8"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 60
            periodSeconds: 30
            failureThreshold: 3
          volumeMounts:
            - name: model-storage
              mountPath: /models
          ports:
            - containerPort: 8080
              name: inference
      volumes:
        - name: model-storage
          emptyDir: {}
---
apiVersion: v1
kind: Service
metadata:
  name: slm-phi3-mini
spec:
  selector:
    app: slm-phi3-mini
  ports:
    - port: 8080
      targetPort: 8080
kubectl apply -f llama-deployment.yaml
kubectl wait --for=condition=ready pod -l app=slm-phi3-mini --timeout=300s
```

**Note:** Add a `readinessProbe` that checks the `llama.cpp` `/health` endpoint before the pod is marked ready. Loading the model can take **15–30 seconds on a cold start**, and without a readiness probe Kubernetes may start sending traffic before the server is actually ready to handle requests.

A `livenessProbe` is also included. The readiness probe ensures traffic only reaches the pod after the model has finished loading, while the liveness probe restarts the container if the server becomes unresponsive after startup. The `initialDelaySeconds: 60` on the liveness probe gives the server enough time to finish loading the model before Kubernetes begins health checks.

Both probes use the same `/health` endpoint exposed by `llama.cpp` when the `--metrics` flag is enabled.

### Step 3: Validate and Measure

```
# Smoke test
 curl http://$(kubectl get svc slm-phi3-mini -o jsonpath='{.spec.clusterIP}'):8080/v1/chat/completions \
   -H "Content-Type: application/json" \
   -d '{ 
        "model": "phi-3-mini", 
        "messages": [{"role": "user", "content": "Classify this support ticket: My login is broken"}], 
        "max_tokens": 20, 
        "temperature": 0.0 
  }'

 # Load test with hey (install: go install github.com/rakyll/hey@latest)
 hey -n 500 -c 20 \
   -m POST \
   -H "Content-Type: application/json" \
   -d '{
        "model":"phi-3-mini",
         "messages": [{"role": "user", "content": "Classify this support ticket: My payment failed but money was deducted"}], 
        "max_tokens": 50 
        "temperature": 0.0 
  }' \
   http://$(kubectl get svc slm-phi3-mini -o jsonpath='{.spec.clusterIP}'):8080/v1/chat/completions


 # Prometheus metrics (requires --metrics flag in llama-server command)
# Adjust namespace and service name to match your Prometheus installation
 kubectl port-forward svc/prometheus-operated -n monitoring 9090:9090 &

# Tokens generated per second (generation throughput)
 curl "http://localhost:9090/api/v1/query?query=rate(llamacpp:predicted_tokens_seconds[5m])"
# Prompt eval throughput
curl "http://localhost:9090/api/v1/query?query=rate(llamacpp:prompt_tokens_seconds[5m])"
# KV cache pressure — watch this under load
curl "http://localhost:9090/api/v1/query?query=llamacpp:kv_cache_usage_ratio"

```

### Step 4: Compare Costs

Once you have throughput numbers from Step 3, the next step is to estimate **cost per request**. Below is a simple example using current on-demand pricing in us-east-1 (March 2026):

* **r7g.4xlarge** – ~$0.8568/hour (16 vCPU, 128 GiB RAM)
* **g6.xlarge** – ~$0.8048/hour (4 vCPU + 1× L4 GPU, 16 GiB RAM)

Assume the model **phi-3-mini (Q4_K_M)** and an average response size of **50 output tokens per request**. with a low-to-moderate concurrency:

* r7g.4xlarge. Typical measured throughput: 35 tokens/sec total 
    * → Requests/sec = 35 ÷ 50 ≈ **0.7**
    * → Requests/hour ~2,520 
    * → Cost per 1,000 requests: $0.8568 ÷ 2,520 × 1,000 → **~$0.34 per 1,000 requests**
* g6.xlarge (L4 GPU). Typical measured throughput: 50 tokens/sec total (GPUs are underutilized without sustained high batching or many parallel requests)
    * → Requests/sec = 50 ÷ 50 ≈1 ****
    * → Requests/hour ~3,600 
    * → Cost per 1,000 requests: $0.8048 ÷ 3,600 × 1,000 → **~$0.22 per 1,000 requests**

**Important Adjustment**
Many production SLM workloads (support ticket classification, intent detection, agent routing, lightweight summarization) run at **10–30 requests/sec** or less across the entire cluster, often with bursty or intermittent traffic. In these conditions:

* The GPU frequently sits partially idle, but you still pay nearly the full $0.8048/hour.
* Graviton instances benefit heavily from spot pricing, reserved instances, or Savings Plans (commonly reducing effective cost by 50–70%).

After discounts, Graviton’s effective cost per 1,000 requests can easily drop to **$0.10–$0.20**, while the GPU remains closer to **$0.20–$0.35+** due to lower utilization.

**Quick formula to calculate your own numbers**
After running the load test from Step 3, plug in your own numbers:

```
requests/sec = your_tokens_per_sec ÷ 50 
requests/hour = requests/sec × 3600 
cost per 1,000 requests = (hourly price ÷ requests/hour) × 1,000
```

For the low-to-moderate traffic patterns typical of CPU-optimized workloads (< ~30–50 requests/sec cluster-wide), Graviton-based inference is usually significantly more cost-effective (especially once you apply spot, reserved, or Savings Plan pricing). The GPU only becomes clearly cheaper per request when you sustain high concurrency and can keep it well-utilized. Run the Step 3 load test with representative traffic levels, record your actual tokens/sec, factor in your expected utilization and any discounts, then use the formula to see which platform delivers the best economics for your specific use case.

## Conclusion

Most of the work in a production AI pipeline — routing, classification, retrieval, orchestration, and a growing share of inference — runs well on CPU. Graviton4 with SVE2 and up to 50% better price-performance than x86 makes it the default starting point for CPU inference in 2026. Q4_K_M quantization on `llama.cpp` makes 7B models viable on CPU, and Karpenter + KEDA + Spot give you cost-optimal scaling without sacrificing availability.

The thresholds in this guide are informed starting points. Your model, data, and SLOs determine the right answer — benchmark before you commit.

### Resources and Further Reading

|Resource	|Link	|
|---	|---	|
|Amazon EKS Documentation	|https://docs.aws.amazon.com/eks/	|
|---	|---	|
|AWS Graviton Developer Guide	|https://github.com/aws/aws-graviton-getting-started	|
|Karpenter Documentation	|https://karpenter.sh/docs/	|
|KEDA Documentation	|https://keda.sh/docs/	|
|llama.cpp GitHub	|https://github.com/ggerganov/llama.cpp	|
|Ray Serve Documentation	|https://docs.ray.io/en/latest/serve/	|
|OpenSearch on Graviton	|https://docs.aws.amazon.com/opensearch-service/	|
|AWS Spot Best Practices	|https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/spot-best-practices.html	|