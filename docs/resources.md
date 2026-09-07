# Resources and Quotas

## Resource Usage and Declaration

When launching the jobs and services, the users may declare the types and amount of resources necessary for the workload execution. The amount of resources may be specified in terms of CPU, memory, GPU or disk space. If not explicitly specified, the default values are used.

The following set of rules is used by the management layer to assign resources to the jobs and services:

1. **Workspaces**. Workspace resource configuration is explicit, defining the CPU, memory and storage amount. The configuration may be changed at any time, the workspace should be restarted. The disk space, however, **cannot be changed**.

2. **Jobs and Services**. For the jobs and services the resource declaration may be done explicity or through profilies:

- If no resource constraints are explicitly defined by the user, the default values are used for CPU, memory, and disk space. Specifically,
    - No GPU resources are assigned by default.
    - The default amount of memory is X Gb.
    - By default, the CPU usage is set to X core.
    - The default amount of disk space is X Gb.
- The user can define explicitly the **requested** amount of CPU, memory, and disk space. The max amount of CPU and memory is then set to 110% of that amount.
- To use GPUs the user should use the predefined profile defined in the cluster configuration. The profile defines the type and number of GPUs to be requested, as well as some other predefined values preconfigured according to the HW layout (e.g., memory and cores). While the profile defines the **maximum** amount of resources allocated to the job, the user can still define the **requested** amount (which is good to reduce the quota usage of the tenant).

See below the list of the currently defined profiles.

!!! Note
    The list of profiles and their characteristics is subject to change upon the migration of the cluster and changes in the HW setup.

!!! Note
    The list of profiles available to a single unit may vary. The list of the profiles available in the tenant may be obtained from the management layer using SDK or UI.

| **Profile** | **GPU type** | **Compute Capability** | **GPU count** | **VRAM per GPU** | **Max CPU** | **Max Memory** | **Max Ephimeral Disk** |
|--------------|-------------|---------------|-------------|---------------|---------------|--------------|-------------|
| **1xV100** | Tesla V100 | 7.0 | 1 | 32Gi | 10 | 61Gi | 61Gi |
| **2xV100** | Tesla V100 | 7.0 | 2 | 32Gi | 20 | 122Gi | 122Gi |
| **3xV100** | Tesla V100 | 7.0 | 3 | 32Gi | 30 | 183Gi | 183Gi |
| **4xV100** | Tesla V100 | 7.0 | 4 | 32Gi | 40 | 244Gi | 244Gi |
| **5xV100** | Tesla V100 | 7.0 | 5 | 32Gi | 50 | 305Gi | 305Gi |
| **6xV100** | Tesla V100 | 7.0 | 6 | 32Gi | 60 | 366Gi | 366Gi |
| **7xV100** | Tesla V100 | 7.0 | 7 | 32Gi | 70 | 427Gi | 427Gi |
| **8xV100** | Tesla V100 | 7.0 | 8 | 32Gi | 80 | 488Gi | 488Gi |
| **1x5000** | RTX 5000 | 7.5 | 1 | 16Gi | 13 | 39Gi | 39Gi |
| **2x5000** | RTX 5000 | 7.5| 2 | 16Gi | 26 | 78Gi | 78Gi |
| **3x5000** | RTX 5000 | 7.5 | 3 | 16Gi | 39 | 117Gi | 117Gi |
| **1xA5000** | RTX A5000 | 8.6 | 1 | 24Gi | 13 | 59Gi | 59Gi |
| **2xA5000** | RTX A5000 | 8.6 | 2 | 24Gi | 26 | 118Gi | 118Gi |
| **3xA5000** | RTX A5000 | 8.6 | 3 | 24Gi | 39 | 177Gi | 177Gi |


## Quotas

Execution resources are shared across tenants, and each tenant is assigned a resource quota. The platform treats storage limits and computational resource allocations differently:

- **Hard limits** cannot be exceeded. They apply to the number of Persistent Volume Claims (PVCs), PVC storage, and S3 storage. A request that would exceed one of these limits is rejected.
- **Computational allocations** cover CPU, memory, GPU, and ephemeral storage. Workloads can use the tenant's allocated capacity through the reserved queue or request currently unused capacity through the shared queue.

To view the resources assigned to a tenant and its current usage, open the [ATLAS monitoring dashboard](https://observability.atlas.fbk.eu/). You must sign in with your FBK account.

### Workload queues

The platform provides two ways to run computational workloads: a **reserved queue** and a **shared queue**. Select the appropriate execution option in the workload submission interface; queue placement and resource management are handled automatically.

!!! warning
    Every workload submitted to the shared queue is interruptible and can be evicted at any time. Use it only for workloads that can tolerate termination and restart.

#### Reserved queue

The reserved queue uses resources allocated to the research group or project. This capacity is protected from permanent use by other groups.

- Reserved workloads have higher scheduling priority than shared workloads.
- If allocated resources are temporarily being used by shared workloads, the platform can reclaim them for reserved workloads.
- This queue is appropriate for important batch workloads and workloads that should not depend on spare capacity.

Reserved capacity does not guarantee an immediate start. A workload remains pending if it requests more resources than are currently available, requires an unavailable resource type, or is waiting behind other reserved work.

#### Shared queue

The shared queue provides opportunistic access to resources that are currently unused by their owners. It improves overall platform utilization but provides no continuity guarantee.

- Shared workloads use spare capacity and do not own or reserve resources.
- They have lower scheduling priority than reserved workloads.
- Their start time depends on spare capacity being available.
- A running shared workload can be stopped when its capacity is needed for reserved work.

After eviction, a workload may return to a pending state and restart when sufficient spare capacity becomes available. The exact restart behavior depends on the workload type and the platform abstraction used to submit it.

Shared workloads should be restartable and idempotent, persist outputs outside the running instance, and checkpoint progress when supported. Do not use the shared queue for services or jobs that require uninterrupted execution.

#### Queue comparison

| **Property** | **Reserved queue** | **Shared queue** |
|--------------|--------------------|------------------|
| **Resource source** | Allocated project or group resources | Temporarily unused platform resources |
| **Resource ownership** | Allocated capacity | No allocated capacity |
| **Workload priority** | Higher | Lower |
| **Can reclaim shared capacity** | Yes | No |
| **Interruption expectation** | Does not depend on spare capacity | Possible at any time |
| **Recommended use** | Important or continuity-sensitive batch work | Fault-tolerant, restartable, best-effort work |

#### Choosing a queue

Choose the **reserved queue** when the workload is important, has limited restart support, or needs the resources allocated to the project.

Choose the **shared queue** when the workload is restartable and the benefit of using additional spare resources outweighs the risk of interruption. Suitable examples include parameter sweeps, independent simulations, and jobs that save frequent checkpoints.

For shared workloads:

- Save results and checkpoints to persistent storage, not only to the running instance's local filesystem.
- Make processing restartable and, where possible, idempotent.
- Split long computations into smaller independent units.
- Do not rely on a specific start time or uninterrupted execution window.
- Expect duplicate or partial work around an interruption and handle it safely.

### Default quotas

The following table lists the default quota values. The quotas available to a specific tenant may differ (check the monitoring dashboard to see assigned quotas).

| **Resource** | **Default value** | **Hard limit** | **Description** |
|--------------|-------------------|----------------|-----------------|
| **PVC Storage** | 4 TB | Yes | Persistent Volume Claim storage used by workspaces, jobs, and services |
| **PVCs** | 50 | Yes | Number of Persistent Volume Claims used by workspaces, jobs, and services |
| **S3** | 4 TB | Yes | S3 storage used by workspaces, jobs, and services |
| **Memory** | 128 GB | No | Memory allocated to the tenant for reserved workloads |
| **CPU** | 40 cores | No | CPU cores allocated to the tenant for reserved workloads |
| **GPU** | 2 | No | GPUs allocated to the tenant for reserved workloads |
