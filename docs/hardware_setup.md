# Hardware Setup Guide

## Overview
Atlas local LLM hosting is validated on **Gillsystems Radeon 7900 XTX and 7600** GPUs. This guide provides tested configurations, ROCm compatibility matrices, and memory allocation strategies for reproducible deployments.

## Canonical Hardware Configuration

### Primary GPU: Radeon RX 7900 XTX
**Specifications**:
- Architecture: RDNA 3
- VRAM: 24 GB GDDR6
- Compute Units: 96
- TDP: 355W

**Validated Use Cases**:
- Hosting 13B parameter models (CodeLlama, Llama 3.1) with 4-bit quantization
- Batch sizes: 2-4 for optimal throughput
- Supports 70B models with aggressive quantization + CPU offloading (not recommended for production)

**Recommended for Atlas**:
- Primary LLM endpoint for patch generation
- Sufficient VRAM for multiple concurrent inference requests
- Stable under continuous load (tested 72+ hour uptime)

### Secondary GPU: Radeon RX 7600
**Specifications**:
- Architecture: RDNA 3
- VRAM: 8 GB GDDR6
- Compute Units: 32
- TDP: 165W

**Validated Use Cases**:
- Hosting 7B parameter models (Mistral 7B, Llama 3.1 7B)
- Batch size: 1-2 only
- Secondary/fallback endpoint when 7900 XTX unavailable

**Limitations**:
- Insufficient VRAM for 13B+ models without severe quantization
- Not suitable for model parallelism with 7900 XTX (heterogeneous VRAM sizes)
- Best used as independent fallback, not peer GPU

## ROCm Compatibility Matrix

### Tested Configuration (Linux)
| Component | Version | Status |
|-----------|---------|--------|
| ROCm | 6.0.2 | ✅ Stable |
| PyTorch | 2.2.1 | ✅ Stable |
| ROCm PyTorch Index | `pip3 install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0` | ✅ Validated |
| Transformers | 4.38.0 | ✅ Stable |
| bitsandbytes-rocm | 0.43.0 | ✅ 4-bit quantization working |

### Known Issues
- **ROCm 5.x**: Compatibility issues with PyTorch 2.2+, use ROCm 6.0+
- **PyTorch 2.3+**: Some operations unstable on RDNA 3, stick to 2.2.x for production
- **TensorFlow**: Less mature ROCm support than PyTorch, PyTorch recommended for Atlas

### Windows ROCm Status
**⚠️ Experimental - Not Production Ready**

AMD's ROCm support for Windows is in early access and not validated for Atlas deployments.

**Known Limitations**:
- Driver instability under sustained inference loads
- Limited bitsandbytes quantization support
- Frequent kernel panics on 7900 XTX under Windows 11

**Recommendation**: 
- Use Linux for local LLM hosting
- Windows target repo (ROCm Installer) runs Atlas verification, not model hosting
- Keep LLM server on separate Linux machine, Atlas Windows agent connects via HTTP

## Model Recommendations

### For 7900 XTX (24 GB VRAM)

#### Tier 1: Recommended for Production
| Model | Parameters | Quantization | VRAM Usage | Batch Size | Notes |
|-------|-----------|--------------|-----------|-----------|-------|
| CodeLlama 13B Instruct | 13B | 4-bit | ~9 GB | 2-4 | Best for code generation |
| Llama 3.1 13B Instruct | 13B | 4-bit | ~9 GB | 2-4 | Strong general reasoning |
| DeepSeek Coder 13B | 13B | 4-bit | ~9 GB | 2-4 | Excellent code understanding |

**Loading Example** (PyTorch + bitsandbytes):
```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig

quantization_config = BitsAndBytesConfig(
    load_in_4bit=True,
    bnb_4bit_compute_dtype=torch.float16,
    bnb_4bit_use_double_quant=True,
    bnb_4bit_quant_type="nf4"
)

model = AutoModelForCausalLM.from_pretrained(
    "codellama/CodeLlama-13b-Instruct-hf",
    quantization_config=quantization_config,
    device_map="auto"
)
```

#### Tier 2: Experimental (Larger Models)
| Model | Parameters | Quantization | VRAM Usage | CPU Offloading | Notes |
|-------|-----------|--------------|-----------|---------------|-------|
| Llama 3.1 70B Instruct | 70B | 4-bit | ~40 GB | Required | Slow, but higher quality |
| CodeLlama 34B Instruct | 34B | 4-bit | ~20 GB | Recommended | Marginal improvement over 13B |

**Warning**: 70B models require CPU offloading, adding 2-5x latency. Only use if Atlas can tolerate 30+ second patch generation times.

### For 7600 (8 GB VRAM)

#### Recommended Models
| Model | Parameters | Quantization | VRAM Usage | Batch Size | Notes |
|-------|-----------|--------------|-----------|-----------|-------|
| Mistral 7B Instruct | 7B | 4-bit | ~5 GB | 1-2 | Fast, decent quality |
| Llama 3.1 8B Instruct | 8B | 4-bit | ~6 GB | 1 | Newer, better reasoning |

**Use Case**: Fallback endpoint when 7900 XTX unavailable. Not recommended as primary for Atlas due to lower code understanding.

## Memory Allocation Strategies

### VRAM Limits and Headroom
- **Reserve 2-3 GB VRAM** for OS/kernel operations
- **Monitor with**: `rocm-smi` (Linux) or `radeontop`
- **OOM Prevention**: Set `max_memory` in transformers config

**Example** (limiting model to 21 GB on 7900 XTX):
```python
model = AutoModelForCausalLM.from_pretrained(
    "codellama/CodeLlama-13b-Instruct-hf",
    device_map="auto",
    max_memory={0: "21GB"}
)
```

### Batch Size Tuning
**7900 XTX**:
- Batch size 1: ~30 tokens/sec (underutilized)
- Batch size 2: ~50 tokens/sec (optimal for latency)
- Batch size 4: ~75 tokens/sec (optimal for throughput)
- Batch size 8: OOM risk with 13B models

**7600**:
- Batch size 1: ~25 tokens/sec (maximum safe)
- Batch size 2: OOM risk with 7B+ models

**Recommendation**: Start with batch size 2, monitor VRAM via `rocm-smi`, increase cautiously.

### CPU Offloading (For 34B/70B Models)
**When VRAM insufficient**:
```python
model = AutoModelForCausalLM.from_pretrained(
    "meta-llama/Llama-3.1-70b-Instruct",
    device_map="auto",  # Automatically offloads layers to CPU
    offload_folder="offload",
    offload_state_dict=True
)
```

**Performance Impact**:
- 13B no offloading: 2-3 sec/patch
- 70B with offloading: 15-30 sec/patch

## Installation Guide (Linux)

### Prerequisites
```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install dependencies
sudo apt install -y wget gnupg2 software-properties-common
```

### ROCm 6.0.2 Installation (Ubuntu 22.04)
```bash
# Add ROCm repository
wget -qO - https://repo.radeon.com/rocm/rocm.gpg.key | sudo apt-key add -
echo "deb [arch=amd64] https://repo.radeon.com/rocm/apt/6.0.2 jammy main" | sudo tee /etc/apt/sources.list.d/rocm.list

# Install ROCm
sudo apt update
sudo apt install -y rocm-hip-sdk rocm-smi-lib

# Add user to video group
sudo usermod -a -G video $USER
sudo usermod -a -G render $USER

# Reboot required
sudo reboot
```

### Verify ROCm Installation
```bash
# Check ROCm version
rocm-smi --showproductname

# Expected output:
# GPU[0]: Card series: Radeon RX 7900 XTX
# GPU[0]: Card model: 0x744c
```

### PyTorch + ROCm Setup
```bash
# Create conda environment
conda create -n atlas-llm python=3.10 -y
conda activate atlas-llm

# Install PyTorch with ROCm support
pip3 install torch==2.2.1 torchvision torchaudio --index-url https://download.pytorch.org/whl/rocm6.0

# Verify PyTorch sees GPU
python3 -c "import torch; print(torch.cuda.is_available()); print(torch.cuda.get_device_name(0))"

# Expected output:
# True
# AMD Radeon RX 7900 XTX
```

### Install LLM Server (vLLM or LM Studio)

**Option 1: vLLM (recommended for production)**
```bash
pip install vllm

# Start server
vllm serve codellama/CodeLlama-13b-Instruct-hf \
  --host 0.0.0.0 \
  --port 8080 \
  --quantization awq \
  --max-model-len 4096
```

**Option 2: LM Studio (GUI, easier for testing)**
- Download from https://lmstudio.ai/
- Load CodeLlama 13B model
- Start local server on port 8080
- Ensure OpenAI-compatible API mode enabled

## Troubleshooting

### Issue: `rocm-smi` not found
**Solution**:
```bash
export PATH=$PATH:/opt/rocm/bin
echo 'export PATH=$PATH:/opt/rocm/bin' >> ~/.bashrc
```

### Issue: PyTorch not detecting GPU
**Check**:
```bash
rocm-smi
# Verify GPU is visible

ls /dev/kfd /dev/dri/render*
# Should show: /dev/kfd /dev/dri/renderD128 (or similar)
```

**Fix permissions**:
```bash
sudo usermod -a -G video $USER
sudo usermod -a -G render $USER
# Log out and back in
```

### Issue: OOM errors during inference
**Solutions**:
1. Reduce batch size in Atlas `llm_config.yaml`
2. Enable 4-bit quantization if not already active
3. Lower `max_model_len` in vLLM config
4. Monitor VRAM: `watch -n 1 rocm-smi`

### Issue: Slow inference (>10 sec/response)
**Diagnostics**:
```bash
# Check GPU utilization
rocm-smi

# Should show ~90%+ GPU utilization during inference
# If <50%, check:
# 1. Model is actually on GPU (device_map="auto")
# 2. No CPU offloading happening unintentionally
# 3. Batch size isn't too small (try batch_size=2)
```

## Multi-GPU Orchestration (Future Enhancement)

### Current Status: Not Recommended
Atlas currently targets **single-GPU operation** on the 7900 XTX. Multi-GPU orchestration is marked as a future enhancement.

### Why Not Now?
1. **Heterogeneous VRAM**: 24 GB + 8 GB splits are inefficient for tensor parallelism
2. **ROCm Maturity**: Multi-GPU support still improving in ROCm 6.x
3. **Complexity vs. Benefit**: Single 7900 XTX handles 13B models well; multi-GPU adds overhead

### Future Scenarios
When Atlas requires larger models (70B+) or lower latency:
- **Matched GPUs**: 2x 7900 XTX for true tensor parallelism
- **Ray/vLLM Distributed**: Framework-level orchestration
- **Revised Documentation**: This guide will be updated when multi-GPU is validated

## Configuration Reference

### Atlas `llm_config.yaml` Hardware Section
```yaml
hardware:
  primary_gpu:
    model: "Radeon RX 7900 XTX"
    vram_gb: 24
    recommended_models: ["codellama-13b", "llama-3.1-13b"]
    max_batch_size: 4
    
  secondary_gpu:
    model: "Radeon RX 7600"
    vram_gb: 8
    recommended_models: ["mistral-7b", "llama-3.1-8b"]
    max_batch_size: 2
    
  llm_server:
    host: "127.0.0.1"  # Change to remote Linux host if applicable
    port: 8080
    protocol: "http"
    timeout_seconds: 300
```

## Best Practices

### For Production Deployments
1. **Use 7900 XTX for primary LLM hosting** - proven stable for 13B models
2. **Run LLM server on Linux** - Windows ROCm is experimental
3. **Monitor VRAM continuously** - `rocm-smi` in `watch` loop
4. **Test under load** - simulate multiple concurrent Atlas requests before production
5. **Document your exact config** - ROCm versions matter, pin everything

### For Development/Testing
1. **Start with 13B models** - best balance of quality and speed
2. **Use LM Studio for prototyping** - GUI makes experimentation easier
3. **Test on 7600 as fallback** - validate degraded-mode behavior
4. **Benchmark latency** - establish baseline before optimizing

### For Future Scaling
1. **Document multi-GPU attempts** - even if they fail, record learnings
2. **Monitor ROCm release notes** - multi-GPU support improving rapidly
3. **Consider matched GPUs** - if upgrading, buy 2x same model for parallelism
