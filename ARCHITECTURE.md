# Architecture Documentation

This document provides visual architecture documentation for RKLLM Toolkit - the model conversion pipeline for Rockchip NPU.

## Table of Contents

1. [System Overview](#system-overview)
2. [Pipeline Architecture](#pipeline-architecture)
3. [Component Interaction](#component-interaction)
4. [Data Flow](#data-flow)
5. [Batch Processing Flow](#batch-processing-flow)
6. [Model Conversion Pipeline](#model-conversion-pipeline)

---

## System Overview

```mermaid
graph TB
    subgraph "Input Sources"
        HF[HuggingFace Hub<br/>Model Repository]
        GGUF[GGUF Files<br/>Local Models]
        LoRA[LoRA Adapters<br/>Fine-tuned Weights]
    end

    subgraph "Docker Container"
        subgraph "Pipeline Variants"
            Interactive[Interactive Pipeline<br/>Single Model]
            NonInteractive[Non-Interactive Pipeline<br/>Batch Processing]
        end

        subgraph "Core Classes"
            RemotePipeline[RKLLMRemotePipeline]
            HubHelpers[HubHelpers]
        end

        subgraph "RKLLM SDK"
            RKLLM_Lib[rkllm_toolkit v1.1.2]
            Converter[Model Converter]
            Quantizer[Quantization Engine]
        end
    end

    subgraph "Output"
        LocalModel[Local .rkllm Files]
        HFUpload[HuggingFace Hub<br/>Converted Models]
    end

    HF --> RemotePipeline
    GGUF --> RemotePipeline
    LoRA --> RemotePipeline

    Interactive --> RemotePipeline
    NonInteractive --> RemotePipeline

    RemotePipeline --> RKLLM_Lib
    RKLLM_Lib --> Converter
    RKLLM_Lib --> Quantizer

    Converter --> LocalModel
    Quantizer --> LocalModel

    RemotePipeline --> HubHelpers
    HubHelpers --> HFUpload
    LocalModel --> HFUpload
```

---

## Pipeline Architecture

### Interactive vs Non-Interactive

```mermaid
graph LR
    subgraph "Interactive Mode"
        I_Start[User Starts] --> I_Prompts[Inquirer Prompts]
        I_Prompts --> I_Config[Single Config]
        I_Config --> I_Convert[Convert Model]
        I_Convert --> I_Upload[Upload to Hub]
    end

    subgraph "Non-Interactive Mode"
        N_Start[Script Starts] --> N_Config[Hardcoded Config]
        N_Config --> N_Loop[Nested Iteration]

        subgraph "Iteration Matrix"
            N_Models[Models List]
            N_Quant[Quantization Types]
            N_Hybrid[Hybrid Ratios]
            N_Opt[Optimization Levels]
        end

        N_Loop --> N_Models
        N_Models --> N_Quant
        N_Quant --> N_Hybrid
        N_Hybrid --> N_Opt

        N_Opt --> N_Convert[Convert Each]
        N_Convert --> N_Batch[Batch Upload]
    end
```

---

## Component Interaction

```mermaid
graph TB
    subgraph "Pipeline Scripts"
        Interactive[interactive_pipeline.py]
        NonInteractive[noninteractive_pipeline.py]
    end

    subgraph "RKLLMRemotePipeline Class"
        Init[__init__<br/>Configuration]
        UserInputs[user_inputs<br/>Inquirer Prompts]
        BuildVars[build_vars<br/>Path Generation]
        Pipeline[remote_pipeline_to_local<br/>Main Workflow]
    end

    subgraph "HubHelpers Class"
        Auth[HF Authentication]
        Validate[Repository Validation]
        ModelCard[Model Card Generation]
        Upload[Bulk Upload]
    end

    subgraph "RKLLM Library"
        RKLLM[rkllm.RKLLM]
        Load[load_huggingface<br/>load_gguf]
        Build[build<br/>Quantize & Convert]
        Export[export_rkllm<br/>Save Binary]
    end

    subgraph "External Services"
        HFHub[HuggingFace Hub API]
        HFTransfer[hf_transfer<br/>Rust Downloader]
    end

    Interactive --> Init
    NonInteractive --> Init

    Init --> UserInputs
    UserInputs --> BuildVars
    BuildVars --> Pipeline

    Pipeline --> Auth
    Pipeline --> Validate
    Pipeline --> Load
    Load --> Build
    Build --> Export
    Export --> ModelCard
    ModelCard --> Upload

    Auth --> HFHub
    Validate --> HFHub
    Upload --> HFHub
    Load --> HFTransfer
```

---

## Data Flow

### Single Model Conversion

```mermaid
sequenceDiagram
    participant User
    participant Pipeline as RKLLMRemotePipeline
    participant Hub as HubHelpers
    participant RKLLM as rkllm.RKLLM
    participant HF as HuggingFace Hub

    User->>Pipeline: Start conversion
    Pipeline->>Pipeline: user_inputs()
    Note over Pipeline: Collect model_id, platform,<br/>quantization, optimization

    Pipeline->>Pipeline: build_vars()
    Note over Pipeline: Generate paths,<br/>calculate NPU cores

    Pipeline->>Hub: validate_repo(model_id)
    Hub->>HF: Check repository
    HF-->>Hub: Repository info

    alt Gated Repository
        Hub-->>Pipeline: Requires authentication
        Pipeline->>Pipeline: Prompt for token
    end

    Pipeline->>HF: snapshot_download(model_id)
    HF-->>Pipeline: Model files

    opt LoRA specified
        Pipeline->>HF: snapshot_download(lora_id)
        HF-->>Pipeline: LoRA weights
    end

    Pipeline->>RKLLM: load_huggingface(path, device)
    Note over RKLLM: Load model architecture

    Pipeline->>RKLLM: build(quantization, optimization, hybrid_ratio)
    Note over RKLLM: Quantize and compile

    Pipeline->>RKLLM: export_rkllm(output_path)
    Note over RKLLM: Save .rkllm binary

    Pipeline->>Hub: generate_modelcard()
    Hub->>Hub: Create README.md

    Pipeline->>Hub: upload_model(output_dir)
    Hub->>HF: Push to Hub
    HF-->>Hub: Upload complete

    Hub-->>Pipeline: Success
    Pipeline-->>User: Conversion complete
```

---

## Batch Processing Flow

```mermaid
flowchart TD
    subgraph "Configuration"
        Models["models = [<br/>'org/model-1',<br/>'org/model-2'<br/>]"]
        QuantTypes["quant_types = {<br/>'rk3588': ['w8a8', 'w8a8_g128'],<br/>'rk3576': ['w8a8', 'w4a16']<br/>}"]
        HybridRatios["hybrid_ratios = [0.0, 0.5, 1.0]"]
        OptLevels["optimization = [0, 1]"]
    end

    subgraph "Nested Iteration"
        ForModel[for model in models]
        ForQuant[for qtype in quant_types]
        ForHybrid[for ratio in hybrid_ratios]
        ForOpt[for opt in optimization]
    end

    subgraph "Conversion"
        Pipeline[RKLLMRemotePipeline]
        Build[build_vars]
        Convert[remote_pipeline_to_local]
    end

    subgraph "Output Collection"
        Outputs["output_dirs = []"]
        Append[Append output_dir]
    end

    subgraph "Bulk Upload"
        AfterLoop[After all conversions]
        BulkUpload[HubHelpers.bulk_upload]
    end

    Models --> ForModel
    QuantTypes --> ForQuant
    HybridRatios --> ForHybrid
    OptLevels --> ForOpt

    ForModel --> ForQuant
    ForQuant --> ForHybrid
    ForHybrid --> ForOpt

    ForOpt --> Pipeline
    Pipeline --> Build
    Build --> Convert
    Convert --> Append
    Append --> Outputs

    Outputs --> AfterLoop
    AfterLoop --> BulkUpload
```

---

## Model Conversion Pipeline

### Build Variables Generation

```mermaid
flowchart TD
    subgraph "Input Parameters"
        Platform[platform: rk3588 / rk3576]
        ModelID[model_id: org/model-name]
        Quant[quantization_type]
        Hybrid[hybrid_quantization_ratio]
        Opt[optimization_level]
    end

    subgraph "Platform Mapping"
        RK3588{rk3588?}
        RK3576{rk3576?}
        Cores3588[npu_cores = 3]
        Cores3576[npu_cores = 2]
    end

    subgraph "Path Generation"
        InputDir["./models/{model_name}/"]
        OutputDir["./models/{model_name}-{platform}/"]
        RKLLMFile["{model_name}.rkllm"]
    end

    subgraph "Naming Convention"
        RepoName["{username}/{model}-{platform}-{rkllm_version}"]
    end

    Platform --> RK3588
    RK3588 -->|Yes| Cores3588
    RK3588 -->|No| RK3576
    RK3576 -->|Yes| Cores3576

    ModelID --> InputDir
    ModelID --> OutputDir
    Platform --> OutputDir

    OutputDir --> RKLLMFile
    ModelID --> RepoName
    Platform --> RepoName
```

### Quantization Decision Tree

```mermaid
flowchart TD
    Platform{Target Platform?}

    Platform -->|rk3588| RK3588_Opts
    Platform -->|rk3576| RK3576_Opts

    subgraph RK3588_Opts[RK3588 Options]
        W8A8[w8a8<br/>8-bit weights, 8-bit activations]
        W8A8_G128[w8a8_g128<br/>Group size 128]
        W8A8_G256[w8a8_g256<br/>Group size 256]
        W8A8_G512[w8a8_g512<br/>Group size 512]
    end

    subgraph RK3576_Opts[RK3576 Options]
        W8A8_76[w8a8<br/>8-bit full precision]
        W4A16[w4a16<br/>4-bit weights, 16-bit acts]
        W4A16_G32[w4a16_g32<br/>Group size 32]
        W4A16_G64[w4a16_g64<br/>Group size 64]
        W4A16_G128[w4a16_g128<br/>Group size 128]
    end

    subgraph Hybrid[Hybrid Quantization]
        Ratio0[ratio = 0.0<br/>All low precision]
        Ratio05[ratio = 0.5<br/>Mixed precision]
        Ratio1[ratio = 1.0<br/>All high precision]
    end

    W8A8 --> Hybrid
    W8A8_G128 --> Hybrid
    W4A16 --> Hybrid
```

---

## Container Architecture

```mermaid
graph TB
    subgraph "Docker Host"
        subgraph "Interactive Container"
            IBase[Ubuntu 22.04]
            IPython[Python 3.10]
            ISDK[rkllm_toolkit-1.1.2]
            IInquirer[inquirer CLI]
            IScript[interactive_pipeline.py]
        end

        subgraph "Non-Interactive Container"
            NBase[Ubuntu 22.04]
            NPython[Python 3.10]
            NSDK[rkllm_toolkit-1.1.2]
            NScript[noninteractive_pipeline.py]
        end

        subgraph "Shared Resources"
            Models[(./models/ Volume)]
            HFCache[(~/.cache/huggingface)]
        end
    end

    subgraph "External Services"
        HFHub[HuggingFace Hub]
    end

    IScript --> Models
    NScript --> Models
    IScript --> HFCache
    NScript --> HFCache

    IScript --> HFHub
    NScript --> HFHub
```

---

## Configuration Reference

### Pipeline Parameters

| Parameter | Type | Description |
|-----------|------|-------------|
| `model_id` | string | HuggingFace model repository ID |
| `lora_id` | string | Optional LoRA adapter ID |
| `platform` | enum | `rk3588` or `rk3576` |
| `quantization_type` | string | Quantization method (varies by platform) |
| `hybrid_quantization_ratio` | float | 0.0-1.0, precision mix ratio |
| `optimization_level` | int | 0 (none) or 1 (enabled) |

### Environment Variables

| Variable | Purpose |
|----------|---------|
| `HF_TOKEN` | HuggingFace authentication |
| `HF_HUB_ENABLE_HF_TRANSFER` | Enable Rust-based fast downloads |

---

## File Structure

```
rkllm-toolkit/
├── docker-interactive/
│   ├── Dockerfile              # Interactive container
│   ├── interactive_pipeline.py # User-prompted conversion
│   └── requirements.txt
├── docker-noninteractive/
│   ├── Dockerfile              # Batch container
│   ├── noninteractive_pipeline.py # Automated batch conversion
│   └── requirements.txt
├── models/                     # Model storage (mounted volume)
│   ├── {model-name}/           # Downloaded source model
│   └── {model-name}-{platform}/ # Converted .rkllm output
└── README.md
```

---

## Upload File Exclusions

Files excluded from HuggingFace upload:
- `*.safetensors` (original weights)
- `*.pt` (PyTorch checkpoints)
- `*.bin` (binary weights)
- `*.gguf` (GGUF format)

Files included:
- `*.rkllm` (converted model)
- `config.json` (model config)
- `tokenizer*.json` (tokenizer files)
- `README.md` (generated model card)
