# MCP Platform Setup 

End-to-end steps to stand up the MCP (Model Context Protocol) infrastructure
on OpenShift: MCP Server, MCP Gateway, Broker, Auth, and Authorization.

Based on the official Red Hat documentation:
- [OCP 4.22 — MCP server for Red Hat OpenShift](https://docs.redhat.com/en/documentation/openshift_container_platform/4.22/html/ai_applications/mcp-server)
- [Red Hat Connectivity Link 1.3 — Installing the MCP Gateway](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/1.3/html/installing_the_mcp_gateway/mcp-gateway-install)

> **Status**: MCP server for Red Hat OpenShift is a **Technology Preview** feature.
> Not supported with production SLAs. Do not include personal or sensitive
> information in prompts.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| OpenShift Container Platform | 4.22+ with cluster-admin |
| `oc` CLI | Installed and authenticated |
| Helm CLI | 3.x installed |
| MCP-compatible client | Claude Code, Cursor, VS Code, or OpenShift LightSpeed |
| Identity Provider | Entra ID (for OAuth-based auth, Step 8+) |

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
| OpenShift MCP Server GitHub | https://github.com/openshift/openshift-mcp-server |
| MCP Server Blog Post | https://www.redhat.com/en/blog/model-context-protocol-server-red-hat-openshift-now-available-technology-preview |
