# ⚙️ Spec: BlueBuild Upgrade

## 🎯 Goal
Upgrade the BlueBuild GitHub Action configuration to the latest version to access new features, optimizations, and security patches while maintaining local CLI compatibility.

## 📋 Requirements
1. **GitHub Action Upgrade**:
   - Update `.github/workflows/build.yml` to target `blue-build/github-action@v1`.
2. **Local CLI Verification**:
   - Verify local CLI version is compatible with the version used by the new action.
   - Currently, local CLI is `v0.9.35` (latest) and `@v1` action defaults to `v0.9` installer, guaranteeing perfect parity.
3. **Validation**:
   - Ensure all recipes remain valid post-upgrade.

## 🛠️ Design Decisions
* **Action Pinning**: Pinned to `@v1` to allow automatic minor and patch updates while avoiding manual version maintenance.
* **Layer Rechunking**: Kept `build_chunked_oci` disabled to prioritize fast CI build cycles.

## 🏁 Success Criteria
* [ ] `.github/workflows/build.yml` updated to `blue-build/github-action@v1`.
* [ ] Successful local recipe validation.
* [ ] Successful test build run in the GitHub Action pipeline.

## 📝 Decision Log
### 2026-06-15: GitHub Action Pinning Strategy
* **Decision**: Pin to `@v1` major version.
* **Rationale**: Automatic updates for minor features and bug fixes without breaking changes.

### 2026-06-15: Rechunking Feature
* **Decision**: Disabled.
* **Rationale**: Prioritizing CI build speed over client-side delta optimizations for now.
