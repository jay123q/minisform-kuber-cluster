# minisform-kuber-cluster

## Overview

Single-node Kubernetes homelab cluster managed by **Flux CD v2.7.5**, running on a Minisforum mini PC. All cluster state is GitOps-driven from this repo. The cluster runs Kubernetes v1.35.0 on Fedora 43 Server (x86_64).

- **Node:** single control-plane node at 192.168.1.112
- **Container runtime:** containerd 2.1.6
- **Git source:** ssh://git@github.com/jay123q/minisform-kuber-cluster.git (branch: main)
- **Owner:** jay123q

---

## Repo Structure

```
.
├── .sops.yaml                          # SOPS encryption rules (age key, encrypted_regex for data/stringData only)
├── CLAUDE.md                           # This file — AI assistant context
├── charts/                             # Local Helm charts (versioned templates, not currently deployed via Flux)
│   ├── chart-version-0-0-1/
│   └── chart-version-0-0-2/
└── clusters/my-cluster/
    ├── kustomization.yaml              # TOP-LEVEL: root Kustomization that wires everything together
    ├── flux-system/
    │   ├── gotk-components.yaml        # Flux controllers (DO NOT EDIT — managed by Renovate for version bumps)
    │   ├── gotk-sync.yaml             # Flux self-management: GitRepository + Kustomization (includes SOPS decryption config)
    │   └── kustomization.yaml
    ├── apps/
    │   ├── hello-world/               # Simple test app (Deployment + Service + ConfigMap), LoadBalancer on 192.168.1.200
    │   └── minecraft-gitops.yaml      # GitRepository + 2 Kustomizations pointing to external repo (see Sub-Clusters below)
│   └── schizo-bot-gitops.yaml      # GitRepository for schizo-bot Discord chatbot
    ├── infrastructure/
    │   ├── cilium/                     # CNI - HelmRelease from HelmRepository (currently FAILING, see Known Issues)
    │   ├── metallb/                    # L2 load balancer - HelmRelease v0.15.3
    │   ├── metallb-config/            # IPAddressPool 192.168.1.200-250, L2Advertisement
    │   ├── local-path-provisioner/    # Default StorageClass for PVCs
    │   └── rennovate/                 # Renovate bot (CronJob, see Renovate section)
    └── scripts/
        └── ciliumValues.yaml          # Reference values for Cilium (not directly used by Flux)
```

---

## How Flux Manages This Cluster

### Root Kustomization (flux-system)
The `flux-system` Kustomization watches this repo at `./clusters/my-cluster` and reconciles every 10 minutes. It has:
- **Pruning enabled** (`prune: true`) — resources removed from git are removed from the cluster
- **SOPS decryption** configured via the `sops-age` secret in `flux-system` namespace

The top-level `clusters/my-cluster/kustomization.yaml` includes all infrastructure and app directories. Flux builds this with kustomize, decrypts any SOPS-encrypted files, then applies to the cluster.

### Sub-Clusters (External Repo)
The file `apps/minecraft-gitops.yaml` defines a **separate GitRepository** pointing to `jay123q/minecraft-cluster-gitops` and two Kustomizations that deploy from it:

| Kustomization | Path in minecraft-cluster-gitops | Target Namespace | Health Check |
|---|---|---|---|
| minecraft-cluster | ./cluster | minecraft | Deployment/minecraft-server |
| minecraft-cor-cluster | ./minecraft-cor-server/cluster | minecraft-cor | Deployment/minecraft-cor-server |

Both use the same `flux-system` SSH secret for git auth. Both have health checks and 5m timeouts.

**Important:** Changes to the minecraft workloads are made in the `jay123q/minecraft-cluster-gitops` repo, NOT this one. This repo only defines the Flux source and kustomization pointers.

---

## Infrastructure Components

| Component | Type | Namespace | Status | Notes |
|---|---|---|---|---|
| **Cilium** | HelmRelease v1.18.6 | kube-system | **FAILING** | See Known Issues — cluster still works on previously applied config |
| **MetalLB** | HelmRelease v0.15.3 | metallb-system | Healthy | L2 mode, pool 192.168.1.200-250 |
| **Local Path Provisioner** | HelmRelease | local-path-storage | Healthy | Default StorageClass |
| **Renovate** | CronJob | rennovate | Healthy | See Renovate section |

### Networking
- **CNI:** Cilium with Hubble UI, envoy, eBPF LB
- **Load Balancer:** MetalLB L2 on 192.168.1.200-250
- **Service IPs in use:**
  - 192.168.1.200 — hello-world (port 80)
  - 192.168.1.201 — minecraft-server (ports 25565, 26585)
  - 192.168.1.202 — minecraft-cor-admin (port 26585)
- **K8s API:** 192.168.1.112:6443

---

## SOPS / Secrets Management

- **Encryption:** age key `age1zj4wa4h4z44jh0ftwahr7h3dghkw34caqenvteesvvh32cwv9dxs0aq7hp`
- **Key location on node:** ~/.config/sops/age/keys.txt
- **Cluster secret:** `sops-age` in `flux-system` namespace (key: `age.agekey`)
- **Encryption rules** (`.sops.yaml`): only encrypts `data` and `stringData` fields (via `encrypted_regex`), leaving apiVersion/kind/metadata in plaintext
- **Decryption:** configured on the `flux-system` Kustomization via `spec.decryption.provider: sops`
- **Tools on node:** sops 3.9.4 + age 1.2.1 installed at ~/.local/bin/

### Editing encrypted secrets
```bash
cd ~/Documents/github/minisform-kuber-cluster
sops clusters/my-cluster/infrastructure/rennovate/secret.yaml
# Opens in editor, auto-re-encrypts on save
```

**CRITICAL LESSON LEARNED:** The `gotk-sync.yaml` decryption config must also be applied to the live cluster object via `kubectl patch` — Flux does NOT self-apply changes to its own Kustomization spec from git. If SOPS decryption stops working after a cluster rebuild, run:
```bash
kubectl patch kustomization flux-system -n flux-system --type merge \
  -p '{spec:{decryption:{provider:sops,secretRef:{name:sops-age}}}}'
```
Then also recreate the age secret:
```bash
kubectl create secret generic sops-age \
  --namespace=flux-system \
  --from-file=age.agekey=~/.config/sops/age/keys.txt
```

---

## Renovate (Dependency Management)

Self-hosted Renovate running as a Kubernetes CronJob in the `rennovate` namespace.

- **Schedule:** `0 0 * * 0` (weekly, Sunday at midnight UTC)
- **Image:** renovate/renovate:latest
- **Autodiscover:** enabled, filtered to `jay123q/minisform-kuber-cluster` and `jay123q/minecraft-cluster-gitops`
- **Automerge:** enabled globally, with `schedule:automergeDaily` preset (automerges before 5am)
- **Dependency Dashboard:** enabled (`:dependencyDashboard` preset) — creates a tracking issue in each target repo
- **Config:** mounted from ConfigMap at /config/config.json
- **Auth:** GitHub fine-grained PAT stored in SOPS-encrypted secret `renovate-env`
- **Log level:** debug

### Renovate Config (config.json)
```json
{
  $schema: https://docs.renovatebot.com/renovate-schema.json,
  autodiscoverFilter: [jay123q/minisform-kuber-cluster, jay123q/minecraft-cluster-gitops],
  automerge: true,
  extends: [schedule:automergeDaily, :dependencyDashboard]
}
```

### PAT Permissions Required
The fine-grained GitHub PAT needs these repository permissions for ALL target repos:
- **Contents:** Read and write
- **Issues:** Read and write (Renovate queries issues via GraphQL — this caused platform-unknown-error when missing)
- **Pull requests:** Read and write
- **Metadata:** Read

### Renovate Activity
Renovate has already successfully:
- Created and merged PR #6 updating Flux from v2.7.5 to v2.8.5 (gotk-components.yaml)
- Note: Flux CLI still reports v2.7.5 — the gotk-components may need a `flux install` or pod restart to pick up the new version

### Common Renovate Debugging
- **Could not parse config file**: config.json does NOT support comments. No `#` or `//` in the JSON.
- **platform-unknown-error**: usually a PAT permission issue. Check the debug logs for the specific GraphQL field that's FORBIDDEN.
- **bad-credentials / 401**: PAT was regenerated but the k8s secret still has the old token. Re-encrypt with sops and push.
- **Cron schedule gotcha**: `* */6 * * *` means every minute during hours 0,6,12,18 (60 runs per window). Use `0 */6 * * *` for once every 6 hours.
- **Manual test run:** `kubectl create job --from=cronjob/renovate renovate-test -n rennovate`
- **Check logs:** `kubectl logs -n rennovate -l job-name=renovate-test --tail=50`

---

## Known Issues

### Cilium HelmRelease Failing
```
Helm upgrade failed: values don't meet the specifications of the schema(s):
cilium: at '/agent': got object, want boolean
```
The `agent` key in `cilium-helmrelease.yaml` values is set as an object (`agent.podSecurityContext.enabled: true`) but Cilium 1.18.6 expects `agent` to be a boolean. The cluster still functions because Cilium was previously installed successfully — the failed upgrade just means it's running an older config. To fix: check the Cilium 1.18.x values schema and restructure the values block. Consider removing the `agent` block entirely or replacing it with the correct schema-compliant values.

---

## Workloads Summary

| Namespace | Workload | Type | Schedule/Replicas | Storage | Managed By |
|---|---|---|---|---|---|
| default | hello-world | Deployment | 1 replica | none | this repo |
| minecraft | minecraft-server | Deployment | 1 replica | 10Gi PVC (local-path) | minecraft-cluster-gitops |
| minecraft-cor | minecraft-cor-server | Deployment | 1 replica | 20Gi PVC (local-path) | minecraft-cluster-gitops |
| minecraft-cor | minecraft-cor-backup | CronJob | daily 3am | uses minecraft-cor PVC | minecraft-cluster-gitops |
| rennovate | renovate | CronJob | weekly Sun midnight | none | this repo |
| schizo-bot | schizo-bot | Deployment | 1 replica | 20Gi + 15Gi PVC (local-path) | schizo-bot repo |

---

## Quick Reference Commands

```bash
# Force Flux to pull latest and reconcile
flux reconcile source git flux-system && flux reconcile kustomization flux-system

# Check all Flux resources
flux get all

# Check why a kustomization is failing
flux get kustomization flux-system

# View kustomize-controller logs (SOPS errors show here)
kubectl logs -n flux-system deploy/kustomize-controller --tail=50

# Validate kustomize build locally before pushing
cd clusters/my-cluster/infrastructure/rennovate && kubectl kustomize .

# Trigger manual renovate run
kubectl create job --from=cronjob/renovate renovate-test -n rennovate

# Check renovate logs
kubectl logs -n rennovate -l job-name=renovate-test --tail=50

# Edit SOPS-encrypted secret
cd ~/Documents/github/minisform-kuber-cluster && sops clusters/my-cluster/infrastructure/rennovate/secret.yaml
```

---

## Minecraft Servers — Suspended (2026-05-21)

Both minecraft server workloads have been **intentionally suspended**. Flux will not reconcile them, and the deployments are scaled to zero.

### What was done

1. **Suspended Flux Kustomizations** in this repo:
   - File: `clusters/my-cluster/apps/minecraft-gitops.yaml`
   - Added `spec.suspend: true` to both Kustomization resources (`minecraft-cluster` and `minecraft-cor-cluster`)
   - This prevents Flux from reconciling changes from the `minecraft-cluster-gitops` repo

2. **Scaled deployments to 0 replicas** in `jay123q/minecraft-cluster-gitops`:
   - File: `2026-minecaft-dnd-cluster/minecraft-server.yaml` — `replicas: 0`
   - File: `minecraft-cor-server/cluster/deployment.yaml` — `replicas: 0`

3. **Suspended the backup CronJob** in `jay123q/minecraft-cluster-gitops`:
   - File: `minecraft-cor-server/cluster/minecraft-backup-cron.yaml` — added `spec.suspend: true`

### How to resume

To bring the minecraft servers back online, reverse these changes:

1. In `jay123q/minecraft-cluster-gitops`:
   - Set `replicas: 1` in `2026-minecaft-dnd-cluster/minecraft-server.yaml`
   - Set `replicas: 1` in `minecraft-cor-server/cluster/deployment.yaml`
   - Remove `suspend: true` from `minecraft-cor-server/cluster/minecraft-backup-cron.yaml`
   - Commit and push

2. In this repo (`jay123q/minisform-kuber-cluster`):
   - Remove `suspend: true` from both Kustomizations in `clusters/my-cluster/apps/minecraft-gitops.yaml`
   - Commit and push

3. Force reconciliation (optional, otherwise Flux picks it up within 10 minutes):
   ```bash
   flux reconcile source git flux-system && flux reconcile kustomization flux-system
   ```

### Current state of affected resources

| Resource | Namespace | State | Notes |
|---|---|---|---|
| Kustomization/minecraft-cluster | flux-system | Suspended | Won't reconcile until resumed |
| Kustomization/minecraft-cor-cluster | flux-system | Suspended | Won't reconcile until resumed |
| Deployment/minecraft-server | minecraft | 0 replicas | PVC retained |
| Deployment/minecraft-cor-server | minecraft-cor | 0 replicas | PVC retained |
| CronJob/minecraft-cor-backup | minecraft-cor | Suspended | No backups while server is down |

**Note:** PVCs (`minecraft-data` in `minecraft` namespace, `minecraft-cor-data` in `minecraft-cor` namespace) are retained. World data is safe. Resuming will reattach to existing volumes.

---

## Schizo-Bot (Discord Chatbot) — Added 2026-08-04

A Discord chatbot ("Ben") using Ollama for LLM inference, ChromaDB for RAG-based context retrieval, and persistent conversation memory. Deployed as a single-container pod running both ollama and the Python app.

### Repo

- **Source:** ssh://git@github.com/jay123q/schizo-bot.git (branch: main)
- **GitRepository name in Flux:** `schizo-bot`
- **Kustomization name:** `schizo-bot` (defined in `apps/schizo-bot-gitops.yaml`)
- **Path from git root:** `./cluster`

### Components

| Resource | Namespace | Details |
|----------|-----------|---------|
| Deployment/schizo-bot | schizo-bot | 1 replica, Recreate strategy, image `jay123q/schizo-bot:latest` |
| PVC/schizo-bot-data | schizo-bot | 20Gi (local-path) — app data, ChromaDB (knowledge + memory), txt files |
| PVC/schizo-bot-ollama-models | schizo-bot | 15Gi (local-path) — persistent ollama model storage |
| Secret/schizo-bot-env | schizo-bot | SOPS-encrypted Discord TOKEN, mounted via envFrom |
| ResourceQuota | schizo-bot | 5 CPU req / 9 limit, 9Gi mem req / 13Gi limit |
| LimitRange | schizo-bot | Container defaults: 1 CPU / 2Gi mem request |

### How It Works

1. Container entrypoint starts `ollama serve` in the background
2. Waits for ollama readiness, then pulls `qwen3-embedding` and `qwen3` models
3. Models are stored on the `schizo-bot-ollama-models` PVC — survives restarts (no re-download)
4. Activates Python venv, runs `main.py` (Discord bot mode with memory enabled)
5. On each `!m` command from Discord:
   - Retrieves relevant knowledge chunks from `schizo_docs` collection (RAG)
   - Retrieves the 3 most relevant past exchanges from `conversation_memory` collection
   - Builds a prompt with both knowledge context and conversation history
   - Generates a response via qwen3 that can reference past interactions
   - Stores the new exchange (user name + question + response) back into memory

### Conversation Memory

- **Collection:** `conversation_memory` in ChromaDB (same persistent dir as knowledge base)
- **Embedding model:** qwen3-embedding (same as knowledge base)
- **Recall:** 3 most semantically similar past exchanges per query (configurable via `MEMORY_K`)
- **Storage:** each exchange stored as `[username] asked: ... [Ben] replied: ...` with timestamp metadata
- **Persistence:** survives pod restarts via `schizo-bot-data` PVC
- **Toggle:** `--no-memory` flag disables (on by default)
- **Effect:** Ben recognizes returning users, references past topics, builds rapport over time

### Container Details

- **Base:** python:3.12-slim + ollama (installed via curl) + zstd
- **Python deps:** chromadb, ollama, discord.py, python-dotenv (installed in /app/venv)
- **Env vars:** `TXT_DIR=/app/data/txt-files`, `CHROMA_PERSIST_DIR=/app/data/chroma_db`, `TOKEN` (Discord)
- **Resources:** requests 2 CPU / 4Gi mem, limits 6 CPU / 8Gi mem
- **CLI flags:** `--memory` (default on), `--no-memory`, `--reindex`, `--top-k N`, `--txt-dir PATH`

### Flux Source

Defined in `clusters/my-cluster/apps/schizo-bot-gitops.yaml`:
```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: GitRepository
metadata:
  name: schizo-bot
  namespace: flux-system
spec:
  interval: 1m0s
  url: ssh://git@github.com/jay123q/schizo-bot.git
  ref:
    branch: main
  secretRef:
    name: flux-system
---
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: schizo-bot
  namespace: flux-system
spec:
  interval: 10m
  timeout: 5m
  sourceRef:
    kind: GitRepository
    name: schizo-bot
  path: ./cluster
  prune: true
  wait: true
  decryption:
    provider: sops
    secretRef:
      name: sops-age
  healthChecks:
  - apiVersion: apps/v1
    kind: Deployment
    name: schizo-bot
    namespace: schizo-bot
```

### Service IPs

None allocated — the bot connects outbound to Discord and Ollama runs in-container. No inbound Service needed.

### Deployment Steps

1. Build and push the image:
   ```bash
   cd ~/Documents/github/schizo-bot
   docker build -t jay123q/schizo-bot:latest .
   docker push jay123q/schizo-bot:latest
   ```

2. (If not done) Create and SOPS-encrypt Discord token in `cluster/secret.yaml`

3. Commit and push the schizo-bot repo

4. Flux picks up within 10 minutes, or force:
   ```bash
   flux reconcile source git schizo-bot && flux reconcile kustomization schizo-bot
   ```

### Troubleshooting

- **Pod CrashLoopBackOff:** Check if ollama has enough memory to load models. qwen3-embedding is ~4.7GB, qwen3 is ~5.2GB. Ensure limits are sufficient.
- **Models re-downloading every restart:** Verify `schizo-bot-ollama-models` PVC is bound and mounted at `/root/.ollama`.
- **Bot not responding in Discord:** Check `TOKEN` env var is set via the SOPS secret. Look at pod logs for discord.py connection errors.
- **ChromaDB errors:** May need `--reindex` if txt source files changed structure. Check `/app/data/chroma_db` mount.
- **Memory not working:** Verify the `conversation_memory` collection exists in ChromaDB. Check pod logs for embedding errors.
- **saveChatOutput crash:** Entrypoint creates `chat_history/` and `qwen3-save-output/` dirs. If missing, check entrypoint.sh.
- **IndexError in saveChatOutput:** Fixed — handles LLM outputs shorter than 5 words.
