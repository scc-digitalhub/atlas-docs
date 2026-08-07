# Migration and Roadmap

The fusion of the two clusters and the migration of the machines under the new architectural model will happen incrimentally. First, all the nodes of the old ATLAS Cluster will migrate following this schedule:

- **21/08/2026**: DGX2, RD4321
- **18/09/2026**: RF4421
- **16/10/2026**: RF4422
- **30/10/2026**: RF4423
- **12/11/2026**: RR4341

The migration of the ABACUS cluster will be completed by the end of 2026.

## Data Migration

When migrated, all the data on the servers will be deleted. To avoid losing the data, it is necessary to make a backup copy of the RELEVANT datasets and folders. One can use the Cloud [storage options provided by FBK](https://howto.fbk.eu/en/it-department/storage-and-network-services/) or use directly the Datalake of the new platform.

In the latter case it is necessary to use the corresponding S3 credentials for accessing the Datalake. If the AI platform instance is already in place for the unit, it is possible to use the personal S3 credentials. To obtain the credentials, use the [management layer CLI](https://scc-digitalhub.github.io/docs/cli/installation/) as presented [here](https://scc-digitalhub.github.io/docs/0.15/cli/usage/#obtaining-configuration-and-credentials).

Once credentials are available, it is possible to use tools like [rclone](https://rclone.org/install/) or [AWS CLI](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) for this, for example:

```bash
aws s3 sync <LOCAL_FOLDER> s3://<BUCKET_NAME>/path/to/folder 
```

Alternatively, it is possible to use the functionality of the AI platform catalog for uploading the datasets and artifacts:

```python
import digitalhub as dh

project = dh.get_or_create_project("datasets")

project.log_artifact("mydataset", kind="artifact", source="./path/to/folder/or/file/")

```

!!! note
    Please take into account that the space in the datalake is not infinite; copy only really important data before the space quota runs out. 

## Code Migration

To make the existing applications and experiments run under the new platform, it is necessary to adapt the code to the new architecture. This amounts to adding the scripts/operations to download the data before the original code and to upload the results once the code is complete (or incrementally if intermediate checkpoints should be saved as well).

When the AI platform functionality is used, it is possible for example to use the Python runtime to adapt the existing code to the new platform. See the example [Pytorch project tutorial](https://scc-digitalhub.github.io/docs/0.16/tutorials/ml-migration/migration/) that explains the necessary steps and operations.

# Roadmap

In addition to the existing functionality of the platform, based on the requests from the users, more features and functionality will incrimentally be added to the platform. This includes, but not limited to,:

- Support for [Hydra-compatible](https://hydra.cc/docs) jobs to spawn multiple experiment directly on the platform, without requiring intermediates like Slurm or Ray clusters.
- Support distributed computations using [Ray framework](https://docs.ray.io/en/latest/), such as for hyperparameter search or distributed training. The idea is to create the Ray clusters dynamically, based on the declarative specification, and launch the jobs on them.
- Support for SLURM environment on top of the Kubernetes cluster of the execution platform using the [corresponding Kubernetes operator](https://github.com/SlinkyProject/slurm-operator).  

Additional features and functionality will be added in the future. Please let us know if you have any suggestions.