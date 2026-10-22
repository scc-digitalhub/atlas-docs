# How To

## Run LLMs on Atlas

Deploy an existing LLM on Atlas as an OpenAI-compatible API with DigitalHub's `vllmserve-text` runtime. This page explains how to choose Atlas resources and configure serving. For platform API details, see the serving section of the [platform LLM tutorial (0.16.x)](https://github.com/scc-digitalhub/digitalhub-tutorials/tree/0.16.x/s16-llm-fine-tuning).

### Prerequisites

- Connect to your [Atlas tenant](access.md) from the FBK network or VPN. Use its management UI and an interactive workspace configured for that tenant.
- Use the project setup and serving sections of the tutorial's [notebook-llm.ipynb](https://github.com/scc-digitalhub/digitalhub-tutorials/blob/0.16.x/s16-llm-fine-tuning/notebook-llm.ipynb).
- Obtain access to the chosen Hugging Face model, including approval for gated models, and create the project secret `HF_TOKEN`. Pass it to the service as shown in the tutorial; do not put credentials in source code or notebook cells.
- Check your tenant's GPU availability, computational allocation, and persistent-storage quota in [Resources and Quotas](resources.md).

### Choose an Atlas GPU profile

GPU resources are **not assigned by default**. Select a GPU profile explicitly for the serving deployment.

The names in the [Atlas profile table](resources.md#resource-usage-and-declaration) are display labels, not necessarily the identifiers accepted by the SDK. In your tenant's configured workspace, inspect the available profiles:

```python
import digitalhub as dh

dh.get_k8s_resource_profiles()
```

Use the exact identifier returned by the API or displayed in the workload submission UI. The platform tutorial uses `1xrtxa5000-shared` for serving; this is an example, not a guarantee that the same identifier or queue variant is available to your tenant. Do not derive identifiers by lowercasing the Atlas display labels or appending a queue suffix.

| Atlas profile family | GPUs | VRAM per GPU | Compute capability | Maximum CPU / host RAM / ephemeral disk | LLM considerations |
| --- | --- | --- | --- | --- | --- |
| `1x5000` to `3x5000` | 1-3 RTX 5000 | 16Gi | 7.5 | 13 cores / 39Gi / 39Gi per GPU | Tight memory budget for an 8B model. FP16 weights alone take about 16 GB, leaving little room for inference overhead on one GPU; consider a smaller model or a runtime-supported quantized model. |
| `1xA5000` to `3xA5000` | 1-3 RTX A5000 | 24Gi | 8.6 | 13 cores / 59Gi / 59Gi per GPU | More headroom for single-GPU serving, but context length and concurrency still need sizing. |
| `1xV100` to `8xV100` | 1-8 Tesla V100 | 32Gi | 7.0 | 10 cores / 61Gi / 61Gi per GPU | More VRAM per GPU, but older architecture limits runtime and kernel compatibility; do not select solely by memory size. |

The CPU, host-memory, and ephemeral-disk limits above scale with GPU count; the complete values are in [Resources and Quotas](resources.md). Host RAM is **not** GPU VRAM. Lower CPU or host-memory requests may reduce quota usage, but do not increase GPU memory or change the profile's maximum limits.

Multiple GPUs do not automatically act as one larger GPU. The model must be partitioned by a compatible runtime and configuration; selecting a multi-GPU profile alone does not enable vLLM tensor parallelism.

#### Precision and runtime compatibility

- Use FP16 (`--dtype half`) rather than BF16 on the pre-Ampere RTX 5000 and V100 GPUs, subject to runtime support.
- RTX A5000 supports newer GPU features, but the selected container, CUDA/PyTorch build, attention backend, and quantization implementation must also support the hardware.
- [Current vLLM GPU requirements](https://docs.vllm.ai/en/stable/getting_started/installation/gpu/) specify compute capability 7.5 or higher. V100 has capability 7.0, so it is not a supported choice for current vLLM releases. Check the actual DigitalHub serving image and its version-specific requirements. Use a compatible RTX profile for the tutorial's vLLM workflow.
- Check support for the specific model, quantization format, and attention backend, not just the minimum compute capability.

### Serve the model on Atlas

Use the platform tutorial's project setup and model registration steps, then create a `vllmserve-text` function for your existing model. For a base model without an adapter:

```python
proj = dh.get_or_create_project("atlas-llm-serving")
base_model = proj.new_model(
    name="base-model",
    kind="huggingface",
    path="huggingface://meta-llama/Llama-3.1-8B",
)
vllm_func = proj.new_function(
    name="vllm-text-serve",
    kind="vllmserve-text",
    model_name="base-model",
    url=base_model.key,
)
```

Replace the model identifier with the model you want to serve and can access. No adapter is required. If you already have a compatible adapter, register it and add the tutorial's `adapters` configuration; serving then needs both the base model and adapter.

Choose an available, runtime-compatible GPU profile and use the **reserved queue** for an endpoint that requires continuity. The tutorial's `1xrtxa5000-shared` example is interruptible and should only be used for disposable tests that tolerate downtime.

Adapt the tutorial's `vllm_func.run(...)` call:

- Set `profile` to the exact identifier for your selected GPU and queue.
- Start with a shorter context, for example `args=["--dtype", "half", "--gpu_memory_utilization", "0.90", "--max_model_len", "2048"]`, and tune after checking startup logs and memory usage. These are sizing examples, not guaranteed working values for every model.
- Do not copy the tutorial's `32768` context length or `0.95` GPU-memory utilization blindly. Model weights, any adapter, runtime buffers, KV cache, and concurrent requests all need VRAM. Longer contexts and more simultaneous requests increase demand.
- Keep host-memory and ephemeral-disk requests within the selected profile's limits. The tutorial's `resources={"mem": "24Gi", "disk": "20Gi"}` requests host RAM and ephemeral disk, **not** GPU VRAM. Budget disk for model downloads and staging as well.

Wait for the service to become ready, then use its platform-provided endpoint and the tutorial's OpenAI-compatible inference request. Set the request's model name to the one exposed by your deployment (`base-model` in the example above, or `adapter` when using the tutorial's registered adapter). Follow [Atlas access requirements](access.md); do not assume the endpoint is publicly reachable.

### Diagnose resource problems

| Symptom | What to check |
| --- | --- |
| Job or service remains pending | Exact profile identifier, tenant allocation, queue, requested GPU type/count, CPU and host-memory requests, and current capacity. Reserved capacity does not guarantee an immediate start. |
| CUDA out of memory | Per-GPU VRAM, model precision, sequence length, batch sizes, serving context, and concurrency. A higher host-memory request does not fix GPU exhaustion. |
| Host-memory termination or disk-full error | Host RAM, ephemeral-disk limits, PVC free space, and cache/checkpoint paths. These are separate from VRAM. |
| Unsupported GPU, dtype, or kernel error | GPU compute capability and the container's CUDA, PyTorch, vLLM, quantization, and attention-backend compatibility. |
| Workload restarts or endpoint disappears | Shared-queue eviction, persistent output paths, and application-level resume support. |

Consult [Resources and Quotas](resources.md) for current limits and [Troubleshooting](troubleshooting.md) for general Atlas issues.

### Optional: Fine-tune on Atlas

Fine-tuning is only needed if you want to adapt a model; it is not a prerequisite for serving. Open the tutorial notebook with its `src/` directory and `requirements.txt` from the same branch. Use the tutorial's Python 3.11 training environment and dependencies rather than upgrading packages independently.

The tutorial's training script uses FP16 compute, eager attention, and 4-bit NF4 quantization by default. Keep those settings as the starting point on RTX 5000 and V100, and check container compatibility. The tutorial's training profile `1xrtx5000` is an example; discover the exact identifier available to your tenant. The script's `device_map="auto"` is not a substitute for a tested distributed-training setup.

Training with QLoRA produces an adapter, not a standalone quantized model for serving. Serving still needs the base model, and its memory footprint must be sized independently of training.

Follow the tutorial's project, secret, function creation, and image-build steps. In its training submission:

1. Replace `profile` with a GPU profile available to your tenant.
2. Start with `quantization=4`, `train_batch_size=1`, and a modest `max_sequence_length` (for example, 1024). These are starting points, not a guarantee that a model will fit. Size evaluation batches too.
3. Use gradient accumulation to increase the effective batch size without increasing the per-device batch size. If CUDA runs out of memory, reduce sequence length and per-device training/evaluation batches before requesting more GPUs.
4. Mount persistent storage at `/app/local_data`, as in the notebook. Set `HF_HOME` to `/app/local_data/huggingface` so model downloads use that volume.

For example, after building `function_train` as described in the tutorial:

```python
TRAIN_PROFILE = "<exact GPU profile identifier from your tenant>"

train_run = function_train.run(
    action="job",
    profile=TRAIN_PROFILE,
    parameters={
        "model_id": "meta-llama/Llama-3.1-8B",
        "model_name": "atlas-llm-adapter",
        "hf_dataset_name": "team-bay/data-science-qa",
        "train_data_path": "DataScienceDataset.csv",
        "dev_data_path": "DataScienceDataset.csv",
        "quantization": 4,
        "train_batch_size": 1,
        "eval_batch_size": 1,
        "max_sequence_length": 1024,
    },
    secrets=["HF_TOKEN"],
    envs=[{"name": "HF_HOME", "value": "/app/local_data/huggingface"}],
    volumes=[{
        "volume_type": "persistent_volume_claim",
        "name": "atlas-llm-data",
        "mount_path": "/app/local_data",
        "spec": {"size": "50Gi"},
    }],
)
```

Replace the placeholder before running. The model and dataset above come from the platform example; use separate training and validation splits for a meaningful evaluation. The 50Gi volume is an illustrative starting size, not a storage requirement or an assurance of sufficient space.

#### Storage and interruption constraints

- Budget persistent storage for the downloaded base model, datasets, Hugging Face cache, checkpoints, and final adapter/tokenizer files. A 4-bit training configuration does not mean the downloaded base-model files occupy only 4 bits per parameter.
- The tutorial writes checkpoints under `/app/local_data/checkpoints/ground` and final weights under `/app/local_data/weights/ground`. Keep these paths on the persistent volume. Ephemeral disk is limited by the profile and is not suitable for durable outputs.
- PVC count, PVC capacity, and S3 capacity have [hard quota limits](resources.md#quotas). Check available capacity before requesting a volume; GPU availability does not imply storage availability.
- Use the reserved queue for important training runs. Shared-queue jobs may be evicted at any time. Persist checkpoints and implement and test checkpoint resume before relying on shared capacity: the tutorial calls `trainer.train()` without an explicit `resume_from_checkpoint`, so a platform restart alone does not guarantee training resumes from its last checkpoint.
