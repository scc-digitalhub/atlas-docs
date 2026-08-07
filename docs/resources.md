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

The execution resources are shared between all tenants. The usage of resources is limited by the quota assigned to each tenant. We define two types of resource limits:

- the resources that cannot be exceeded (fixed quota). This includes the **total number of PVCs** (Persistent Volume Claims), the **amount of S3 storage** assigned to the tenant, the **amount of PVC storage** used by the tenant. 
- the resources that can be exceeded (overbooking scenario). This includes for examply, memory, CPU, GPU, ephimeral storage, etc.

When launching jobs, services, and workspaces the following rules are applied:

- If the total amount of resources used by the workloads of the tenant does not exceed the quota, the new workloads will be started/executed successfully.
- If with the new workload the amount of fixed resources exceeds the quota, the new workload will be rejected.
- If with the new workload the amount of overbooking resources exceeds the quota, and there are no free resources available on the whole cluster, the new workload will be rejected.
- If with the new workload the amount of overbooking resources exceeds the quota, and there are free resources available on the whole cluster, the new workload will be started/executed successfully. This is called **overbooking scenario**. However, if the other tenants start workloads within their quotas and there are no free resources available on the whole cluster, the overbooked workload will be preempted: the jobs will return to the queue and pass to pending state, while the services will be stopped.

To see the status of the resources/quota used by/assigned to the tenant, use the corresponding monitoring dashboard.

The full list of quotas and default values is presented in the following table.

| **Resource** | **Value** | **Fixed** | **Description** |
|--------------|-----------|-----------------|-----------------|
| **PVC Storage** | 4Tb | yes | Amount of storage for Persistent Volume claims used by workspaces and jobs/services |
| **PVCs**     | 50  | yes | Number of PVCs used by workspaces and jobs/services |
| **S3** | 4Tb | yes | Amount of S3 storage used by workspaces and jobs/services |
| **Memory** | 128Gb | no | Amount of memory used by workspaces and jobs/services |
| **CPU** | 40 cores | no | Amount of CPU cores used by workspaces and jobs/services |
| **GPU** | 2 | no | Amount of GPU used by workspaces and jobs/services |
