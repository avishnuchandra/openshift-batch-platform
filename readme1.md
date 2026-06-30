# OpenShift Spring Batch Platform

A cloud-native, multi-module Spring Boot (3.5.14) and Spring Batch (5.x) architecture utilizing Java 21. This platform decouples file acquisition from batch processing by leveraging OpenShift `CronJobs` and dynamic Kubernetes `Jobs` for highly scalable, lock-free execution.

## 🏗 Architecture Overview

This project replaces traditional, always-on JVM polling (`@Scheduled`) with a Kubernetes-native orchestration approach.

1. **`s3-import-job` (Orchestrator)**: Runs as an OpenShift `CronJob`. It acts as a lightweight adapter that polls AWS S3, persists processing metadata to a relational database, downloads the file to a shared `ReadWriteMany` PVC, and uses the Fabric8 client to dynamically trigger the downstream processing pod.
2. **`file-processing-job` (Processor)**: Runs as a dynamic Kubernetes `Job`. It mounts the shared PVC, reads the local file using Spring Batch chunk processing, manages state via a persistent `JobRepository`, and updates the database and S3 status upon completion before cleaning up the PVC.

## 🛡️ Resilience & Security Features (Fortified)

To operate safely in a distributed cluster environment, this architecture includes several fail-safes:

* **The Reaper Mechanism:** Automatically scans for and fails database records stuck in `PROCESSING` for more than 2 hours to prevent cluster-wide deadlocks from silently crashed pods.
* **Pessimistic Locking:** Uses `@Lock(LockModeType.PESSIMISTIC_WRITE)` to prevent race conditions if multiple orchestrator pods are accidentally spun up simultaneously.
* **PVC Garbage Collection:** A pre-flight routine that deletes orphaned files on the shared volume, preventing storage exhaustion.
* **Safe K8s DNS Naming:** Uses a SHA-256 hashing algorithm to guarantee that dynamically generated Kubernetes Job names strictly adhere to the RFC 1123 63-character limit and character restrictions.

## 📦 Module Breakdown

* `common-core`: Shared JPA Entities (`FileMetadata`), Repositories, S3 interfaces, and DTOs.
* `s3-import-job`: The CronJob orchestrator containing the Fabric8 API launcher and resilience routines.
* `file-processing-job`: The Spring Batch processor containing the `JobRepository`, `FlatFileItemReader`, and chunk processing logic.

---

## 💻 Local Development & Testing

The project is pre-configured to run seamlessly on a local workstation without requiring an active Kubernetes cluster or AWS credentials. The `local` Spring profile utilizes a shared file-based H2 database and local directories to simulate the OpenShift PVC.

### 1. Run the Orchestrator
Execute the `S3ImportApplication.java` main method.
* **Action:** It creates the H2 database at `./local-h2-db/batchdb`, simulates downloading a file to `./local-temp-storage`, writes a `PROCESSING` lock to the DB, and gracefully exits.
* **Note:** Because the `local` profile is active, it will *not* attempt to contact the Kubernetes API. It will print the required environment variables to the console for the next step.

### 2. Run the Batch Processor
Execute the `FileProcessingApplication.java` main method.
* **Action:** Spring Batch connects to the *same* local H2 database, reads the file from `./local-temp-storage`, processes the data chunks, marks the database record as `COMPLETED`, and deletes the physical file.

---

## 🚀 OpenShift Deployment

To deploy this architecture to an OpenShift cluster using GitOps (e.g., ArgoCD), the `SPRING_PROFILES_ACTIVE=openshift` profile must be injected.

### Prerequisites
1. **Container Registry:** Build the Docker/OCI images for `s3-import-job` and `file-processing-job` and push them to your registry (e.g., Quay, AWS ECR).
2. **Database:** A PostgreSQL instance accessible from the OpenShift cluster.
3. **AWS S3:** Valid AWS IAM credentials with read/write access to the target buckets.

### Kubernetes Manifests Required
The orchestrator requires specific cluster configurations to operate:
1. **`ReadWriteMany` PVC:** A persistent volume (e.g., CephFS, NFS) mounted to `/mnt/moss-data` to share files between the CronJob and the dynamic Job.
2. **ServiceAccount & RBAC:** A `ServiceAccount` (e.g., `batch-executor-sa`) bound to a `Role` with `create`, `get`, `list`, `watch`, and `delete` permissions for `jobs` in the `batch` API group.
3. **Secrets:** Standard Kubernetes secrets holding `DB_URL`, `DB_USERNAME`, `DB_PASSWORD`, `AWS_ACCESS_KEY_ID`, and `AWS_SECRET_ACCESS_KEY`.

### Environment Variables

| Variable | Description | Target Module |
| :--- | :--- | :--- |
| `SPRING_PROFILES_ACTIVE` | Set to `openshift` for production. | All |
| `PVC_MOUNT_PATH` | The path where the shared volume is mounted (e.g., `/mnt/moss-data`). | `s3-import-job` |
| `FILE_ID` | UUID passed dynamically from the orchestrator. | `file-processing-job` |
| `FILE_NAME` | The name of the file to process. | `file-processing-job` |
| `LOCAL_FILE_PATH` | The absolute path to the file on the PVC. | `file-processing-job` |
