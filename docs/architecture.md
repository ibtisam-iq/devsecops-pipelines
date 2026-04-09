# DevSecOps Pipelines — Architecture

## Problem Statement

How can we design a reusable DevSecOps system that works across different applications without coupling pipelines together?

---

## Core Challenge

When working with multiple applications, there are two possible approaches:

### 1. Centralized Pipeline (Rejected)

* Single Docker config
* Single scanning config
* Shared pipeline logic

Problem:

* Tight coupling
* One change breaks multiple pipelines
* Difficult to debug

---

### 2. Self-Contained Pipelines (Chosen Approach)

Each application has its own:

* Pipeline configuration
* Dockerfile
* Security scanning setup

Benefits:

* Isolation
* Reliability
* Clear ownership

---

## Final Architecture

```text
devsecops-pipelines/
│
├── pipelines/
│   ├── java-monolith/
│   │   ├── app/        (submodule)
│   │   ├── Jenkinsfile
│   │   ├── docker/
│   │   ├── sonar/
│   │   └── trivy/
│   │
│   ├── node-monolith/
│   └── python-monolith/
```

---

## Submodule Strategy

Each pipeline uses a submodule:

* Keeps application code separate
* Avoids duplication
* Allows independent updates

---

## Execution Flow

Pipeline runs as follows:

1. Fetch code (submodule)
2. Build application
3. Run code quality checks
4. Perform security scanning
5. Build Docker image
6. Push artifact

---

## Design Decisions

### Decision 1: No Domain-Based Naming

* Applications are treated as generic inputs
* Focus is on pipeline capability

---

### Decision 2: No Centralized Docker/Trivy/Sonar

* Avoids cross-pipeline breakage
* Keeps pipelines independent

---

### Decision 3: One Repository for Pipelines

* Shows capability across multiple systems
* Keeps portfolio clean and focused

---

## What This Proves

This architecture demonstrates:

* Ability to design scalable pipeline systems
* Understanding of CI/CD separation
* Awareness of coupling vs isolation trade-offs

---

## Conclusion

This repository is not about applications.

It is about:

> Designing repeatable, secure, and independent DevSecOps pipelines for any system.
