---
title: "Resolving OOM in PPO/GRPO with Large Models"
collection: journal
excerpt: "PPO and GRPO training with models >7B caused OOM errors on A100 GPUs due to multiple full model replicas. This post details optimization strategies to fix it."

---

## Issue

When training reinforcement learning algorithms like **PPO** or **GRPO** with **models larger than 7B parameters** (e.g., LLaMA, Mistral) on an **NVIDIA A100 (80GB)** GPU, we consistently ran into **Out-Of-Memory (OOM)** errors — even with mixed precision enabled.

This is because PPO/GRPO architectures typically require loading **multiple full model replicas** during training:

- **Policy model**
- **Reference (old) policy model**
- **Reward model**
- **Value model**

The combined memory footprint far exceeds GPU capacity without aggressive optimization.

## Solution

We applied a set of targeted memory-saving strategies to resolve the issue:

### 1. 🤝 Model Sharing with LoRA & Adaptive Weight Loading

Instead of loading **two full policy models** (policy + reference), we:

- Applied **LoRA adapters** to the policy model
- Used **`adaptive_weight_loading`** to **switch adapters** between the reference and current policy dynamically during training
- This allowed us to **load only one model** at a time (with adapter swap), saving ~50% of memory immediately.

### 2. 🧠 Gradient Checkpointing (Selective)

Enabled **gradient checkpointing** to reduce memory during backward passes. However:

- Applied it **only to non-attention layers** due to instability when used with **flash attention** or **eager execution**
- In full attention blocks, checkpointing **caused severe slowdowns or training stalls**, so we kept it off there.

### 3. 🌸 Use bfloat16 / float16 Precision (No Quantization)

Switched model precision to **bfloat16** or **float16** (depending on model support), which reduced memory usage **without quantizing weights**. This preserved model performance while keeping memory consumption manageable.

## 💡 Takeaway

Training large models with PPO/GRPO requires careful memory engineering:

- Use **LoRA + adapter switching** to avoid redundant model loading  
- Enable **gradient checkpointing selectively**, especially avoiding attention bottlenecks  
- Prefer **bfloat16/float16 precision** over quantization when full accuracy is needed

With these strategies, we were able to successfully train 7B+ models for RL fine-tuning on a single A100 GPU without OOM errors.
