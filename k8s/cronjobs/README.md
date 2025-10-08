# S3 to EFS Cleanup CronJob

This directory contains the Kubernetes manifests for a CronJob that runs in EKS
and removes files from an EFS mount once they have been confirmed to exist in an
Amazon S3 bucket. File names are sourced from a PostgreSQL compatible database.

## Components

- `s3-efs-cleanup-cronjob.yaml` – Defines the `ConfigMap` that stores the Python
  cleanup script and the CronJob specification.
- `s3-efs-cleanup-settings.yaml` – Provides an optional ConfigMap with default
  values for the job. Update the values or remove keys that you will provide via
  other mechanisms.

The CronJob expects the following supporting resources to exist in the cluster:

- A `PersistentVolumeClaim` named `efs-shared-pvc` that is backed by your EFS
  access point or file system.
- A `Secret` named `s3-efs-cleanup-secrets` that contains the database
  connection details and the target S3 bucket name (see below).
- A `ServiceAccount` named `s3-efs-cleanup` that has permissions to mount the
  EFS volume and, via IAM Roles for Service Accounts (IRSA), access the S3
  bucket.

## Required secret keys

Create the `s3-efs-cleanup-secrets` secret in the same namespace as the CronJob
with the following keys:

| Key        | Description                                |
| ---------- | ------------------------------------------ |
| `s3Bucket` | Name of the S3 bucket that stores the files |
| `dbHost`   | Database hostname                           |
| `dbPort`   | Database port (optional, defaults to 5432)  |
| `dbName`   | Database name                               |
| `dbUser`   | Database user                               |
| `dbPassword` | Database password                         |

Example:

```bash
kubectl create secret generic s3-efs-cleanup-secrets \
  --from-literal=s3Bucket=my-files-bucket \
  --from-literal=dbHost=mydb.example.us-east-1.rds.amazonaws.com \
  --from-literal=dbName=ingestion \
  --from-literal=dbUser=cleanup \
  --from-literal=dbPassword='replace-with-password'
```

## Deployment

Apply the manifests after customising them for your environment:

```bash
kubectl apply -f k8s/cronjobs/s3-efs-cleanup-settings.yaml
kubectl apply -f k8s/cronjobs/s3-efs-cleanup-cronjob.yaml
```

Use the following environment variables to tailor runtime behaviour:

- `S3_PREFIX` – Optional prefix for S3 keys. The script joins this prefix with
  the file name returned by the database.
- `FILE_QUERY` – Query used to obtain file names (defaults to a query returning
  `file_name`).
- `FILE_NAME_COLUMN` – Name of the column in the query results that stores the
  file name (defaults to `file_name`).
- `EFS_PATH_COLUMN` – Optional column that stores an explicit EFS path. When not
  provided, the file name is resolved relative to `EFS_BASE_PATH`.
- `EFS_BASE_PATH` – Base directory inside the EFS mount that contains the files
  (defaults to `/efs`).
- `LOG_LEVEL` – Logging level for the Python script (defaults to `INFO`).

## IAM considerations

Grant the service account IAM permissions to:

- Read objects and object metadata from the target S3 bucket (`s3:GetObject`
  and `s3:ListBucket` to scope to specific prefixes).
- Read and write to the EFS access point via the EFS CSI driver.

## Database connectivity

The cleanup job uses the [psycopg](https://www.psycopg.org/psycopg3/docs/)
client to connect to PostgreSQL-compatible databases. Ensure that the
Kubernetes nodes can reach the database endpoint (for example through a VPC
peering connection or by running the workload in the same VPC as the database).
