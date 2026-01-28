---
title: Deployment Guide
weight: 4
---

Complete guide for deploying the AegisLab ecosystem components.

## Quick Navigation

{{< cards >}}
{{< card link="overview" title="Overview" icon="information-circle" >}}
{{< card link="prerequisites" title="Prerequisites" icon="check-circle" >}}
{{< card link="dependencies" title="Dependencies" icon="puzzle" >}}
{{< card link="workflow" title="Deployment Workflow" icon="play" >}}
{{< card link="verification" title="Verification & Troubleshooting" icon="beaker" >}}
{{< /cards >}}

## Deployment Modes

AegisLab supports four deployment modes:

- **Debug Mode**: Local development with Docker + Kubernetes
- **Test Mode**: CI/CD testing with Kind cluster
- **Staging Mode**: Team collaboration on remote Kubernetes
- **Production Mode**: Production deployment with Helm

Choose the appropriate mode based on your use case and follow the detailed guides in each section.
