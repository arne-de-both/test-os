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
