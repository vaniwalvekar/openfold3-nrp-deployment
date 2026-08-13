\# OpenFold3 on NRP Nautilus



Kubernetes deployment manifests for running \[OpenFold3](https://github.com/aqlaboratory/openfold-3) — a fully open source biomolecular structure prediction model based on AlphaFold3 — on the \[National Research Platform](https://nrp.ai/) (NRP) Nautilus cluster.



This repo documents a working, end-to-end deployment: containerizing OpenFold3, pushing it to Docker Hub, provisioning persistent storage on NRP, and submitting a real GPU inference job that predicts a protein's 3D structure.



\## What this does



Runs OpenFold3 inference on \*\*ubiquitin\*\* (PDB: 1UBQ), a small, well-characterized 76-residue protein, using:

\- The \[ColabFold MSA server](https://github.com/sokrypton/ColabFold) for multiple sequence alignment generation (no local sequence databases required)

\- A single NVIDIA A10 GPU on NRP's federated cluster

\- Ceph-backed persistent storage for model weights and output structures



Output: 5 predicted 3D structures (`.cif` files) with per-residue confidence scores (pLDDT, PAE, PDE).



\## Docker image



Built from \[`aqlaboratory/openfold-3`](https://github.com/aqlaboratory/openfold-3)'s `docker/Dockerfile.pixi` (`devel` target, `openfold3-cuda12` pixi environment).



docker pull vaniwalvekar/openfold3-vani:cuda12



\## Files



| File | Purpose |

|---|---|

| `pvc.yaml` | Persistent Volume Claim — 10Gi Ceph-backed storage for model weights and output |

| `openfold-job.yaml` | Kubernetes Job — runs OpenFold3 inference on a GPU node |

| `file-browser.yaml` | Throwaway utility pod for browsing/copying files out of the PVC via `kubectl cp` |



\## Prerequisites



\- `kubectl` configured against NRP Nautilus (see \[NRP Getting Started](https://nrp.ai/documentation/userdocs/start/getting-started/))

\- An NRP namespace with metadata (PI, Institution, Software, Description) filled in — required by NRP's admission webhook before jobs will run

\- `kubectl-oidc\_login` plugin installed for CILogon authentication



\## Usage



```bash

\# 1. Create persistent storage

kubectl apply -f pvc.yaml --validate=false



\# 2. Submit the inference job

kubectl apply -f openfold-job.yaml --validate=false



\# 3. Watch it run

kubectl get pods

kubectl logs -f <pod-name>



\# 4. Once Completed, browse/copy results out of the PVC

kubectl apply -f file-browser.yaml --validate=false

kubectl cp file-browser:/data/output ./results

kubectl delete pod file-browser

```



\## Key design notes



\- \*\*No `nodeSelector` pinned to a specific reserved pool\*\* — NRP hosts institution-specific reserved node pools (tainted, e.g. CSU TIDE) that aren't schedulable by default. This job uses `nodeAffinity` to require `NVIDIA-A10` specifically, since it's freely available cluster-wide and architecturally compatible with the image's compiled CUDA kernels (compute capability 8.6).

\- \*\*`limits` == `requests`\*\* for CPU/memory, per NRP's cluster resource policy.

\- \*\*`/dev/shm` explicitly sized to 4Gi\*\* — PyTorch's multi-process data loading needs more than Kubernetes' 64MB default.

\- \*\*`yes |` piped into the run command\*\* — `run\_openfold` prompts interactively to confirm the model checkpoint download; this isn't answerable inside a non-interactive container without it.

\- \*\*`--use-msa-server=true`\*\* (not a bare flag) — this CLI option requires an explicit boolean value.



\## Viewing results



Predicted structures (`.cif` files) can be viewed at \[molstar.org](https://molstar.org) — drag and drop the file directly into the browser.



Confidence is summarized in each `\*\_confidences\_aggregated.json` file's `avg\_plddt` field (>90 = very high confidence).



\## Acknowledgments



Built on \[OpenFold3](https://github.com/aqlaboratory/openfold-3) (Apache 2.0) by the OpenFold Team, running on \[NRP Nautilus](https://nrp.ai/), supported by NSF.





