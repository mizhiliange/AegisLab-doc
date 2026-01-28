---
title: Prerequisites
weight: 2
---

> [!NOTE]
> **Version Compatibility**: This guide is tested with Kubernetes v1.28+ and Helm 3.13+. Older versions may work but are not officially supported.

Before deploying AegisLab, ensure the following prerequisites are met:

## Software Requirements

| Tool     | Version                      | Installation                                                                                     |
| -------- | ---------------------------- | ------------------------------------------------------------------------------------------------ |
| devbox   | Latest                       | `curl -fsSL https://nix-community.org/detect-devbox.sh`                                          |
| Docker   | 20.10+                       | `apt-get install docker.io`                                                                      |
| Helm     | 3.x                          | `curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3`                       |
| kubectl  | Match K8s version            | Follow Kubernetes docs                                                                           |
| kubectx  | Latest                       | `git clone https://github.com/ahmetb/kubectx $HOME/kubectx`                                      |
| Skaffold | Latest                       | `curl -Lo skaffold https://storage.googleapis.com/skaffold/releases/latest/skaffold-linux-amd64` |
| Kind     | Latest (for local testing)   | `go install sigs.k8s.io/kind@latest`                                                             |
| Minikube | Latest (alternative to Kind) | Follow Minikube docs                                                                             |

## Kubernetes Cluster Requirements

- **Version**: v1.28 or later
- **Resources**:
  - Development: 8 CPU, 16GB RAM minimum
  - Production: 16 CPU, 32GB RAM minimum
- **Capabilities**: StatefulSets, PVCs, CRDs (Chaos Mesh)
- **Network**: Pod-to-Pod communication, Service exposure
