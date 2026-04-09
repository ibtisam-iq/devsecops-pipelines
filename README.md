# DevSecOps Pipelines

## Overview

This repository contains multiple DevSecOps pipeline implementations applied to different application codebases.

The purpose is not to build applications, but to demonstrate how real-world applications can be taken as input and processed through secure, automated delivery pipelines.

---

## What This Repository Demonstrates

* Building pipelines around existing codebases
* CI workflows using Jenkins and GitHub Actions
* Security integration (Trivy, SonarQube)
* Docker-based artifact creation
* Standardized pipeline design across different stacks

---

## Repository Structure

```text
devsecops-pipelines/
│
├── pipelines/
│   ├── java-monolith/
│   ├── node-monolith/
│   └── python-monolith/
│
├── shared/
├── docs/
└── README.md
```

---

## Key Design Principle

Each pipeline is **self-contained**.

This means:

* Each pipeline has its own Dockerfile
* Each pipeline has its own scanning configuration
* Each pipeline can run independently

This avoids cross-dependency and ensures that changes in one pipeline do not break others.

---

## Why Not Centralized Configuration?

A common approach is to centralize tools like Docker, Trivy, or SonarQube at the root level.

This repository intentionally avoids that.

Instead, each pipeline is isolated to:

* Improve reliability
* Avoid tight coupling
* Maintain clear ownership

---

## Pipeline Philosophy

This repository follows a simple principle:

> Any application can be taken as input and transformed into a secure, deployable artifact using a consistent DevSecOps pipeline.

---

## Architecture

For detailed architecture and design decisions:

👉 See: `docs/architecture.md`

---

## Pipelines Included

### Java Monolith

* Spring Boot application
* Maven build
* Docker image creation
* Security scanning

(More pipelines will be added over time)

---

## Author

Muhammad Ibtisam Iqbal

DevOps Engineer | Cloud Infrastructure | Kubernetes (CKA, CKAD)

