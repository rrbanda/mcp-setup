# MCP Platform Setup 

End-to-end steps to stand up the MCP (Model Context Protocol) infrastructure
on OpenShift: MCP Server, MCP Gateway, Broker, Auth, and Authorization.

Based on the official Red Hat documentation:
- [OCP 4.22 — MCP server for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_applications/mcp-server)
- [Red Hat Connectivity Link 1.3 — Installing the MCP Gateway](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install)
- [RHOAI 3.5 — Working with the MCP Catalog](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/index)

> **Status**: MCP server for Red Hat OpenShift and the RHOAI MCP Catalog are both
> **Technology Preview** features. Not supported with production SLAs. Do not
> include personal or sensitive information in prompts.

---

## Version Compatibility

All MCP features require **OCP 4.22**. There are no supported workarounds for
older clusters.

| Component | OCP 4.19 | OCP 4.20 | OCP 4.21 | OCP 4.22 |
|-----------|----------|----------|----------|----------|
| Red Hat OpenShift AI 3.5 (general) | ✅ | ✅ | — | — |
| MCP Catalog / MCP Lifecycle Operator (RHOAI 3.5) | ❌ | ❌ | ❌ | ✅ |
| MCP Server Helm chart (OCP standalone) | ❌ | ❌ | ❌ | ✅ |
| MCP Gateway (Red Hat Connectivity Link 1.3) | ❌ | ❌ | ❌ | ✅ |

**Sources:**
- RHOAI 3.5 supports **OCP 4.19–4.20** for general installation
  ([install guide](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/pdf/installing_and_uninstalling_openshift_ai_self-managed)).
- The MCP Lifecycle Operator in RHOAI 3.5 explicitly requires **OCP 4.22 or later**
  ([MCP Catalog prerequisites](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/enabling-mcp-lifecycle-management)).
- The MCP Server Helm chart enforces `kubeVersion: ">= 1.34.0-0"` (Kubernetes 1.34+ = OCP 4.21+),
  but is only documented and supported under the
  [OCP 4.22 AI applications](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_applications/mcp-server) docs.

> **⚠️ If your cluster is on OCP 4.20 or earlier:** You cannot install MCP through
> any supported path. The Helm chart will fail the `kubeVersion` check
> (OCP 4.20 = Kubernetes 1.33, below the 1.34 minimum), and bypassing it
> puts you in an untested, unsupported configuration. **Upgrade to OCP 4.22
> before proceeding.**

### MCP servers available in the RHOAI 3.5 Catalog

The RHOAI 3.5 MCP Catalog includes the **Red Hat OpenShift MCP Server** (the same
server available as a standalone Helm chart) along with other servers. When
deployed through the catalog, the MCP Lifecycle Operator manages the server
lifecycle instead of Helm — but the **OCP 4.22 requirement is the same** for both
paths.

([Release notes — Developer Preview features](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/release_notes/developer-preview-features_relnotes),
[Red Hat Ecosystem Catalog — MCP servers](https://catalog.redhat.com/en/categories/ai/mcpservers))

| Server | Provider | Description |
|--------|----------|-------------|
| **Red Hat OpenShift** | Red Hat | Cluster management and troubleshooting |
| Red Hat Ansible Automation Platform | Red Hat | Playbook execution and configuration orchestration |
| Red Hat Insights | Red Hat | Platform intelligence and remediation recommendations |
| Confluent Cloud | Partner | Kafka and Flink streaming |
| EDB Postgres AI | Partner | Database queries and schema management |
| HashiCorp Terraform | Partner | Infrastructure as code |
| Microsoft Azure | Partner | Cloud resource management |
| Dynatrace | Partner | Performance monitoring and troubleshooting |
| MongoDB | Community | Document collections and RAG workflows |
| MariaDB | Community | Relational database connectivity |

In addition, RHOAI 3.5 provides **`rhoai-mcp`** (Developer Preview) — a separate
MCP server that lets AI coding agents (Claude Code, OpenCode, Gemini CLI)
interact with OpenShift AI itself for model recommendations, project management,
and Kubernetes manifest generation. This is distinct from the OpenShift cluster
management MCP server above.

---

## Installation Paths

There are two ways to deploy the OpenShift MCP Server. Both require **OCP 4.22+**.

| Path | When to use | Details |
|------|-------------|---------|
| **RHOAI 3.5 MCP Catalog** (recommended) | You already have OpenShift AI 3.5 on OCP 4.22 | Enable the MCP Lifecycle Operator in the DataScienceCluster, then deploy the Red Hat OpenShift MCP Server (and other servers) from the AI Hub dashboard — no Helm or manifests needed |
| **Standalone Helm chart** (this guide) | You do not have RHOAI, or need manual control | Install the MCP Server via Helm, then manually set up Gateway, Auth, and AuthZ (Steps 1–9 below) |

The RHOAI 3.5 catalog deploys the **same Red Hat OpenShift MCP Server** that the
standalone Helm chart installs, but manages it through the MCP Lifecycle
Operator instead of Helm. The MCP Gateway is a separate install in both paths.

If you have RHOAI 3.5 on OCP 4.22, see the [RHOAI MCP Catalog quick start](#rhoai-35-mcp-catalog-quick-start) section before following the manual Helm steps.

---

## RHOAI 3.5 MCP Catalog Quick Start

If OpenShift AI 3.5 is already installed on an **OCP 4.22+** cluster, you can
deploy MCP servers through the AI Hub dashboard instead of the manual Helm
steps below.

### Enable the MCP Lifecycle Operator

```bash
# Check your DataScienceCluster name
oc get datasciencecluster

# Enable the MCP Lifecycle Operator
oc patch datasciencecluster default-dsc --type=merge \
  -p '{"spec":{"components":{"mcplifecycleoperator":{"managementState":"Managed"}}}}'
```

### Enable the MCP Catalog dashboard

```bash
oc patch odhdashboardconfig odh-dashboard-config -n redhat-ods-applications \
  --type=merge -p '{"spec":{"dashboardConfig":{"mcpCatalog":true}}}'
```

After enabling, navigate to **AI Hub → MCP servers** in the OpenShift AI
dashboard to browse and deploy MCP servers from the catalog.

> For full details see: [Enable MCP server lifecycle management](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/enabling-mcp-lifecycle-management)
> and [Deploy and manage MCP servers](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/deploying-and-managing-mcp-servers).

---

## Prerequisites (Standalone Helm Install)

| Requirement | Details |
|-------------|---------|
| OpenShift Container Platform | **4.22+** with cluster-admin |
| `oc` CLI | Installed and authenticated |
| Helm CLI | 3.x installed |
| MCP-compatible client | Claude Code, Cursor, VS Code, or OpenShift LightSpeed |
| Identity Provider | Entra ID (for OAuth-based auth, Step 6+) |

---

## Step 1: Install the MCP Server Helm Chart

The official install path is Helm only. Two modes: read-only (recommended) or full access.

### Read-only access (recommended)

```bash
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    openshift-helm-charts/redhat-openshift-mcp-server \
    --set config.toolsets='{core,config,helm,metrics,netedge}' \
    --set ingress.host=mcp-server.apps.<cluster-domain> \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"use-view-role","roleRef":{"name":"view","external":true}}]'
```

### Full cluster access

```bash
helm upgrade -i -n openshift-mcp-server --create-namespace openshift-mcp-server \
    openshift-helm-charts/redhat-openshift-mcp-server \
    --set config.toolsets='{core,config,helm,metrics,netedge}' \
    --set ingress.host=mcp-server.apps.<cluster-domain> \
    --set openshift=true \
    --set-json 'rbac.extraClusterRoleBindings=[{"name":"admin-access","roleRef":{"name":"cluster-admin","external":true}}]'
```

Replace `<cluster-domain>` with your cluster's apps domain
(e.g., `oc get ingresses.config/cluster -o jsonpath='{.spec.domain}'`).

### Available toolsets

| Toolset | Description | Default | Status |
|---------|-------------|---------|--------|
| `core` | Pods, resources, events, namespaces | Yes | Technology Preview |
| `config` | View kubeconfig / cluster configuration | Yes | Technology Preview |
| `netedge` | NetEdge troubleshooting for OpenShift | No | Technology Preview |
| `helm` | Helm chart and release management | No | Technology Preview |
| `metrics` | Prometheus and Alertmanager queries | No | Technology Preview |
| `ossm` | OpenShift Service Mesh management | No | Technology Preview |
| `kubevirt` | KubeVirt virtual machine management | No | Developer Preview |
| `tekton` | Tekton pipeline management | No | Developer Preview |

Verify:

```bash
oc get pods -n openshift-mcp-server
# Expected: openshift-mcp-server pod Running
```

---

## Step 2: Revoke Access to Sensitive Custom Resources

Prevent AI agents from reading Secrets, ConfigMaps, and RBAC objects.

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
helm upgrade openshift-mcp-server openshift-helm-charts/redhat-openshift-mcp-server \
    -n openshift-mcp-server \
    --set-file config.deniedResources=deny-resources.toml
```

---

## Step 3: Configure RBAC Enforcement

Extend ClusterRoles and RoleBindings for fine-grained access.

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
helm upgrade openshift-mcp-server openshift-helm-charts/redhat-openshift-mcp-server \
    -n openshift-mcp-server \
    -f rbac-values.yaml
```

> Skip the ACM-related roles if this is a single-cluster setup without RHACM.

---

## Step 4: Install the MCP Gateway

Install via Red Hat Connectivity Link 1.3.

> See: [Installing the MCP Gateway (Red Hat Connectivity Link)](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install)

After installation, the controller automatically creates the HTTPRoute for
gateway access. The MCP gateway acts as a reverse proxy that aggregates
multiple MCP servers into a single endpoint with authentication and rate limiting.

Verify:

```bash
# Gateway controller running
oc get pods -n <gateway-namespace> -l "app.kubernetes.io/name=mcp-gateway"

# MCPGatewayExtension exists
oc get mcpgatewayextension -A

# Broker-router running
oc logs -n <gateway-namespace> deployment/mcp-gateway

# EnvoyFilter created
oc get envoyfilter -n <gateway-namespace> -l app.kubernetes.io/managed-by=mcp-gateway-controller
```

Replace `<gateway-namespace>` with the namespace where you installed the gateway.

---

## Step 5: Configure the MCP Gateway (Connect Server to Gateway)

### 5a. Verify the MCP server service name

```bash
oc get svc -n openshift-mcp-server
```

Note the service `NAME` — use it as `backendRefs.name` below.

### 5b. Create the HTTPRoute

```yaml
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
    namespace: <gateway-namespace>
    sectionName: mcp
  hostnames:
  - <mcp-server-host>
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /mcp
    backendRefs:
    - name: openshift-mcp-server
      port: 8080
```

```bash
export MCP_SERVER_HOST=mcp-server.apps.<cluster-domain>
export GATEWAY_NS=<gateway-namespace>

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

```yaml
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
```

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

### 5e. Verify tools are accessible through the gateway

```bash
# Initialize MCP session
curl -s -D /tmp/mcp_headers -X POST \
  http://mcp-gateway.apps.<cluster-domain>:8001/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{
    "protocolVersion":"2025-06-18","capabilities":{},
    "clientInfo":{"name":"test-client","version":"1.0.0"}}}'

# Extract session ID
SESSION_ID=$(grep -i "mcp-session-id:" /tmp/mcp_headers | cut -d' ' -f2 | tr -d '\r')
echo "Session ID: $SESSION_ID"

# List tools
curl -X POST http://mcp-gateway.apps.<cluster-domain>:8001/mcp \
  -H "Content-Type: application/json" \
  -H "mcp-session-id: $SESSION_ID" \
  -d '{"jsonrpc":"2.0","id":2,"method":"tools/list"}'

# Clean up
rm -f /tmp/mcp_headers
```

Tools should appear prefixed with `openshift_`.

---

## Step 6: Set Up MCP Gateway Authentication

OAuth-based authentication using an Identity Provider (e.g., Entra ID).

### 6a. Configure OAuth metadata discovery

```bash
oc set env deployment/mcp-gateway -n <gateway-namespace> \
  OAUTH_RESOURCE_NAME="MCP Server" \
  OAUTH_RESOURCE="http://mcp-gateway.apps.<cluster-domain>:8001/mcp" \
  OAUTH_AUTHORIZATION_SERVERS="https://login.microsoftonline.com/<tenant-id>/v2.0" \
  OAUTH_BEARER_METHODS_SUPPORTED="header" \
  OAUTH_SCOPES_SUPPORTED="openid,mcp-server,offline_access"
```

### 6b. Create HTTPRoute for OAuth discovery

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-gateway-oauth-metadata
  namespace: <gateway-namespace>
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    namespace: <gateway-namespace>
    sectionName: mcp
  hostnames:
  - <mcp-gateway-host>
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /.well-known/oauth-protected-resource
    backendRefs:
    - name: mcp-gateway
      port: 8080
```

```bash
export MCP_GATEWAY_HOST=mcp-gateway.apps.<cluster-domain>
export GATEWAY_NS=<gateway-namespace>

oc apply -f - <<EOF
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: mcp-gateway-oauth-metadata
  namespace: ${GATEWAY_NS}
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    namespace: ${GATEWAY_NS}
    sectionName: mcp
  hostnames:
  - ${MCP_GATEWAY_HOST}
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /.well-known/oauth-protected-resource
    backendRefs:
    - name: mcp-gateway
      port: 8080
EOF
```

### 6c. Configure IdP (Entra ID)

In the Azure Portal / Entra admin center:
1. Go to your App Registration
2. Under **Expose an API**, add a custom scope named `mcp-server`
3. Ensure scopes are available: `openid`, `offline_access`, `mcp-server`
4. Grant admin consent if required by your organization

### 6d. Mount IdP CA certificate for Authorino

```bash
oc create configmap entra-ca \
  --from-file=ca.crt=/path/to/entra-ca.crt \
  -n kuadrant-system

oc patch deployment authorino -n kuadrant-system --type=json -p='[
  {
    "op": "add",
    "path": "/spec/template/spec/volumes/-",
    "value": {"name": "entra-ca", "configMap": {"name": "entra-ca"}}
  },
  {
    "op": "add",
    "path": "/spec/template/spec/containers/0/volumeMounts/-",
    "value": {"name": "entra-ca", "mountPath": "/etc/ssl/certs/entra-ca.crt", "subPath": "ca.crt"}
  }
]'
```

### 6e. Apply the AuthPolicy

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: mcp-auth-policy
  namespace: <gateway-namespace>
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    sectionName: mcp
  defaults:
    when:
      - predicate: "!request.path.contains('/.well-known')"
    rules:
      authentication:
        'entra-id':
          jwt:
            issuerUrl: https://login.microsoftonline.com/<tenant-id>/v2.0
      response:
        unauthenticated:
          code: 401
          headers:
            'WWW-Authenticate':
              value: >-
                Bearer resource_metadata=http://<mcp-gateway-host>/.well-known/oauth-protected-resource/mcp
          body:
            value: |
              {"error":"Unauthorized","message":"Authentication required."}
```

```bash
export TENANT_ID=<your-entra-tenant-id>
export GATEWAY_NS=<gateway-namespace>
export MCP_GATEWAY_HOST=mcp-gateway.apps.<cluster-domain>

oc apply -f - <<EOF
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: mcp-auth-policy
  namespace: ${GATEWAY_NS}
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    sectionName: mcp
  defaults:
    when:
      - predicate: "!request.path.contains('/.well-known')"
    rules:
      authentication:
        'entra-id':
          jwt:
            issuerUrl: https://login.microsoftonline.com/${TENANT_ID}/v2.0
      response:
        unauthenticated:
          code: 401
          headers:
            'WWW-Authenticate':
              value: Bearer resource_metadata=http://${MCP_GATEWAY_HOST}/.well-known/oauth-protected-resource/mcp
          body:
            value: |
              {"error":"Unauthorized","message":"Authentication required."}
EOF
```

### 6f. Verify authentication

```bash
# OAuth discovery should return metadata
curl http://<mcp-gateway-host>/.well-known/oauth-protected-resource

# Unauthenticated request should return 401
curl -v http://<mcp-gateway-host>/mcp \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

---

## Step 7: Set Up MCP Gateway Authorization (Tool-Level)

Control which users can access which MCP tools via JWT claims.

### 7a. Configure IdP to include role claims

The issued OAuth token must include `resource_access` claims:

```json
{
  "resource_access": {
    "openshift-mcp-server/my-mcp-server-reg": {
      "roles": ["pods_list", "events_list", "nodes_list", "deployments_get"]
    }
  }
}
```

- Namespace must match the `MCPServerRegistration` CR namespace
- Roles represent the allowed tools

### 7b. Apply tool-level AuthPolicy

```yaml
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: mcp-tool-auth-policy
  namespace: openshift-mcp-server
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    sectionName: mcp
  rules:
    authentication:
      'sso-server':
        jwt:
          issuerUrl: https://login.microsoftonline.com/<tenant-id>/v2.0
    authorization:
      'tool-access-check':
        patternMatching:
          patterns:
            - predicate: >-
                request.headers['x-mcp-toolname'] in
                (has(auth.identity.resource_access) &&
                auth.identity.resource_access.exists(p,
                p == request.headers['x-mcp-servername'])
                ? auth.identity.resource_access[request.headers['x-mcp-servername']].roles
                : [])
    response:
      unauthenticated:
        headers:
          'WWW-Authenticate':
            value: >-
              Bearer resource_metadata=http://<mcp-gateway-host>/.well-known/oauth-protected-resource/mcp
        body:
          value: |
            {"error":"Unauthorized","message":"MCP Tool Access denied: Authentication required."}
      unauthorized:
        body:
          value: |
            {"error":"Forbidden","message":"MCP Tool Access denied: Insufficient permissions for this tool."}
```

```bash
export TENANT_ID=<your-entra-tenant-id>
export MCP_GATEWAY_HOST=mcp-gateway.apps.<cluster-domain>

oc apply -f - <<EOF
apiVersion: kuadrant.io/v1
kind: AuthPolicy
metadata:
  name: mcp-tool-auth-policy
  namespace: openshift-mcp-server
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: mcp-gateway
    sectionName: mcp
  rules:
    authentication:
      'sso-server':
        jwt:
          issuerUrl: https://login.microsoftonline.com/${TENANT_ID}/v2.0
    authorization:
      'tool-access-check':
        patternMatching:
          patterns:
            - predicate: |
                request.headers['x-mcp-toolname'] in (has(auth.identity.resource_access) && auth.identity.resource_access.exists(p, p == request.headers['x-mcp-servername']) ? auth.identity.resource_access[request.headers['x-mcp-servername']].roles : [])
    response:
      unauthenticated:
        headers:
          'WWW-Authenticate':
            value: Bearer resource_metadata=http://${MCP_GATEWAY_HOST}/.well-known/oauth-protected-resource/mcp
        body:
          value: |
            {"error":"Unauthorized","message":"MCP Tool Access denied: Authentication required."}
      unauthorized:
        body:
          value: |
            {"error":"Forbidden","message":"MCP Tool Access denied: Insufficient permissions for this tool."}
EOF
```

### 7c. Verify authorization

```bash
# Authenticate with a user that has the required role claims
# Try allowed tools: pods_list, events_list, nodes_list
# Try restricted tool: pods_delete — should return 403 Forbidden

# Check AuthPolicy status
oc get authpolicy -A

# View authorization logs
oc logs -n kuadrant-system -l authorino-resource=authorino
```

---

## Step 8: Connect an MCP Client

Add the following to your client configuration:

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

Accept the CA certificate when prompted by the client.

---

## Step 9: Verification Checklist

```bash
# 1. MCP Server running
oc get pods -n openshift-mcp-server

# 2. MCP Gateway running
oc get pods -n <gateway-namespace> -l "app.kubernetes.io/name=mcp-gateway"

# 3. MCPGatewayExtension exists
oc get mcpgatewayextension -A

# 4. MCPServerRegistration is Ready with tools discovered
oc get mcpsr -A

# 5. EnvoyFilter created
oc get envoyfilter -n <gateway-namespace> \
  -l app.kubernetes.io/managed-by=mcp-gateway-controller

# 6. AuthPolicy active
oc get authpolicy -A

# 7. Unauthenticated request returns 401
curl -s -o /dev/null -w "%{http_code}" \
  http://mcp-gateway.apps.<cluster-domain>/mcp

# 8. Authenticated request returns tools
# (use your IdP flow to obtain a Bearer token, then):
curl -X POST https://mcp-gateway.apps.<cluster-domain>/mcp \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

---

## Emergency: Revoke AI Agent Access

Multiple options, in order of urgency:

```bash
# 1. Remove specific tools
# In config.toml: enabled_tools = ["pods_list", "pods_get"]
# Or: disabled_tools = ["resources_delete", "pods_delete"]

# 2. Force read-only
# In config.toml: read_only = true

# 3. Disable destructive operations only
# In config.toml: disable_destructive = true

# 4. Remove the MCPServerRegistration (removes from gateway)
oc delete mcpsr my-mcp-server-reg -n openshift-mcp-server

# 5. Uninstall the MCP server entirely
helm uninstall openshift-mcp-server -n openshift-mcp-server

# 6. Revoke RBAC
oc delete clusterrolebinding <binding-name>
```

---

## Step Order

```
1. Install MCP Server (Helm chart)
   └─► 2. Revoke access to sensitive CRs (denied_resources)
         └─► 3. Configure RBAC (ClusterRoles, RoleBindings)
               └─► 4. Install MCP Gateway (Red Hat Connectivity Link 1.3)
                     └─► 5. Configure Gateway (HTTPRoute + MCPServerRegistration)
                           └─► 6. Authentication (OAuth / Entra ID + AuthPolicy)
                                 └─► 7. Authorization (tool-level via JWT claims)
                                       └─► 8. Connect MCP client
                                             └─► 9. Verify
```

---

## Reference

| Resource | URL |
|----------|-----|
| MCP Server for Red Hat OpenShift (OCP 4.22) | https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_applications/mcp-server |
| Installing the MCP Gateway (RHCL 1.3) | https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install |
| RHOAI 3.5 — Working with the MCP Catalog | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/index |
| RHOAI 3.5 — Enable MCP Lifecycle Operator | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/enabling-mcp-lifecycle-management |
| RHOAI 3.5 — Deploy and Manage MCP Servers | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/working_with_the_mcp_catalog/deploying-and-managing-mcp-servers |
| RHOAI 3.5 — Install Guide (supported OCP versions) | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/pdf/installing_and_uninstalling_openshift_ai_self-managed |
| RHOAI 3.5 — Release Notes (Tech Preview features) | https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.5/html/release_notes/technology-preview-features_relnotes |
| Red Hat Ecosystem Catalog — MCP Servers | https://catalog.redhat.com/en/categories/ai/mcpservers |
| OpenShift MCP Server GitHub | https://github.com/openshift/openshift-mcp-server |
| MCP Server Blog Post | https://www.redhat.com/en/blog/model-context-protocol-server-red-hat-openshift-now-available-technology-preview |
