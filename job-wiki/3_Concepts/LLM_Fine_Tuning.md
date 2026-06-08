# LLM Fine-Tuning & Quantisation (Core Concepts & Gaps)

This document covers the theory behind Parameter-Efficient Fine-Tuning (PEFT) and Model Quantisation. It represents a growth area in your profile — while you understand the architectural concepts, you are actively building hands-on depth with these tools.

[← Back to Index](../README.md) | [← Previous: Kafka Deep Dive](./Kafka_Deep_Dive.md) | [Next: Gateway Stack: LiteLLM + Langfuse + LLM Guard](./Gateway_Stack_LiteLLM.md)

---

## 📈 Parameter-Efficient Fine-Tuning (PEFT)

Full fine-tuning of Large Language Models requires updating all parameters of the network. This is computationally expensive, requiring massive GPU clusters (A100/H100) and substantial VRAM. PEFT solves this by freezing the base model and training only a tiny fraction of additional parameters.

### 1. LoRA (Low-Rank Adaptation)
* **The Concept:** Instead of updating the massive weight matrix $W_0$ (of shape $d \times k$), LoRA decomposes the weight update $\Delta W$ into two low-rank matrices $A$ and $B$:
  $$\Delta W = B \times A$$
  where $B$ is of shape $d \times r$ and $A$ is of shape $r \times k$. The rank $r \ll \min(d, k)$ (typically $r = 8$ or $16$).

```text
Input x ──┬──> [ Frozen Weight W0 ] ──> Output W0(x)
          │                                 │
          └──> [ Matrix A (r) ]             │
                     │                      ▼
               [ Matrix B (d) ] ──> Output B(A(x)) ──> Summed Final Output
```

* **How it works:**
  - $A$ is initialized with a Gaussian distribution, and $B$ is initialized with zeros, so $\Delta W$ starts at zero.
  - During training, only $A$ and $B$ are updated, reducing trainable parameters by **99%**.
  - At inference, the weights $B \times A$ are mathematically merged back into the base weights $W_0$, resulting in zero latency overhead.

### 2. QLoRA (Quantised Low-Rank Adaptation)
QLoRA takes LoRA further by quantizing the base model to 4-bit representation, allowing fine-tuning on a single consumer-grade GPU.
* **Core Innovations:**
  - **NF4 (NormalFloat 4):** A specialized 4-bit data type optimized for normally distributed model weights.
  - **Double Quantisation:** Quantizes the quantization constants, saving an additional 0.37 bits per parameter.
  - **Paged Optimizers:** Uses CPU RAM backup to prevent out-of-memory (OOM) spikes during gradient checkpoint steps.

---

## ⚙️ Quantisation (FP16 → INT8 → INT4)

Quantisation reduces the numerical precision of weights to shrink the model's disk and memory footprint.

* **Precision comparison:**
  - **FP32:** Single precision float (32 bits / 4 bytes per parameter).
  - **FP16 / BF16:** Half precision float (16 bits / 2 bytes). Standard for model training.
  - **INT8:** 8-bit integer (1 byte). Reduces memory footprint by **50%** compared to FP16.
  - **INT4:** 4-bit integer (0.5 bytes). Allows a 70B parameter model to fit on a single V100 GPU.
* **Calibration:** During quantization, we map the continuous float values to discrete integers using scaling factors. If done poorly, this causes a drop in model intelligence (quantization error).

---

## 🛠️ Hugging Face Ecosystem & Your Study Plan

### Core Libraries:
1. **`transformers`:** Standard API for loading pre-trained models (`AutoModelForCausalLM`, `AutoTokenizer`).
2. **`peft`:** Used to wrap transformers models for LoRA training:
   ```python
   from peft import LoraConfig, get_peft_model
   config = LoraConfig(r=8, lora_alpha=32, target_modules=["q_proj", "v_proj"], lora_dropout=0.05, bias="none")
   model = get_peft_model(base_model, config)
   ```
3. **`bitsandbytes`:** Integrates quantization modules directly into transformers (e.g. `load_in_4bit=True`).

### 📚 Honest Gap & Growth Plan:
* **Current Knowledge:** High conceptual understanding. You have used pre-trained models via APIs (Claude, OpenAI, Vertex AI) and built RAG pipelines.
* **The Gap:** You have not run LoRA/QLoRA training loops in production or fine-tuned custom weights (e.g., Llama-3-8B) on raw GPU clusters.
* **Active Prep Plan:** You are currently practicing fine-tuning on small datasets (like custom customer support logs) using Google Colab notebooks with Llama-3, using Hugging Face's `SFTTrainer` (Supervised Fine-Tuning Trainer) to bridge this gap.
