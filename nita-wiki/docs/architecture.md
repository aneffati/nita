---
title: Architecture
description: System architecture, component interactions, and data flows in NITA
---

# :material-sitemap: System Architecture

This page provides a comprehensive view of how NITA is designed, how its components interact, and how data flows through the platform during build and test operations.

---

## High-Level Architecture

NITA runs as a set of Kubernetes pods within the `nita` namespace. Four pods run persistently, while Ansible and Robot containers are launched on-demand by Jenkins as ephemeral Kubernetes jobs.

```mermaid
graph TB
    subgraph "Host Machine"
        subgraph "Kubernetes Cluster"
            subgraph "nita namespace"
                PROXY["🛡️ Nginx Proxy<br/>Pod"]
                WEBAPP["🌐 NITA Webapp<br/>Pod (Django)"]
                JENKINS["⚙️ Jenkins<br/>Pod"]
                DB["🗄️ MariaDB<br/>Pod"]

                subgraph "Ephemeral Jobs"
                    ANSIBLE["📦 Ansible<br/>Container"]
                    ROBOT["🧪 Robot Framework<br/>Container"]
                end
            end

            PV1["💾 PV: jenkins-home<br/>20Gi"]
            PV2["💾 PV: mariadb<br/>2Gi"]
        end

        FS["📂 /var/nita_project<br/>Host Filesystem"]
    end

    USER["👤 User"] -->|"HTTPS :443"| PROXY
    PROXY -->|"HTTP :8000"| WEBAPP
    PROXY -->|"HTTPS :8443"| JENKINS
    WEBAPP -->|"SQL :3306"| DB
    JENKINS -->|"K8s Job API"| ANSIBLE
    JENKINS -->|"K8s Job API"| ROBOT
    JENKINS --- PV1
    DB --- PV2
    JENKINS --- FS
    ANSIBLE --- FS
    ROBOT --- FS

    ANSIBLE -->|"Netconf / SSH"| DEVICES["🖧 Network<br/>Devices"]
    ROBOT -->|"SSH / Netconf"| DEVICES

    style PROXY fill:#1565C0,color:#fff
    style WEBAPP fill:#1976D2,color:#fff
    style JENKINS fill:#1E88E5,color:#fff
    style DB fill:#2196F3,color:#fff
    style ANSIBLE fill:#0D47A1,color:#fff
    style ROBOT fill:#0D47A1,color:#fff
```

---

## Component Overview

| Component | Image | Type | Port(s) | Purpose |
|---|---|---|---|---|
| **Nginx Proxy** | `nginx:stable` | Persistent | 443 (HTTPS) | TLS termination, reverse proxy |
| **NITA Webapp** | `juniper/nita-webapp:25.10-1` | Persistent | 8000 | Web UI, REST API, project management |
| **Jenkins** | `juniper/nita-jenkins:23.12-1` | Persistent | 8443, 8080 | Job orchestration, pipeline execution |
| **MariaDB** | `mariadb:10.4.12` | Persistent | 3306 | Relational database (Django backend) |
| **Ansible** | `juniper/nita-ansible:22.8-1` | Ephemeral | — | Configuration management |
| **Robot** | `juniper/nita-robot:22.8-1` | Ephemeral | — | Automated testing |

---

## Network Topology

All pods communicate within the Kubernetes cluster network managed by **Calico**. Only the Nginx proxy is exposed externally.

```mermaid
graph LR
    subgraph "External Access"
        BROWSER["🌐 Browser"]
    end

    subgraph "Kubernetes Cluster Network (Calico)"
        PROXY["Nginx Proxy<br/>:443 ↔ External"]
        WEBAPP["Webapp<br/>ClusterIP :8000"]
        JENKINS["Jenkins<br/>ClusterIP :8443/:8080"]
        DB["MariaDB<br/>ClusterIP :3306"]
    end

    BROWSER -->|"HTTPS :443"| PROXY
    PROXY -->|"→ /jenkins/*"| JENKINS
    PROXY -->|"→ /*"| WEBAPP
    WEBAPP --> DB
```

!!! note "Service Types"
    All internal services use **ClusterIP** — they are only accessible within the cluster. The Nginx proxy pod uses **hostPort 443** to expose NITA to the host network.

---

## Data Flow: Build Process

When a user triggers a **Build** action from the Webapp, the following sequence occurs:

```mermaid
sequenceDiagram
    participant User
    participant Webapp as NITA Webapp
    participant DB as MariaDB
    participant Jenkins
    participant Ansible as Ansible (K8s Job)
    participant Devices as Network Devices

    User->>Webapp: Trigger "Build" action
    Webapp->>DB: Retrieve network config data
    Webapp->>Jenkins: Trigger Jenkins job
    Jenkins->>Jenkins: Write YAML config files
    Jenkins->>Ansible: Launch K8s Job with Ansible container
    Ansible->>Ansible: Execute Ansible playbooks
    Ansible->>Devices: Push configurations (Netconf/SSH)
    Devices-->>Ansible: Confirm changes
    Ansible-->>Jenkins: Job completes (exit code)
    Jenkins-->>Webapp: Report build status
    Webapp-->>User: Display results
```

---

## Data Flow: Test Process

When a user triggers a **Test** action, Robot Framework validates the network state:

```mermaid
sequenceDiagram
    participant User
    participant Webapp as NITA Webapp
    participant Jenkins
    participant Robot as Robot Framework (K8s Job)
    participant Devices as Network Devices

    User->>Webapp: Trigger "Test" action
    Webapp->>Jenkins: Trigger Jenkins job
    Jenkins->>Robot: Launch K8s Job with Robot container
    Robot->>Devices: Execute test commands (SSH/Netconf)
    Devices-->>Robot: Return command output
    Robot->>Robot: Compare output against expected state
    Robot-->>Jenkins: Return test results
    Jenkins-->>Webapp: Report test status
    Webapp-->>User: Display pass/fail results
```

---

## Storage Architecture

NITA uses a combination of Kubernetes Persistent Volumes and host filesystem mounts:

```mermaid
graph TB
    subgraph "Persistent Volumes"
        PV1["PV: task-pv-volume<br/>20Gi, RWO, manual"]
        PV2["PV: pv-volume<br/>2Gi, RWO, manual"]
    end

    subgraph "Persistent Volume Claims"
        PVC1["PVC: jenkins-home<br/>Bound → task-pv-volume"]
        PVC2["PVC: mariadb<br/>Bound → pv-volume"]
    end

    subgraph "Host Path"
        HP["/var/nita_project<br/>Project files"]
    end

    PV1 --> PVC1
    PV2 --> PVC2
    PVC1 --> JENKINS["Jenkins Pod"]
    PVC2 --> DB["MariaDB Pod"]
    HP --> JENKINS
    HP --> ANSIBLE["Ansible Jobs"]
    HP --> ROBOT["Robot Jobs"]
```

| Storage | Size | Access | Used By | Purpose |
|---|---|---|---|---|
| `jenkins-home` PVC | 20 Gi | ReadWriteOnce | Jenkins | Jenkins home directory, job configs, plugins |
| `mariadb` PVC | 2 Gi | ReadWriteOnce | MariaDB | Database files |
| `/var/nita_project` | Host | ReadWrite | Jenkins, Ansible, Robot | Shared project files |

---

## ConfigMaps

NITA uses Kubernetes ConfigMaps to inject certificates and proxy configuration:

| ConfigMap | Contents | Used By |
|---|---|---|
| `jenkins-crt` | Jenkins SSL certificate | Jenkins pod |
| `jenkins-keystore` | Jenkins keystore (JKS) | Jenkins pod |
| `proxy-config-cm` | Nginx configuration | Proxy pod |
| `proxy-cert-cm` | Nginx TLS certificates | Proxy pod |

---

## RBAC & Security

Jenkins requires Kubernetes API access to launch ephemeral Ansible and Robot jobs:

```mermaid
graph LR
    SA["ServiceAccount:<br/>internal-jenknis-pod"] --> RB["RoleBinding"]
    RB --> ROLE["Role: modify-pods<br/>(create, delete, get pods/jobs)"]
    ROLE --> API["Kubernetes API"]
    JENKINS["Jenkins Pod"] --> SA
```

| Resource | Name | Purpose |
|---|---|---|
| ServiceAccount | `internal-jenknis-pod` | Identity for Jenkins pod |
| Role | `modify-pods` | Permissions to manage pods and jobs |
| RoleBinding | — | Binds ServiceAccount to Role |

---

## Technology Stack Summary

```mermaid
mindmap
  root((NITA Platform))
    Infrastructure
      Kubernetes 1.28
      Calico CNI
      containerd runtime
    Core Services
      Jenkins 23.12
      NITA Webapp (Django)
      MariaDB 10.4
      Nginx (reverse proxy)
    Automation Tools
      Ansible
      Robot Framework
      Jinja2 Templates
    Data & Config
      Excel Spreadsheets
      YAML Inventories
      Netconf / SSH
    CLI
      nita-cmd (Bash)
      kubectl
```
