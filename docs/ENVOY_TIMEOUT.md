# Envoy API Timeout Configuration

## Problem

Olares uses **Envoy sidecar** for routing all application traffic. By default, the API timeout is set to **15 seconds**. This causes issues for:

- Long-running API requests (AI generation, file processing, etc.)
- Streaming responses (SSE, WebSocket initialization)
- Large file uploads/downloads

**Symptom**: API requests are cut off after exactly 15 seconds with no response.

## Solution: `options.apiTimeout` in OlaresManifest.yaml

Add `apiTimeout: 0` under `options` section to disable the timeout (unlimited):

```yaml
# OlaresManifest.yaml

options:
  apiTimeout: 0  # Unlimited timeout (default is 15 seconds)
  analytics:
    enabled: false
  dependencies:
    - name: olares
      type: system
      version: '>=1.10.0-0'
```

### apiTimeout Values

| Value | Meaning |
|-------|---------|
| `0` | Unlimited (no timeout) |
| `15` | 15 seconds (default) |
| `300` | 5 minutes |
| `3600` | 1 hour |

## Technical Details

### How It Works

1. **OlaresManifest.yaml** `options.apiTimeout` is read during app installation
2. Olares generates **Envoy sidecar ConfigMap**: `olares-sidecar-config-{appname}`
3. The timeout value maps to Envoy's `RouteAction.Timeout` in the generated config

### Envoy Configuration Location

```bash
# View the generated Envoy config
kubectl get cm olares-sidecar-config-{appname} -n {appname}-{username} -o yaml
```

### Before (Default 15s)

```yaml
# In Envoy config (envoy.yaml)
routes:
  - match:
      prefix: /
    route:
      cluster: original_dst
      timeout: 15s  # <-- This is the problem
```

### After (apiTimeout: 0)

```yaml
# In Envoy config (envoy.yaml)
routes:
  - match:
      prefix: /
    route:
      cluster: original_dst
      timeout: 0s  # <-- Unlimited
```

## Applying the Change

### For New Applications

Simply add `apiTimeout: 0` to your OlaresManifest.yaml before packaging and installing.

### For Existing Applications

1. **Update OlaresManifest.yaml** with `apiTimeout: 0`
2. **Bump version** in both:
   - `Chart.yaml`: `version: x.x.x` (increment)
   - `OlaresManifest.yaml`: `version: 'x.x.x'` (must match)
3. **Re-package the chart**:
   ```bash
   helm package /path/to/your-chart
   ```
4. **Upload to Market** (My Olares → Upload custom chart)
5. **Upgrade the application** when prompted

**Important**: Version must be higher than the currently installed version, or the upgrade will be rejected.

## Verification

After installation/upgrade, verify the timeout is applied:

### Method 1: Via Control Hub

1. Open Control Hub → Browse → {your-app-namespace}
2. Expand Configmaps → `olares-sidecar-config-{appname}`
3. View `envoy.yaml` content
4. Look for `timeout: 0s` in the route configuration

### Method 2: Via kubectl (if available)

```bash
kubectl get cm olares-sidecar-config-{appname} -n {appname}-{username} -o yaml | grep -A5 timeout
```

## Related: DevBox vs Custom Apps

| App Source | Default Timeout | ConfigMap Pattern |
|------------|-----------------|-------------------|
| DevBox (`source: devbox`) | 300s (5 min) | `devbox-sidecar-configs` |
| Custom/Market (`source: custom`) | 15s | `olares-sidecar-config-{app}` |

DevBox applications automatically get a longer timeout (300s). Custom applications need explicit `apiTimeout` configuration.

## Common Issues

### Issue: Upgrade rejected - version not higher

```
Error: new version x.x.x is not higher than current version x.x.x
```

**Solution**: Increment the version number in both Chart.yaml and OlaresManifest.yaml.

### Issue: Timeout still 15s after upgrade

**Possible causes**:
1. OlaresManifest.yaml change not included in the packaged chart
2. App not actually upgraded (check version in Market)
3. Pod not restarted after ConfigMap update

**Solution**: Verify the version, re-package ensuring OlaresManifest.yaml is updated, and confirm upgrade completed.

## Example: Complete OlaresManifest.yaml

```yaml
olaresManifest.version: '0.10.0'
olaresManifest.type: app

metadata:
  name: myapp
  title: My Application
  description: "Application with unlimited API timeout"
  icon: https://example.com/icon.png
  version: '0.1.1'
  categories:
    - Productivity

permission:
  appData: true

spec:
  versionName: '1.0.0'
  developer: yourname
  requiredMemory: 512Mi
  limitedMemory: 2048Mi
  requiredDisk: 1024Mi
  limitedDisk: 10240Mi
  requiredCpu: 250m
  limitedCpu: 2000m
  supportArch:
    - amd64
    - arm64

entrances:
  - name: myapp
    title: My App
    icon: https://example.com/icon.png
    host: myapp-svc
    port: 8080
    authLevel: private
    openMethod: default

options:
  apiTimeout: 0  # <-- CRITICAL: Unlimited timeout
  analytics:
    enabled: false
  dependencies:
    - name: olares
      type: system
      version: '>=1.10.0-0'

middleware:
  postgres:
    username: myapp
    databases:
      - name: myapp
```

## References

- [Olares Documentation - OlaresManifest](https://docs.olares.com/developer/develop/package/manifest.html)
- [Envoy Proxy - Route Configuration](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/route/v3/route_components.proto#envoy-v3-api-field-config-route-v3-routeaction-timeout)
