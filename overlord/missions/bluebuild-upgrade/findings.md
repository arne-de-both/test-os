# 🔍 Findings: QEMU / binfmt_misc Build Failure in Kubernetes

## ⚠️ The Issue
The GitHub Actions workflow failed during the `Set up QEMU` and `Extracting available platforms` steps with the following error:
```
cannot mount binfmt_misc filesystem at /proc/sys/fs/binfmt_misc
error: no such device
...
##[error]Unexpected end of JSON input
```

## 🧠 Root Cause Analysis
1. **GitHub Action Change**: Upgrading from `uses: blue-build/github-action@v1.8` to `uses: blue-build/github-action@v1` (which resolves to `v1.11.1`) introduced a hardcoded `docker/setup-qemu-action` step. This step was not present in `v1.8.x`.
2. **Kubernetes Runner Constraints**: The self-hosted runner `quantus-builders` is hosted on a personal Kubernetes cluster (likely using actions-runner-controller or runner-scale-set in a container environment).
3. **binfmt_misc Restriction**: The QEMU setup action tries to run a privileged container (`docker.io/tonistiigi/binfmt`) to mount the `binfmt_misc` filesystem on `/proc/sys/fs/binfmt_misc`. This fails in the Kubernetes runner because:
   - The container runtime of the runner pod/container does not allow mounting host filesystems (lacks permissions/privileges).
   - The `binfmt_misc` kernel module might not be loaded on the Kubernetes worker nodes.
4. **JSON Parse Failure**: Because the `binfmt` registration failed, the subsequent step `Extracting available platforms` returned empty or invalid output, causing the setup action's Javascript code to crash with `Unexpected end of JSON input`.

## 🛠️ Proposed Solutions / Next Steps

### Option A: Configure the Kubernetes Runner Nodes (Host Level)
Ensure `binfmt_misc` is supported and loaded on the Kubernetes worker nodes:
1. Run `sudo modprobe binfmt_misc` on the Kubernetes host nodes.
2. Ensure the runner pods have the necessary security contexts (e.g. `privileged: true` or capability `SYS_ADMIN`) to interact with `binfmt_misc`.

### Option B: Revert to Action v1.8 (With the COPR Fixes)
If host-level QEMU configuration is not desired or possible:
1. Revert the workflow to use the older action version that does not run QEMU:
   ```yaml
   uses: blue-build/github-action@v1.8
   ```
2. Keep the updated COPR and package configuration (`lionheartp/Hyprland` and `hyprland-guiutils`), which resolved the original `dnf` build issue under `v1.8`.

### Option C: Request a Skip Option from BlueBuild
Submit a feature request/PR to `blue-build/github-action` to make the `Set up QEMU` step optional (e.g., via a new input parameter like `setup_qemu: false`).

---

# 🔍 Findings: `envsubst: command not found` (exit 127) — 2026-08-03

## ⚠️ The Issue
Every scheduled build failed at ~45s with exit code 127:
```
/home/runner/_work/_temp/....sh: line 3: envsubst: command not found
```
The failure is in the `sigstore/cosign-installer` step nested inside
`blue-build/github-action@v1`.

## 🧠 Root Cause Analysis (evidence-based)
1. **Nothing in this repo changed.** Last repo commit: 2026-06-22. Builds kept
   succeeding after the v1 upgrade (last full success: **2026-07-06**, run
   `28784447533`, 16m20s).
2. **Upstream floating-tag bump.** `blue-build/github-action@v1` is a moving tag
   that transitively pulls `sigstore/cosign-installer`. The installer SHA changed
   between the last success and the first failure:
   - 2026-07-06 (✓): `sigstore/cosign-installer@faadad0c…` — script did **not** call `envsubst`.
   - 2026-07-07 (✗, run `28856568639`): `sigstore/cosign-installer@6f9f177…` — new script line 3: `install_dir=$(envsubst <<<"${input_install_dir}")`.
   Unbroken failures every day since 2026-07-07.
3. **Runner lacks `gettext`.** `quantus-builders` is an ARC runner-scale-set pod
   (runner `2.336.0`, `/home/runner`), whose **job steps run in an Ubuntu 24.04**
   container (the minimal `actions/actions-runner` image). That image does not
   ship `gettext-base` (which provides `envsubst`). GitHub-*hosted* runners have
   it via their full provisioned image, so this only breaks self-hosted.
   - NOTE: the log line `Operating System: Alpine Linux v3.24 (containerized)` is
     the **BuildKit `docker-container` builder**, not the runner. The runner is Ubuntu 24.04.

## ✅ Applied Fix (2026-08-03)
Workflow pre-step in `.github/workflows/build.yml` that installs `gettext-base`
before the BlueBuild action (idempotent; handles root and sudo). Band-aid that is
resilient to future upstream floating-tag surprises, costs a few seconds/build.

## 🛠️ Future Improvements (if the pre-step proves insufficient)
- **Root-cause A — bake `gettext` into the runner image.** Cleanest: add
  `gettext-base` to the custom ARC runner image (or via the runner-scale-set
  Helm `template.spec.containers[].image`). Fixes all workflows, no per-build cost.
  Config lives on the K8s cluster, **not in this repo**.
- **Root-cause B — full GitHub-parity image (`catthehacker/ubuntu:full-24.04`).**
  Gives hosted-runner parity (fewer future surprises) but is **multi-GB** →
  **rejected 2026-08-03** as too heavy for the cluster's node disk / pull time.
- **Pin the action** to a specific `blue-build/github-action` SHA/version instead
  of the floating `@v1` tag, so upstream changes can't silently break builds.
- Still-open, separate issue: the **QEMU / binfmt_misc** privilege failure above
  is unrelated and not fixed by any image/package change (needs pod security
  context / node config).
