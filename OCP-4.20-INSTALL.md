# MCP Server Setup on OCP 4.20

Working instructions for installing the MCP Server, MCP Gateway, and MCP Broker
on **OCP 4.20** (Kubernetes 1.33).

> **Note:** OCP 4.20 is below the Red Hat-packaged Helm chart's `kubeVersion`
> constraint (`>= 1.34.0-0`) and below the official documentation target
> (OCP 4.22). The upstream chart uses only standard Kubernetes APIs
> (Deployments, Services, Ingress, ConfigMaps, RBAC) that are fully available
> on OCP 4.20. This guide uses the **upstream community chart** which has no
> version gate.
>
> **This is an unsupported configuration.** Red Hat support does not cover
> MCP Server on OCP 4.20. Proceed only for PoC / development use.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| OpenShift Container Platform | **4.20** with cluster-admin |
| `oc` CLI | Installed and authenticated |
| Helm CLI | 3.x installed |
| MCP-compatible client | Claude Code, Cursor, VS Code, or OpenShift LightSpeed |

Confirm your cluster version:

```bash
oc get clusterversion
oc version
```

Get your cluster's apps domain (used throughout this guide):

```bash
export CLUSTER_DOMAIN=$(oc get ingresses.config/cluster -o jsonpath='{.spec.domain}')
echo "Cluster domain: ${CLUSTER_DOMAIN}"
```

---

## Step 1: Install the MCP Server (Upstream Chart)

Use the **upstream community chart** from `oci://ghcr.io/containers/charts/kubernetes-mcp-server`
instead of `openshift-helm-charts/redhat-openshift-mcp-server`. The community
chart has no `kubeVersion` constraint.

> **Chart options:**
> - `oci://ghcr.io/containers/charts/kubernetes-mcp-server` — community chart, **no version gate** (verified)
> - `oci://ghcr.io/openshift/charts/openshift-mcp-server` — OpenShift-specific chart from `release-0.2.0` branch (may have its own version constraints)
> - `openshift-helm-charts/redhat-openshift-mcp-server` — Red Hat certified chart, has `kubeVersion: ">= 1.34.0-0"` (will fail on OCP 4.20)

### Read-only access (recommended)

```bash
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    oci://ghcr.io/containers/charts/kubernetes-mcp-server \
    --set config.toolsets='{core,config,helm,metrics,netedge}' \
    --set ingress.host=mcp-server.apps.${CLUSTER_DOMAIN} \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"use-view-role","roleRef":{"name":"view","external":true}}]'
```

### Full cluster access

```bash
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    oci://ghcr.io/containers/charts/kubernetes-mcp-server \
    --set config.toolsets='{core,config,helm,metrics,netedge}' \
    --set ingress.host=mcp-server.apps.${CLUSTER_DOMAIN} \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"admin-access","roleRef":{"name":"cluster-admin","external":true}}]'
```

### Alternative: Use the Red Hat chart with version check removed

If you must use the Red Hat-packaged chart:

```bash
# Pull the chart locally
helm pull openshift-helm-charts/redhat-openshift-mcp-server --untar

# Remove the kubeVersion constraint
sed -i.bak '/^kubeVersion:/d' redhat-openshift-mcp-server/Chart.yaml

# Install from the local directory
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    ./redhat-openshift-mcp-server \
    --set config.toolsets='{core,config,helm,metrics,netedge}' \
    --set ingress.host=mcp-server.apps.${CLUSTER_DOMAIN} \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"use-view-role","roleRef":{"name":"view","external":true}}]'
```

### Verify

```bash
oc get pods -n openshift-mcp-server
# Expected: openshift-mcp-server pod Running

# Test the MCP server endpoint directly
curl -s -X POST https://mcp-server.apps.${CLUSTER_DOMAIN}/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18","capabilities":{},
    "clientInfo":{"name":"test-client","version":"1.0.0"}}}'
```

---

## Step 2: Revoke Access to Sensitive Custom Resources

Create a file `deny-resources.toml`:

```toml
[[denied_resources]]
group = ""
version = "v1"
kind = "Secret"

[[denied_resources]]
group = ""
version = "v1"
kind = "ConfigMap"

[[denied_resources]]
group = "rbac.authorization.k8s.io"
version = "v1"
kind = "Role"

[[denied_resources]]
group = "rbac.authorization.k8s.io"
version = "v1"
kind = "RoleBinding"

[[denied_resources]]
group = "rbac.authorization.k8s.io"
version = "v1"
kind = "ClusterRole"

[[denied_resources]]
group = "rbac.authorization.k8s.io"
version = "v1"
kind = "ClusterRoleBinding"
```

Apply:

```bash
helm upgrade openshift-mcp-server oci://ghcr.io/containers/charts/kubernetes-mcp-server \
    -n openshift-mcp-server \
    --set-file config.deniedResources=deny-resources.toml
```

---

## Step 3: Configure RBAC

Create a file `rbac-values.yaml`:

```yaml
extraClusterRoles:
  - name: acm-managed-clusters
    rules:
      - apiGroups: ["cluster.open-cluster-management.io"]
        resources: ["managedclusters"]
        verbs: ["list", "watch"]

extraClusterRoleBindings:
  - name: acm-managed-clusters
    roleRef:
      name: acm-managed-clusters

extraRoles:
  - name: acm-cluster-proxy-discovery
    namespace: multicluster-engine
    rules:
      - apiGroups: [""]
        resources: ["services"]
        verbs: ["get"]

extraRoleBindings:
  - name: acm-cluster-proxy-discovery
    namespace: multicluster-engine
    roleRef:
      name: acm-cluster-proxy-discovery
```

Apply:

```bash
helm upgrade openshift-mcp-server oci://ghcr.io/containers/charts/kubernetes-mcp-server \
    -n openshift-mcp-server \
    -f rbac-values.yaml
```

> Skip the ACM-related roles if this is a single-cluster setup without RHACM.

---

## Step 4: Install the MCP Gateway

Install via Red Hat Connectivity Link 1.3.

> See: [Installing the MCP Gateway](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install)

> **OCP 4.20 Note:** The MCP Gateway depends on Gateway API CRDs (`HTTPRoute`,
> `Gateway`). On OCP 4.20 these may not be installed by default. If the gateway
> install fails, check for the Gateway API CRDs first:
>
> ```bash
> oc get crd gateways.gateway.networking.k8s.io
> oc get crd httproutes.gateway.networking.k8s.io
> ```
>
> If missing, install them manually:
>
> ```bash
> oc apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
> ```

Set the gateway namespace (use the namespace where you installed the gateway):

```bash
export GATEWAY_NS=<gateway-namespace>
```

Verify:

```bash
# Gateway controller running
oc get pods -n ${GATEWAY_NS} -l "app.kubernetes.io/name=mcp-gateway"

# MCPGatewayExtension exists
oc get mcpgatewayextension -A

# Broker-router running
oc logs -n ${GATEWAY_NS} deployment/mcp-gateway

# EnvoyFilter created
oc get envoyfilter -n ${GATEWAY_NS} -l app.kubernetes.io/managed-by=mcp-gateway-controller
```

---

## Step 5: Connect Server to Gateway

### 5a. Verify the MCP server service

```bash
oc get svc -n openshift-mcp-server
```

### 5b. Create the HTTPRoute

```bash
export MCP_SERVER_HOST=mcp-server.apps.${CLUSTER_DOMAIN}

oc apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-api-key-server-route
  namespace: openshift-mcp-server
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    namespace: ${GATEWAY_NS}
    sectionName: mcp
  hostnames:
  - ${MCP_SERVER_HOST}
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp
    backendRefs:
    - name: openshift-mcp-server
      port: 8080
EOF
```

### 5c. Create the MCPServerRegistration

```bash
oc apply -f - <<EOF
apiVersion: mcp.kuadrant.io/v1alpha1
kind: MCPServerRegistration
metadata:
  name: my-mcp-server-reg
  namespace: openshift-mcp-server
spec:
  prefix: "openshift_"
  targetRef:
    group: "gateway.networking.k8s.io"
    kind: "HTTPRoute"
    name: "mcp-api-key-server-route"
    namespace: openshift-mcp-server
EOF
```

### 5d. Verify registration

```bash
oc wait --for=condition=Ready mcpsr/my-mcp-server-reg \
  -n openshift-mcp-server --timeout=120s

oc get mcpsr -A
# READY should be True, TOOLS should show discovered tool count
```

### 5e. Verify tools through the gateway

```bash
# Initialize MCP session
curl -s -D /tmp/mcp_headers -X POST \
  http://mcp-gateway.apps.${CLUSTER_DOMAIN}:8001/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18","capabilities":{},
    "clientInfo":{"name":"test-client","version":"1.0.0"}}}'

# Extract session ID
SESSION_ID=$(grep -i "mcp-session-id:" /tmp/mcp_headers | cut -d' ' -f2 | tr -d '\r')
echo "Session ID: $SESSION_ID"

# List tools
curl -X POST http://mcp-gateway.apps.${CLUSTER_DOMAIN}:8001/mcp \
  -H "Content-Type: application/json" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

# Clean up
rm -f /tmp/mcp_headers
```

Tools should appear prefixed with `openshift_`.

---

## Step 6: Connect an MCP Client

At this point the MCP Server and Gateway are functional. You can connect a
client directly **without authentication** for PoC testing.

**Claude Code** (`.mcp.json`):

```json
{
  "mcpServers": {
    "openshift": {
      "type": "http",
      "url": "https://mcp-gateway.apps.<cluster-domain>/mcp"
    }
  }
}
```

**Cursor** (MCP settings):

```json
{
  "mcpServers": {
    "openshift": {
      "type": "http",
      "url": "https://mcp-gateway.apps.<cluster-domain>/mcp"
    }
  }
}
```

Replace `<cluster-domain>` with your cluster's apps domain.

Accept the CA certificate when prompted by the client.

---

## Step 7 (Optional): Authentication & Authorization

Setting up OAuth authentication (Entra ID) and tool-level authorization follows
the same steps as the main guide. See the [README.md](README.md) Steps 6–7.

> **OCP 4.20 Note:** The authentication steps depend on Kuadrant / Authorino.
> Verify these operators are available and compatible with your OCP 4.20 cluster
> before proceeding. If they are not available, run without authentication for
> PoC purposes.

---

## Verification Checklist

```bash
# 1. Cluster version
oc get clusterversion

# 2. MCP Server running
oc get pods -n openshift-mcp-server

# 3. MCP Server responding
curl -sk https://mcp-server.apps.${CLUSTER_DOMAIN}/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18","capabilities":{},
    "clientInfo":{"name":"test-client","version":"1.0.0"}}}'

# 4. MCP Gateway running
oc get pods -n ${GATEWAY_NS} -l "app.kubernetes.io/name=mcp-gateway"

# 5. MCPServerRegistration is Ready
oc get mcpsr -A

# 6. Tools accessible through gateway
curl -s -X POST http://mcp-gateway.apps.${CLUSTER_DOMAIN}:8001/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18","capabilities":{},
    "clientInfo":{"name":"test-client","version":"1.0.0"}}}'
```

---

## Troubleshooting

### Helm chart version check fails

```
Error: chart requires kubeVersion: >= 1.34.0-0 ...
```

**Cause:** You're using the Red Hat-packaged chart which has a `kubeVersion`
constraint. OCP 4.20 runs Kubernetes 1.33.

**Fix:** Use the upstream chart instead:

```bash
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    oci://ghcr.io/containers/charts/kubernetes-mcp-server \
    --set ingress.host=mcp-server.apps.${CLUSTER_DOMAIN} \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"use-view-role","roleRef":{"name":"view","external":true}}]'
```

### Gateway API CRDs not found

```
error: the server doesn't have a resource type "httproutes"
```

**Cause:** Gateway API CRDs are not installed on OCP 4.20 by default.

**Fix:**

```bash
oc apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.1/standard-install.yaml
```

### MCPServerRegistration not becoming Ready

```bash
oc describe mcpsr my-mcp-server-reg -n openshift-mcp-server
oc logs -n ${GATEWAY_NS} deployment/mcp-gateway-controller
oc logs -n ${GATEWAY_NS} deployment/mcp-gateway
```

### MCP Server pod not starting

```bash
oc describe pod -n openshift-mcp-server -l app.kubernetes.io/name=kubernetes-mcp-server
oc logs -n openshift-mcp-server -l app.kubernetes.io/name=kubernetes-mcp-server
```

---

## Known Limitations on OCP 4.20

| Area | Status | Notes |
|------|--------|-------|
| MCP Server (core, config, helm, metrics, netedge) | ✅ Expected to work | Uses standard K8s APIs only |
| MCP Gateway (Red Hat Connectivity Link 1.3) | ⚠️ May need Gateway API CRDs installed manually | See Step 4 note |
| Kuadrant / Authorino (Auth) | ⚠️ Verify operator compatibility | May not be available for 4.20 |
| RHOAI 3.5 MCP Catalog | ❌ Not available | MCP Lifecycle Operator requires OCP 4.22 |
| Red Hat Support | ❌ Not covered | MCP is documented for OCP 4.22 only |

---

## Upgrade Path

When the cluster is upgraded to **OCP 4.22**, you can:

1. **Switch to the Red Hat-packaged chart** for supported install:
   ```bash
   helm upgrade openshift-mcp-server openshift-helm-charts/redhat-openshift-mcp-server \
       -n openshift-mcp-server \
       --set config.toolsets='{core,config,helm,metrics,netedge}' \
       --set ingress.host=mcp-server.apps.${CLUSTER_DOMAIN} \
       --set openshift=true \
       --set-json 'rbac.extraClusterRoleBindings=[{"name":"use-view-role","roleRef":{"name":"view","external":true}}]'
   ```

2. **Or switch to the RHOAI 3.5 MCP Catalog** (recommended if you have RHOAI):
   ```bash
   # Uninstall the Helm-based server
   helm uninstall openshift-mcp-server -n openshift-mcp-server

   # Enable the MCP Lifecycle Operator
   oc patch datasciencecluster default-dsc --type=merge \
     -p '{"spec":{"components":{"mcplifecycleoperator":{"managementState":"Managed"}}}}'

   # Enable the MCP Catalog dashboard
   oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
     --type=merge -p '{"spec":{"dashboardConfig":{"mcpCatalog":true}}}'

   # Then deploy from AI Hub → MCP servers in the dashboard
   ```

---

## Reference

| Resource | URL |
|----------|-----|
| Upstream community chart (no version gate) | `oci://ghcr.io/containers/charts/kubernetes-mcp-server` |
| OpenShift MCP Server GitHub | https://github.com/openshift/openshift-mcp-server |
| MCP Server for Red Hat OpenShift (OCP 4.22 docs) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_applications/mcp-server |
| Installing the MCP Gateway (RHCL 1.3) | https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install |
| RHOAI 3.5 MCP Catalog | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/index |
| Gateway API CRDs | https://gateway-api.sigs.k8s.io/guides/#installing-gateway-api |
