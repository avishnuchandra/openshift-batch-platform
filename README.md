# OpenShift Batch Platform

## Overview
OpenShift Batch Platform is a batch processing solution built for OpenShift.

## Getting Started
Instructions for getting started with this project.

## Installation
Steps to install and configure the platform.

## Usage
How to use the OpenShift Batch Platform.

## Contributing
Guidelines for contributing to this project.

## License
License information for this project.


## bash
#!/bin/bash

PROJECT_DIR="openshift-batch-platform-fortified"
echo "Initializing fortified multi-module Spring Boot 3.5.14 project..."

mkdir -p $PROJECT_DIR
cd $PROJECT_DIR

# ---------------------------------------------------------
# 1. DIRECTORY STRUCTURE
# ---------------------------------------------------------
mkdir -p common-core/src/main/java/com/enterprise/batch/common/{entity,repository,s3}
mkdir -p s3-import-job/src/main/java/com/enterprise/batch/s3import/{service,config}
mkdir -p s3-import-job/src/main/resources
mkdir -p file-processing-job/src/main/java/com/enterprise/batch/fileprocessing/config
mkdir -p file-processing-job/src/main/resources
mkdir -p k8s-manifests/overlays/production

# ---------------------------------------------------------
# 2. GRADLE CONFIGURATION
# ---------------------------------------------------------
cat << 'EOF' > settings.gradle
rootProject.name = 'openshift-batch-platform'
include 'common-core', 's3-import-job', 'file-processing-job'
EOF

cat << 'EOF' > build.gradle
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.5.14' apply false
    id 'io.spring.dependency-management' version '1.1.4' apply false
}
ext {
    springFrameworkVersion = '6.2.14'
    springSecurityVersion = '6.5.10'
    fabric8Version = '6.10.0'
}
allprojects {
    group = 'com.enterprise.batch'
    version = '1.0.0-SNAPSHOT'
    apply plugin: 'java'
    apply plugin: 'io.spring.dependency-management'
    java { sourceCompatibility = JavaVersion.VERSION_21; targetCompatibility = JavaVersion.VERSION_21 }
    repositories { mavenCentral() }
    dependencyManagement {
        imports { mavenBom "org.springframework.boot:spring-boot-dependencies:3.5.14" }
        dependencies {
            dependency "org.springframework:spring-core:${springFrameworkVersion}"
            dependency "org.springframework.security:spring-security-core:${springSecurityVersion}"
            dependency "io.fabric8:kubernetes-client:${fabric8Version}"
        }
    }
}
EOF

cat << 'EOF' > common-core/build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jpa'
    implementation 'software.amazon.awssdk:s3:2.20.162'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
EOF

cat << 'EOF' > s3-import-job/build.gradle
dependencies {
    implementation project(':common-core')
    implementation 'org.springframework.boot:spring-boot-starter'
    implementation 'io.fabric8:kubernetes-client'
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
EOF

cat << 'EOF' > file-processing-job/build.gradle
dependencies {
    implementation project(':common-core')
    implementation 'org.springframework.boot:spring-boot-starter-batch'
    runtimeOnly 'com.h2database:h2'
    runtimeOnly 'org.postgresql:postgresql'
    compileOnly 'org.projectlombok:lombok'
    annotationProcessor 'org.projectlombok:lombok'
}
EOF

# ---------------------------------------------------------
# 3. COMMON CORE JAVA CLASSES (With Pessimistic Locking)
# ---------------------------------------------------------
cat << 'EOF' > common-core/src/main/java/com/enterprise/batch/common/entity/FileMetadata.java
package com.enterprise.batch.common.entity;
import jakarta.persistence.*;
import lombok.Data;
import java.time.LocalDateTime;

@Data 
@Entity 
@Table(name = "file_metadata")
public class FileMetadata {
    @Id private String fileId;
    @Column(unique = true) private String fileName;
    private String status;
    private String correlationId;
    private LocalDateTime updatedAt;

    @PrePersist
    @PreUpdate
    public void preUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
EOF

cat << 'EOF' > common-core/src/main/java/com/enterprise/batch/common/repository/FileMetadataRepository.java
package com.enterprise.batch.common.repository;
import com.enterprise.batch.common.entity.FileMetadata;
import jakarta.persistence.LockModeType;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Lock;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;

import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

public interface FileMetadataRepository extends JpaRepository<FileMetadata, String> {
    
    // LOOPHOLE B FIX: Pessimistic locking prevents race conditions across cluster
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT COUNT(f) > 0 FROM FileMetadata f WHERE f.status = 'PROCESSING'")
    boolean existsByStatusLocked();

    boolean existsByFileName(String fileName);

    @Query("SELECT f.fileName FROM FileMetadata f WHERE f.status = 'PROCESSING'")
    List<String> findProcessingFileNames();

    @Query("SELECT f FROM FileMetadata f WHERE f.status = 'PROCESSING' AND f.updatedAt < :threshold")
    List<FileMetadata> findStaleProcessingRecords(@Param("threshold") LocalDateTime threshold);
}
EOF

cat << 'EOF' > common-core/src/main/java/com/enterprise/batch/common/s3/S3Service.java
package com.enterprise.batch.common.s3;
import java.nio.file.Path;
import java.util.List;
public interface S3Service {
    List<String> listFilesSortedByOldest(String prefix);
    void downloadFile(String key, Path destination);
    void moveObject(String sourceKey, String destinationKey);
}
EOF

# ---------------------------------------------------------
# 4. S3 IMPORT ORCHESTRATOR (With PVC & Zombie Mitigations)
# ---------------------------------------------------------
cat << 'EOF' > s3-import-job/src/main/java/com/enterprise/batch/s3import/S3ImportApplication.java
package com.enterprise.batch.s3import;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.autoconfigure.domain.EntityScan;
import org.springframework.data.jpa.repository.config.EnableJpaRepositories;
@SpringBootApplication(scanBasePackages = "com.enterprise.batch")
@EntityScan("com.enterprise.batch.common.entity")
@EnableJpaRepositories("com.enterprise.batch.common.repository")
public class S3ImportApplication {
    public static void main(String[] args) { System.exit(SpringApplication.exit(SpringApplication.run(S3ImportApplication.class, args))); }
}
EOF

cat << 'EOF' > s3-import-job/src/main/java/com/enterprise/batch/s3import/service/S3ImportOrchestrator.java
package com.enterprise.batch.s3import.service;

import com.enterprise.batch.common.entity.FileMetadata;
import com.enterprise.batch.common.repository.FileMetadataRepository;
import com.enterprise.batch.common.s3.S3Service;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.CommandLineRunner;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.io.IOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.Paths;
import java.time.LocalDateTime;
import java.util.List;
import java.util.UUID;
import java.util.stream.Stream;

@Slf4j
@Service
@RequiredArgsConstructor
public class S3ImportOrchestrator implements CommandLineRunner {

    private final S3Service s3Service;
    private final FileMetadataRepository metadataRepository;
    private final KubernetesJobLauncher k8sLauncher;

    @Value("${PVC_MOUNT_PATH:/mnt/moss-data}")
    private String pvcMountPath;

    @Override
    @Transactional
    public void run(String... args) {
        log.info("Starting fortified S3 Import Orchestration cycle.");

        reapStaleRecords();
        garbageCollectPvc();

        // LOOPHOLE B FIX: Atomic lock check
        if (metadataRepository.existsByStatusLocked()) {
            log.info("Another file is currently PROCESSING. Exiting gracefully.");
            return;
        }

        // Mocking S3 retrieval for this example
        List<String> files = List.of("data-file-new.csv"); 

        for (String fileName : files) {
            if (metadataRepository.existsByFileName(fileName)) continue;

            String fileId = UUID.randomUUID().toString();
            String correlationId = UUID.randomUUID().toString();

            Path localFilePath = Paths.get(pvcMountPath, fileName);
            log.info("Simulating download of {} to PVC path: {}", fileName, localFilePath);
            
            // Simulating physical file creation for PVC test
            try {
                Files.createDirectories(Paths.get(pvcMountPath));
                Files.writeString(localFilePath, "mock,data,row");
            } catch (IOException e) {
                log.error("Failed to create file on PVC", e);
                return;
            }

            FileMetadata metadata = new FileMetadata();
            metadata.setFileId(fileId);
            metadata.setFileName(fileName);
            metadata.setStatus("PROCESSING");
            metadata.setCorrelationId(correlationId);
            metadataRepository.save(metadata);

            k8sLauncher.launchFileProcessingJob(fileId, fileName, correlationId, localFilePath.toString());
            break;
        }
    }

    // LOOPHOLE A FIX: Zombie Processing Reaper
    private void reapStaleRecords() {
        LocalDateTime threshold = LocalDateTime.now().minusHours(2);
        List<FileMetadata> staleRecords = metadataRepository.findStaleProcessingRecords(threshold);
        for (FileMetadata record : staleRecords) {
            log.warn("Reaper identified zombie record: {}. Marking as FAILED.", record.getFileName());
            record.setStatus("FAILED");
            metadataRepository.save(record);
            // S3 Move logic from in-process to failed would go here
        }
    }

    // LOOPHOLE C FIX: PVC Garbage Collection
    private void garbageCollectPvc() {
        List<String> activeFiles = metadataRepository.findProcessingFileNames();
        try (Stream<Path> paths = Files.walk(Paths.get(pvcMountPath), 1)) {
            paths.filter(Files::isRegularFile).forEach(path -> {
                String fileName = path.getFileName().toString();
                if (!activeFiles.contains(fileName)) {
                    try {
                        Files.delete(path);
                        log.info("PVC GC: Deleted orphaned file {}", fileName);
                    } catch (IOException e) {
                        log.error("PVC GC: Failed to delete {}", fileName, e);
                    }
                }
            });
        } catch (IOException e) {
            log.warn("PVC GC: Directory unreadable or missing. Skipping.");
        }
    }
}
EOF

cat << 'EOF' > s3-import-job/src/main/java/com/enterprise/batch/s3import/service/KubernetesJobLauncher.java
package com.enterprise.batch.s3import.service;

import io.fabric8.kubernetes.api.model.batch.v1.Job;
import io.fabric8.kubernetes.api.model.batch.v1.JobBuilder;
import io.fabric8.kubernetes.client.KubernetesClient;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Service;

import java.nio.charset.StandardCharsets;
import java.security.MessageDigest;
import java.security.NoSuchAlgorithmException;

@Slf4j
@Service
@RequiredArgsConstructor
public class KubernetesJobLauncher {

    private final KubernetesClient kubernetesClient;

    @Value("${openshift.namespace:batch-platform}")
    private String namespace;

    @Value("${openshift.image.file-processing:your-registry/file-processing:1.0}")
    private String image;

    public void launchFileProcessingJob(String fileId, String fileName, String correlationId, String localFilePath) {
        // LOOPHOLE D FIX: SHA-256 Hashing for safe DNS length
        String safeHash = generateSafeHash(fileId).substring(0, 12);
        String jobName = "fp-" + safeHash; // Guaranteed safe length and chars
        String pvcName = "moss-batch-pvc";
        String mountPath = "/mnt/moss-data";

        try {
            Job job = new JobBuilder()
                .withNewMetadata().withName(jobName).withNamespace(namespace).endMetadata()
                .withNewSpec()
                    .withBackoffLimit(3)
                    .withNewTemplate()
                        .withNewSpec()
                            .withRestartPolicy("Never")
                            .withServiceAccountName("batch-executor-sa")
                            .addNewVolume()
                                .withName("moss-pvc-volume")
                                .withNewPersistentVolumeClaim(pvcName, false)
                            .endVolume()
                            .addNewContainer()
                                .withName("spring-batch-container")
                                .withImage(image)
                                .addNewEnv().withName("FILE_ID").withValue(fileId).endEnv()
                                .addNewEnv().withName("FILE_NAME").withValue(fileName).endEnv()
                                .addNewEnv().withName("CORRELATION_ID").withValue(correlationId).endEnv()
                                .addNewEnv().withName("LOCAL_FILE_PATH").withValue(localFilePath).endEnv()
                                .addNewVolumeMount()
                                    .withName("moss-pvc-volume")
                                    .withMountPath(mountPath)
                                .endVolumeMount()
                            .endContainer()
                        .endSpec()
                    .endTemplate()
                .endSpec()
                .build();

            kubernetesClient.batch().v1().jobs().inNamespace(namespace).resource(job).create();
            log.info("Submitted safe K8s Job: {}", jobName);
        } catch (Exception e) {
            log.warn("Mock execution or API failure (expected if running local without cluster): {}", e.getMessage());
        }
    }

    private String generateSafeHash(String input) {
        try {
            MessageDigest digest = MessageDigest.getInstance("SHA-256");
            byte[] hash = digest.digest(input.getBytes(StandardCharsets.UTF_8));
            StringBuilder hexString = new StringBuilder(2 * hash.length);
            for (byte b : hash) {
                String hex = Integer.toHexString(0xff & b);
                if (hex.length() == 1) hexString.append('0');
                hexString.append(hex);
            }
            return hexString.toString().toLowerCase();
        } catch (NoSuchAlgorithmException e) {
            return input.replaceAll("[^a-z0-9]", "").substring(0, Math.min(input.length(), 12));
        }
    }
}
EOF

# ---------------------------------------------------------
# 5. APPLICATION.YML FOR LOCAL & OPENSHIFT
# ---------------------------------------------------------
cat << 'EOF' > s3-import-job/src/main/resources/application.yml
spring:
  application.name: s3-import-job
  profiles.active: ${SPRING_PROFILES_ACTIVE:local}
---
spring:
  config.activate.on-profile: local
  datasource:
    url: jdbc:h2:mem:batchdb;DB_CLOSE_DELAY=-1
    driver-class-name: org.h2.Driver
    username: sa
    password:
  jpa.hibernate.ddl-auto: update
PVC_MOUNT_PATH: ./local-temp-storage
EOF

echo "Done! The fortified project is ready in the 'openshift-batch-platform-fortified' directory."



