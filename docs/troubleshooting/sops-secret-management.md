# SOPS Secret Management Troubleshooting

## Issue Summary

**Problem**: Apps failing to deploy due to missing SOPS secrets in target namespaces
**Error**: `secrets "sops-age" not found` in application namespaces
**Root Cause**: Architectural misalignment with onedr0p pattern for SOPS decryption inheritance

## Investigation Timeline

### Initial Symptoms
- Kustomization build failures: `secrets "sops-age" not found` (19,325+ failures for echo, 38,622+ for home-assistant)
- HelmRelease dependency failures: `dependency 'network/cloudflare-tunnel' is not ready` (462,795+ events)
- OCIRepository missing: `OCIRepository "app-template" not found`

### Root Cause Analysis

**Commit Chain That Introduced Issues:**

1. **8b06063** (July 19, 15:19:36) - Initial architectural change
   - Added main `apps/kustomization.yaml` with `namespace: flux-system`
   - Used `components: - ../components/common` pattern
   - **Impact**: Changed how SOPS secrets are distributed

2. **1f86cea** (July 19, 15:36:55) - Attempted fix
   - Moved from components to direct resource references
   - Removed namespace target from main kustomization
   - **Impact**: Broke SOPS secret inheritance pattern

3. **0b05e2a** (July 19, 15:40:25) - Partial fix
   - Added explicit `namespace: flux-system` to SOPS kustomization
   - **Impact**: Fixed SOPS location but didn't solve distribution

4. **3273444** (July 19, 15:41:43) - Completed partial fix
   - Added explicit `namespace: flux-system` to OCIRepository kustomization
   - **Impact**: Fixed OCIRepository but SOPS distribution remained broken

## onedr0p Pattern Analysis

### Key Architectural Differences

| Component | onedr0p Pattern | Our Broken Setup | Issue |
|-----------|-----------------|------------------|--------|
| **SOPS Secret Location** | flux-system only | flux-system only | ✅ Same |
| **Decryption Inheritance** | postBuild patches | Per-Kustomization refs | ❌ Missing |
| **Apps Structure** | `components: common` | `resources: direct refs` | ⚠️ Different |
| **Secret Distribution** | Flux-managed inheritance | Expected per-namespace | ❌ Mismatch |

### onedr0p's Working Pattern

```yaml
# Single cluster-level Kustomization with inheritance
apiVersion: kustomize.toolkit.fluxcd.io/v1
kind: Kustomization
metadata:
  name: cluster-apps
  namespace: flux-system
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Only needs to exist in flux-system
  postBuild:
    substitute: {}
    substituteFrom:
      - kind: Secret
        name: cluster-secrets
        optional: true
  patches:
    # Child Kustomizations inherit SOPS decryption automatically
    - patch: |
        - op: add
          path: /spec/decryption
          value:
            provider: sops
            secretRef:
              name: sops-age
      target:
        kind: Kustomization
        labelSelector: "kustomize.toolkit.fluxcd.io/name"
```

### Our Broken Pattern

```yaml
# Multiple Kustomizations each expecting their own SOPS secret
spec:
  decryption:
    provider: sops
    secretRef:
      name: sops-age  # Expected in each namespace but only exists in flux-system
```

## Resolution Strategy

### 1. SOPS Inheritance Implementation

**Add postBuild patches** to cluster-apps Kustomization to propagate SOPS decryption capability to child Kustomizations:

```yaml
spec:
  # ... existing cluster-apps config ...
  postBuild:
    substitute: {}
    substituteFrom:
      - kind: Secret
        name: cluster-secrets
        optional: true
  patches:
    - patch: |
        - op: add
          path: /spec/decryption
          value:
            provider: sops
            secretRef:
              name: sops-age
      target:
        kind: Kustomization
        labelSelector: "kustomize.toolkit.fluxcd.io/name"
```

### 2. Structural Realignment

**Revert to components pattern** in `kubernetes/apps/kustomization.yaml`:

```yaml
# FROM (broken):
resources:
  - ../components/common/repos
  - ../components/common/sops
  - ./cert-manager
  # ...

# TO (working):
components:
  - ../components/common
resources:
  - ./cert-manager
  # ...
```

### 3. Remove Explicit SOPS References

Remove individual SOPS `secretRef` configurations from namespace-level Kustomizations - inheritance will handle decryption.

## Key Learnings

### 1. SOPS Decryption Inheritance
- **Centralized**: SOPS secrets should exist only in flux-system namespace
- **Inherited**: Use Flux postBuild patches to propagate decryption capability
- **Automatic**: Child Kustomizations don't need explicit SOPS references

### 2. Flux Architecture Patterns
- **Components over Resources**: Use Kustomize components for shared resources
- **Dependency Management**: Proper dependency chains prevent chicken-and-egg issues
- **Inheritance**: Leverage Flux's built-in inheritance mechanisms

### 3. Troubleshooting Methodology
- **Event Analysis**: High event counts (19k+, 462k+) indicate systemic issues
- **Root Cause**: Look for architectural changes, not just symptoms
- **Reference Patterns**: Study working implementations (onedr0p) for guidance

## Prevention

### 1. Testing Strategy
- **Dry-run**: Always test Kustomization changes with `kubectl diff`
- **Staged Rollout**: Apply architectural changes gradually
- **Monitor Events**: Watch for recurring error patterns

### 2. Architecture Guidelines
- **Follow Patterns**: Align with proven architectures (onedr0p template)
- **Document Changes**: Record architectural decisions and rationale
- **Validate Inheritance**: Ensure shared resources propagate correctly

### 3. Monitoring
- **Event Tracking**: Monitor high-frequency error events
- **Resource Status**: Check Kustomization and HelmRelease health
- **Dependency Chains**: Validate dependency resolution

## Related Files

- `kubernetes/flux/cluster/ks.yaml` - Main cluster Kustomizations
- `kubernetes/apps/kustomization.yaml` - App structure definition  
- `kubernetes/components/common/` - Shared components
- `.sops.yaml` - SOPS encryption configuration

## References

- [onedr0p/home-ops](https://github.com/onedr0p/home-ops) - Reference architecture
- [Flux Kustomization Inheritance](https://fluxcd.io/flux/components/kustomize/kustomizations/#post-build-variable-substitution)
- [SOPS Integration](https://fluxcd.io/flux/guides/mozilla-sops/)