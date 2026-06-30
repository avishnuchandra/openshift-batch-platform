Act as a Senior Cloud-Native Java Architect. Design and generate the complete source code for a multi-module, production-grade Spring Batch platform orchestrated by Kubernetes/OpenShift.

**Tech Stack & Versions:**
* Java 21
* Spring Boot 3.5.14
* Spring Framework 6.2.14
* Spring Batch 5.x (Must use Spring Batch 5 standards; do NOT use the deprecated `JobBuilderFactory` or `StepBuilderFactory`)
* Fabric8 Kubernetes Client 6.10.0
* AWS SDK v2 (S3) 2.20.162
* Gradle (Multi-module setup)

**Project Structure:**
The project must be named `openshift-batch-platform` with the following modules:
1. `common-core`: Contains JPA entities, repositories, and S3 interfaces.
2. `s3-import-job`: An OpenShift `CronJob` that acts as the orchestrator.
3. `file-processing-job`: A dynamic Kubernetes `Job` that executes the Spring Batch chunk processing.

**Architectural Rules:**
* **NO `@Scheduled` annotations.** All scheduling is handed off to OpenShift `CronJobs`.
* **Profiles:** Implement two Spring profiles. `local` uses an in-memory H2 database, a local directory to mock the PVC, and skips the actual K8s Job creation. `openshift` uses PostgreSQL, a real `ReadWriteMany` PVC, and executes the Fabric8 K8s client.

**Module 1: `s3-import-job` (Orchestrator Workflow & Fortifications)**
This is a run-to-completion Spring Boot app triggered by a K8s CronJob. It must execute the following steps in order:
1. **Reaper Mechanism:** Scan the `file_metadata` database table for records stuck in `PROCESSING` for more than 2 hours and update them to `FAILED` (to prevent zombie deadlocks).
2. **PVC Garbage Collection:** Scan the local PVC mount path (`/mnt/moss-data`) and delete any physical files that do not have a corresponding `PROCESSING` record in the database (to prevent storage exhaustion).
3. **Pessimistic Lock:** Check if any file is currently `PROCESSING`. You MUST use `@Lock(LockModeType.PESSIMISTIC_WRITE)` on the Spring Data JPA repository query to prevent cluster race conditions. If a file is processing, exit gracefully.
4. **Acquire:** List files from S3 (`input/`), take the oldest, and skip if it already exists in the database.
5. **Download:** Download the file from S3 to the local PVC mount path.
6. **Persist:** Save a new record to the database with a generated UUID (`fileId`), `fileName`, `correlationId`, and status `PROCESSING`.
7. **Trigger K8s Job:** Use the Fabric8 client to dynamically create a Kubernetes Job for the `file-processing-job`. 
   * *CRITICAL:* Generate the K8s Job name using a SHA-256 hash of the `fileId` truncated to 12 characters (e.g., `fp-<hash>`) to strictly comply with the RFC 1123 63-character DNS limit.
   * Mount the PVC volume into the new Pod.
   * Pass `FILE_ID`, `FILE_NAME`, `CORRELATION_ID`, and `LOCAL_FILE_PATH` as container environment variables.

**Module 2: `file-processing-job` (Spring Batch Processor)**
This is a Spring Batch application executed by the dynamic K8s Job.
1. It must read the `FILE_ID`, `FILE_NAME`, and `LOCAL_FILE_PATH` from the environment variables.
2. Use a `FlatFileItemReader` to read the file *directly from the PVC file system*, NOT from S3.
3. Configure a persistent `JobRepository` so Kubernetes can handle pod-level retries if a crash occurs.
4. Implement a `JobExecutionListener`:
   * On `COMPLETED`: Update DB status to COMPLETED, move S3 file to `success/`.
   * On `FAILED`: Update DB status to FAILED, move S3 file to `failed/`.
   * `afterJob` MUST physically delete the file from the PVC to clean up storage.

**Deliverables Required:**
Please generate the exact code for:
1. The root and module `build.gradle` files.
2. `application.yml` for both apps covering `local` and `openshift` profiles.
3. The `FileMetadata` Entity and `FileMetadataRepository` (with the pessimistic lock).
4. `S3ImportOrchestrator.java` (implementing the Reaper, GC, lock, and workflow).
5. `KubernetesJobLauncher.java` (Fabric8 implementation with SHA-256 naming).
6. `FileProcessingBatchConfig.java` (Spring Batch 5 configuration).
7. The Kubernetes YAML manifests for the `ReadWriteMany` PVC and the RBAC `ServiceAccount`/`Role` needed by Fabric8 to spawn jobs.
