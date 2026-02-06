
# Secure Pipeline Assessment

## Task Overview

1. **Deploy** the provided `broken-manifest.yaml` to a local Kubernetes cluster (Minikube or Kind).
2. **Diagnose** why the pod enters a `CrashLoopBackOff` state.
3. **Root Cause Analysis (RCA):**
	- Provide a Markdown document explaining your findings.
4. **Submit a Fixed Manifest:**
	- Ensure the service remains stable after your fix.

---

## Candidate Submission Guide

- **Format:** A single `.zip` file or a GitHub Gist link.

### Contents

- `RCA.md`: Documenting the identification of `OOMKilled`, the memory leak analysis, and the commands used for discovery.
- `fixed-manifest.yaml`: The corrected Kubernetes Deployment file.
- (Optional) **screenshots**: Evidence of the pod in a Running state after the fix.