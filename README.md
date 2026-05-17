# DevOps + Jenkins Learning Repository

> A practical DevOps learning repository where I document everything I learn while working as a DevOps Engineer — from Jenkins basics to production-grade CI/CD architectures.

---

# About This Repository

This repository is my personal DevOps knowledge base.

Instead of learning only theory, this repo focuses on:
- real-world DevOps workflows
- Jenkins pipelines
- CI/CD architecture
- Git branching strategies
- production deployment concepts
- troubleshooting
- automation
- infrastructure practices

The goal is to:
- revise concepts quickly in the future
- create structured notes
- help junior engineers learn DevOps
- document production-level understanding

---

# Repository Structure

```text
.
├── cicd-basics
├── jenkins-basics-installation
├── jenkins2
├── multibranch
├── pipelines
├── production-grade-jenkins-final
└── README.md
```

---

# Folder Overview

| Folder | Description |
|---|---|
| `cicd-basics` | CI/CD fundamentals, GitFlow, Trunk-Based Development, artifact promotion, deployment strategies |
| `jenkins-basics-installation` | Jenkins installation, setup, configuration, and basic usage |
| `jenkins2` | Additional Jenkins concepts, experiments, and advanced notes |
| `multibranch` | Jenkins Multibranch Pipeline concepts and setup |
| `pipelines` | Jenkins pipelines, stages, agents, declarative/scripted pipelines |
| `production-grade-jenkins-final` | Real-world production Jenkins architecture and enterprise-level setup |

---

# What You Will Learn

This repository covers:

## CI/CD Concepts
- Continuous Integration
- Continuous Delivery
- Continuous Deployment
- Build Once Promote Many
- Artifact Promotion
- Immutable Deployments
- Deployment Strategies

---

## Jenkins
- Jenkins installation
- Jenkins architecture
- Jenkins pipelines
- Jenkins agents
- Jenkinsfiles
- Shared libraries
- Multibranch pipelines
- Production Jenkins setup

---

## Git & Branching
- GitFlow
- Trunk-Based Development
- Feature branches
- Promotion PRs
- Protected branches
- Merge strategies

---

## Deployments
- Dev → Stage → Prod promotions
- Canary deployments
- Blue-Green deployments
- Rollbacks
- Smoke testing
- Production safety checks

---

## DevOps Engineering Practices
- Automation
- CI/CD pipeline design
- Infrastructure thinking
- Release engineering
- Production readiness
- Reliability concepts

---

# CI/CD High-Level Flow

```mermaid
flowchart LR

A[Developer Push] --> B[Pull Request]

B --> C[CI Pipeline]

C --> D[Build]

D --> E[Test]

E --> F[Security Scan]

F --> G[Merge]

G --> H[Build Artifact]

H --> I[Push To Registry]

I --> J[Deploy Dev]

J --> K[Promote Stage]

K --> L[Stage Validation]

L --> M[Promote Production]

M --> N[Canary Rollout]

N --> O[Monitoring]
```

---

# Repository Philosophy

This repository focuses on:
- learning by doing
- understanding concepts deeply
- documenting real workflows
- production-oriented thinking

This is NOT:
- copy-paste tutorial notes
- certification dumps
- only theoretical explanations

The aim is to understand:
> WHY systems are designed in a certain way.

---

# Recommended Learning Order

If you are a beginner, follow this order:

```text
1. jenkins-basics-installation
2. pipelines
3. multibranch
4. cicd-basics
5. production-grade-jenkins-final
```

---

# Example Topics Covered

- What happens when a PR is opened?
- How Jenkins triggers builds
- Why CI runs on merge results
- Difference between CI/CD/CDp
- Why deployments should use digests
- How promotion pipelines work
- Why production rollouts are gradual
- How enterprise Jenkins is designed

---

# Technologies & Tools

This repository may include concepts around:

- Jenkins
- Git
- GitHub
- Docker
- Kubernetes
- Helm
- Linux
- Bash
- CI/CD
- GitOps
- DevSecOps

---

# Future Additions

Planned topics:

- Kubernetes internals
- Helm deep dive
- ArgoCD
- Terraform
- Monitoring & Observability
- Prometheus
- Grafana
- Service Mesh
- SRE concepts
- Infrastructure as Code
- Platform Engineering
- Incident Handling
- Scaling Jenkins
- Secure CI/CD

---

# Why I Built This Repository

As a DevOps Engineer, I realized:

Most people learn tools.
Very few understand systems.

This repository is my effort to document:
- practical learning
- architecture thinking
- production engineering concepts
- DevOps mindset

---

# Important Mindset

Tools change.

Concepts remain valuable.

A strong DevOps Engineer should understand:
- systems
- workflows
- reliability
- automation
- scalability
- deployment safety
- infrastructure design

not just buttons in tools.

---

# Contributions

This is primarily a personal learning repository, but:
- suggestions
- corrections
- improvements
- discussions

are always welcome.

---

# Final Goal

The long-term goal of this repository is to become:

> A complete practical DevOps knowledge base from beginner to production-grade engineering.

---