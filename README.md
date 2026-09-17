# Enterprise Traefik Proxy Ingress & FQDN Architecture on Red Hat OpenShift (AWS)

<!-- ======================================================================= -->
<!-- ARCHITECTURAL BADGES & CLUSTER TAXONOMY -->
<!-- ======================================================================= -->
<p align="center">
  <a href="https://www.redhat.com/en/technologies/cloud-computing/openshift"><img src="https://img.shields.io/badge/OpenShift-v4.14%2B-EE0000?style=for-the-badge&logo=redhatopenshift&logoColor=white" alt="OpenShift 4.14+"/></a>
  <a href="https://aws.amazon.com/rosa/"><img src="https://img.shields.io/badge/AWS-ROSA%20%2F%20NLB-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" alt="AWS ROSA"/></a>
  <a href="https://traefik.io/"><img src="https://img.shields.io/badge/Traefik%20Proxy-v3.0%2B-24A1C1?style=for-the-badge&logo=traefik&logoColor=white" alt="Traefik Proxy v3"/></a>
  <a href="https://gateway-api.sigs.k8s.io/"><img src="https://img.shields.io/badge/Gateway%20API-v1.x%20Standard-326CE5?style=for-the-badge&logo=kubernetes&logoColor=white" alt="Gateway API v1"/></a>
  <a href="https://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws"><img src="https://img.shields.io/badge/Organization-nubenetes-0969DA?style=for-the-badge&logo=github&logoColor=white" alt="Nubenetes Org"/></a>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/Security-Strict%20mTLS%20(TLS%201.3)-0A7EA4?style=flat-square&logo=shield&logoColor=white" alt="Strict mTLS"/>
  <img src="https://img.shields.io/badge/SCC-restricted--v2%20Compliant-4C51BF?style=flat-square&logo=redhat&logoColor=white" alt="OpenShift restricted-v2 SCC"/>
  <img src="https://img.shields.io/badge/DNS-AWS%20Route%2053%20%2B%20ExternalDNS-527FFF?style=flat-square&logo=amazonroute53&logoColor=white" alt="Route53 ExternalDNS"/>
  <img src="https://img.shields.io/badge/L4%20Ingress-PROXY%20Protocol%20v2-232F3E?style=flat-square&logo=amazon&logoColor=white" alt="PROXY Protocol v2"/>
  <img src="https://img.shields.io/badge/Protocol-HTTP%2F3%20%26%20gRPC-00ADD8?style=flat-square&logo=go&logoColor=white" alt="HTTP3 gRPC"/>
  <img src="https://img.shields.io/badge/Traffic-North--South%20%26%20East--West-38A169?style=flat-square&logo=diagram-next&logoColor=white" alt="Traffic Routing"/>
  <img src="https://img.shields.io/badge/License-Apache%202.0-blue.svg?style=flat-square" alt="License"/>
  <img src="https://img.shields.io/badge/Context-://github.com/nubenetes-8A2BE2?style=flat-square&logo=git&logoColor=white" alt="Repo Context"/>
</p>

---

## Executive Summary

This enterprise Proof of Concept (PoC) repository—hosted under the organization path `https://github.com/nubenetes` (repository reference: `://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws`)—establishes a reference architecture for advanced Fully Qualified Domain Name (FQDN) management, automated edge routing, and zero-trust microservice communication within **Red Hat OpenShift Container Platform v4.14+ (ROSA / self-managed OCP on AWS)**.

In mission-critical enterprise environments, default platform routing constructs often struggle to address the convergence of hybrid cloud domain delegation, sub-second route propagation without proxy reload overhead, multi-tenant RBAC boundaries, and East-West mutual TLS (mTLS) enforcement. This PoC implements and benchmark-tests two production-grade architectures:

1. **Solution A (Traefik CRD Stack):** A battle-tested architecture utilizing native Traefik v3.0+ Custom Resource Definitions (`IngressRoute`, `Middleware`, `TLSOption`, `ServersTransport`).
2. **Solution B (Kubernetes Gateway API Stack):** A forward-looking, standardized architecture implementing the official Kubernetes Gateway API specification (`GatewayClass`, `Gateway`, `HTTPRoute`, `BackendTLSPolicy`) driven by Traefik v3.0+'s native gateway controller.

Both solutions are integrated with **AWS Route 53** via automated ExternalDNS synchronization, terminate high-throughput edge traffic on **AWS Network Load Balancers (NLB)** with PROXY protocol v2, and strictly adhere to Red Hat OpenShift's default `restricted-v2` Security Context Constraints (SCC).

---

## Repository Overview Tags & Technical Taxonomy

The repository is tagged and indexed with the following domain taxonomy:

| Category | Taxonomy Tags | Technical Scope & Implementation |
| :--- | :--- | :--- |
| **Data Plane & Ingress** | `traefik`, `traefik-proxy`, `ingressroute`, `gateway-api`, `httproute` | Dual-engine routing: Traefik v3.0+ Custom Resource Definitions vs. Standard Kubernetes Gateway API v1.x data planes. |
| **Platform & Cloud** | `openshift`, `redhat-openshift`, `ocp4`, `rosa`, `aws` | Engineered for Red Hat OpenShift 4.14+ on AWS (ROSA), enforcing non-root execution (`restricted-v2` SCC). |
| **DNS & L4 Ingress** | `external-dns`, `route53`, `fqdn`, `nlb`, `aws-load-balancer-controller` | Automated DNS record synchronization in AWS Route 53, fronted by AWS Network Load Balancer (NLB) with PROXY protocol v2. |
| **Security & Zero-Trust** | `mtls`, `zero-trust`, `scc-restricted-v2`, `hsts`, `tls-1.3` | Inter-service cryptographic authentication using internal CAs (`RequireAndVerifyClientCert` and `BackendTLSPolicy`). |
| **Architecture & Discipline** | `platform-engineering`, `cloud-native`, `proof-of-concept`, `haproxy`, `service-mesh` | Enterprise architectural benchmarks contrasting sidecar-less mesh ingress against traditional OpenShift HAProxy routes. |

---

## Repository Structure

```
.
├── README.md                                    # Architectural specification & deployment guide
├── COMPARATIVE_MATRIX.md                        # Deep-dive comparative matrix & OpenShift dilemma analysis
├── LINKEDIN_NEWSLETTER_ES.md                    # Análisis e informe profundo para LinkedIn Newsletter (Español)
├── LINKEDIN_NEWSLETTER_EN.md                    # Executive architecture & Ingress report for LinkedIn Newsletter (English)
└── manifests/
    ├── common/
    │   ├── 00-namespaces-rbac-scc.yaml         # Namespaces, ServiceAccounts, RBAC, and SCC bindings
    │   ├── 01-mock-microservices.yaml          # Sample workloads (Service-A, Service-B) and TLS secrets
    │   └── 02-traefik-controller-deployment.yaml # Traefik Proxy v3.0+ controller & AWS NLB Service
    ├── solution-a-traefik-crds/
    │   ├── 01-ingressroute-north-south.yaml    # External edge IngressRoute for corporate FQDNs
    │   ├── 02-middleware-security.yaml         # HSTS, CORS, header mutations, and IP allowlist middlewares
    │   └── 03-ingressroute-east-west.yaml      # Internal mTLS IngressRoute & TLSOption configuration
    └── solution-b-gateway-api/
        ├── 01-gateway-class.yaml               # Cluster-scoped GatewayClass (traefik.io/gateway-controller)
        ├── 02-gateway-aws.yaml                 # AWS NLB Gateway with Route 53 ExternalDNS annotations
        ├── 03-httproute-north-south.yaml       # North-South HTTPRoute with native filters & URL rewrites
        └── 04-httproute-east-west.yaml         # East-West HTTPRoute with BackendTLSPolicy mTLS enforcement
```

### Direct File & Directory Navigation

- 📄 [**COMPARATIVE_MATRIX.md**](./COMPARATIVE_MATRIX.md) — Comprehensive comparative matrix & OpenShift dilemma analysis
- 📰 [**LINKEDIN_NEWSLETTER_ES.md**](./LINKEDIN_NEWSLETTER_ES.md) — Edición especial de análisis arquitectónico para LinkedIn Newsletter (Español)
- 📰 [**LINKEDIN_NEWSLETTER_EN.md**](./LINKEDIN_NEWSLETTER_EN.md) — Executive Cloud-Native & Ingress Architecture Report for LinkedIn Newsletter (English)
- 📁 [**manifests/common/**](./manifests/common/)
  - [`00-namespaces-rbac-scc.yaml`](./manifests/common/00-namespaces-rbac-scc.yaml) — OpenShift SCC & RBAC
  - [`01-mock-microservices.yaml`](./manifests/common/01-mock-microservices.yaml) — Workloads & TLS secrets
  - [`02-traefik-controller-deployment.yaml`](./manifests/common/02-traefik-controller-deployment.yaml) — Traefik v3.0+ controller & AWS NLB
- 📁 [**manifests/solution-a-traefik-crds/**](./manifests/solution-a-traefik-crds/)
  - [`01-ingressroute-north-south.yaml`](./manifests/solution-a-traefik-crds/01-ingressroute-north-south.yaml) — Edge IngressRoute & Route 53 sync
  - [`02-middleware-security.yaml`](./manifests/solution-a-traefik-crds/02-middleware-security.yaml) — HSTS, CORS & IP allowlists
  - [`03-ingressroute-east-west.yaml`](./manifests/solution-a-traefik-crds/03-ingressroute-east-west.yaml) — Zero-Trust mTLS & TLSOption
- 📁 [**manifests/solution-b-gateway-api/**](./manifests/solution-b-gateway-api/)
  - [`01-gateway-class.yaml`](./manifests/solution-b-gateway-api/01-gateway-class.yaml) — GatewayClass definition
  - [`02-gateway-aws.yaml`](./manifests/solution-b-gateway-api/02-gateway-aws.yaml) — AWS NLB Gateway & listeners
  - [`03-httproute-north-south.yaml`](./manifests/solution-b-gateway-api/03-httproute-north-south.yaml) — Edge HTTPRoute with native filters
  - [`04-httproute-east-west.yaml`](./manifests/solution-b-gateway-api/04-httproute-east-west.yaml) — East-West HTTPRoute & BackendTLSPolicy


---

## North-South Traffic Flow (Edge to Workload)

North-South ingress handles external user requests entering the AWS cloud, traversing the network perimeter, and reaching backend microservices residing inside isolated OpenShift namespaces.

### North-South Architecture Diagram

```mermaid
flowchart TD
    subgraph Edge_DNS ["1. Edge Resolution & L4 Ingress"]
        User(["<b>Client Consumer</b><br/>Browser / Mobile / API"])
        R53["<b>AWS Route 53 DNS</b><br/>• Auto-sync via ExternalDNS<br/>• Dual-Stack Alias A-Record"]
        NLB["<b>AWS Network Load Balancer (NLB)</b><br/>• Scheme: Internet-Facing (L4 TCP)<br/>• Port Mapping: 80:8000, 443:8443<br/>• PROXY Protocol v2 (Real IP)"]
        User -->|"1. Query FQDN"| R53
        R53 -.->|"Resolve Alias"| NLB
        User -->|"2. HTTPS Traffic"| NLB
    end

    subgraph Ingress_Tier ["2. Ingress Infrastructure Tier (traefik-system)"]
        Traefik["<b>Traefik Proxy v3.0+ Controller</b><br/>• SCC: restricted-v2 (UID 65532)<br/>• EntryPoints: web (:8000), websecure (:8443)<br/>• Direct OpenShift Router Bypass"]
    end

    NLB -->|"3. TCP + PROXY v2"| Traefik

    subgraph SolA ["3A. Solution A: Traefik CRDs"]
        direction TB
        IR_A["<b>IngressRoute (HTTP & HTTPS)</b><br/>• Port 8000: 301 Redirect<br/>• Port 8443: api.company.com<br/>• Secret: production-tls-secret<br/>• TLSOption: strict-tls-options"]
        MW["<b>Traefik Middlewares</b><br/>• HSTS 1-Year Preload & CSP<br/>• CORS & X-Forwarded-Host"]
        ServiceB_A["<b>Service-B Pods (Backend)</b><br/>• Namespace: traefik-crd-poc<br/>• Port 8080 (HTTP)"]
        IR_A --> MW --> ServiceB_A
    end

    subgraph SolB ["3B. Solution B: Gateway API"]
        direction TB
        GW["<b>AWS Edge Gateway</b><br/>• Listeners: http-edge & https-edge<br/>• Namespace Selector Delegation"]
        HR["<b>HTTPRoute (Redirect & API)</b><br/>• Port 80: RequestRedirect (301)<br/>• Port 443: api.company.com<br/>• Filters: HeaderModifier, URLRewrite"]
        ServiceB_B["<b>Service-B Pods (Backend)</b><br/>• Namespace: traefik-gateway-poc<br/>• Target Port: 8080 (HTTP)"]
        GW --> HR --> ServiceB_B
    end

    Traefik -->|"Route: Traefik CRDs"| SolA
    Traefik -->|"Route: Gateway API"| SolB
```

### Hop-by-Hop North-South Mechanics Breakdown

1. **DNS Resolution & AWS Route 53 Automation:**
   - External clients issue DNS queries for `company.com` or `api.company.com`.
   - **ExternalDNS Integration:** The ExternalDNS controller running inside OpenShift continuously monitors annotations on the `IngressRoute` (Solution A) or the `Gateway` / `HTTPRoute` (Solution B). It automatically registers AWS Route 53 Alias A-records mapped to the canonical AWS NLB DNS name (`k8s-traefik-*.elb.amazonaws.com`). Changes in route definitions synchronize with Route 53 within 60 seconds without manual DNS tickets.

2. **Edge Ingestion at AWS Network Load Balancer (NLB):**
   - The AWS NLB operates at Layer 4 (TCP), listening on port 80 and port 443, routing traffic across multiple Availability Zones to OpenShift worker nodes.
   - **Real IP Preservation:** The NLB is configured via `service.beta.kubernetes.io/aws-load-balancer-proxy-protocol: "*"` to inject PROXY protocol v2 headers into the TCP stream. This ensures the client's original IPv4/IPv6 source address is preserved and not masked by AWS NAT or node IP translation.

3. **OpenShift Router Layer Bypass:**
   - In standard OpenShift, external traffic passes through the default HAProxy Router (`ingresscontroller.operator.openshift.io`). In this architecture, we **bypass the default OpenShift router** by binding the AWS NLB directly to Traefik's `LoadBalancer` Service.
   - *Advantage:* Bypassing eliminates double-proxying, minimizes packet latency, avoids HAProxy configuration reload penalties, and allows Traefik full ownership of SNI routing, TLS termination, and header mutation.

4. **Security Context Constraints (SCC) & Non-Root Execution:**
   - OpenShift 4.14+ enforces the strict `restricted-v2` SCC. Traefik runs entirely unprivileged under user UID `65532` and GID `65532`.
   - To adhere to non-root port binding restrictions (ports `< 1024` are forbidden for unprivileged pods), Traefik binds internally to unprivileged port `8000` (HTTP), port `8443` (HTTPS), and port `9000` (Ping/Metrics). The AWS NLB handles external port mapping (`80 -> 8000`, `443 -> 8443`).

5. **FQDN Matching & Evaluation (Solution A vs. Solution B):**
   - **Solution A (Traefik CRDs):** Traefik evaluates the incoming HTTP `Host` header and TLS SNI against `IngressRoute.spec.routes[].match` rules (e.g., `Host('api.company.com') && PathPrefix('/api')`). If matched, Traefik applies the chained `Middleware` CRDs (`middleware-security-headers`, `middleware-forwarded-host-mutation`) to inject HSTS headers, mutate `X-Forwarded-Host`, enforce CORS, and route to `service-b`.
   - **Solution B (Gateway API):** Traefik evaluates the `Gateway`'s listeners (`hostname: "*.company.com"`). The matched listener delegates routing decisions to attached `HTTPRoute` resources in permitted namespaces (`allowedRoutes.namespaces.from: Selector`). Standardized filters (`RequestHeaderModifier`, `ResponseHeaderModifier`, `URLRewrite`) execute in memory, modifying request/response headers natively without requiring custom controller-specific CRDs.

---

## East-West Traffic Flow & Mutual TLS (mTLS)

East-West traffic governs internal, secure communication between services across different namespaces within the OpenShift cluster (e.g., frontend `service-a` calling backend `service-b`).

### East-West Architecture Diagram

```mermaid
flowchart TD
    subgraph Client_Tier ["1. Consumer Microservice Tier"]
        ServiceA["<b>Service-A Pod (Client)</b><br/>• OpenShift internal caller<br/>• Target FQDN:<br/>service-b.apps.cluster.local<br/>• Presents Root CA Client Cert"]
    end

    subgraph Ingress_Tier ["2. Ingress & Mesh Data Plane (traefik-system)"]
        TraefikInternal["<b>Traefik Proxy v3.0+ Router</b><br/>• Dedicated Internal Listener (:8443)<br/>• Terminates & Validates Client TLS 1.3"]
    end

    subgraph SolA ["3A. Solution A: Traefik CRDs"]
        direction TB
        TLSOption["<b>TLSOption: strict-mtls</b><br/>• Min TLS 1.3 Profile<br/>• ClientAuth: RequireAndVerify<br/>• CA: internal-ca-secret"]
        MW_IP["<b>Middleware: IPAllowList</b><br/>• Pod CIDR: 10.128.0.0/14<br/>• VPC CIDR: 10.0.0.0/16"]
        ST["<b>ServersTransport</b><br/>• Upstream Pod mTLS<br/>• Validates SAN:<br/>service-b.apps.cluster.local"]
        ServiceB_A["<b>Service-B Pod (Provider API)</b><br/>• Namespace: traefik-crd-poc<br/>• Listens on :8443 (HTTPS)<br/>• Validates Traefik Identity"]
        TLSOption --> MW_IP --> ST --> ServiceB_A
    end

    subgraph SolB ["3B. Solution B: Gateway API"]
        direction TB
        GW_Internal["<b>Gateway Listener (:8443)</b><br/>• Host: *.apps.cluster.local<br/>• Protocol: HTTPS Terminate"]
        HR_Filter["<b>HTTPRoute Security Filters</b><br/>• Injects Caller Identity<br/>• IPAllowList Extension Hook"]
        BTLP["<b>BackendTLSPolicy (v1alpha3)</b><br/>• Target Service: service-b<br/>• Validates SAN:<br/>service-b.apps.cluster.local<br/>• Trust: internal-ca-secret"]
        ServiceB_B["<b>Service-B Pod (Provider API)</b><br/>• Namespace: traefik-gateway-poc<br/>• Listens on :8443 (HTTPS)<br/>• Validates Traefik Identity"]
        GW_Internal --> HR_Filter --> BTLP --> ServiceB_B
    end

    ServiceA -->|"1. In-Cluster HTTPS Call"| TraefikInternal
    TraefikInternal -->|"Route: Traefik CRDs"| SolA
    TraefikInternal -->|"Route: Gateway API"| SolB
```

### Hop-by-Hop East-West & mTLS Mechanics Breakdown

1. **Targeting Internal Cluster FQDNs:**
   - Rather than communicating directly over ephemeral pod IPs or standard Kubernetes CoreDNS `service-b.namespace.svc.cluster.local` addresses without ingress visibility, `Service-A` routes through Traefik using dedicated internal FQDNs: `service-b.apps.cluster.local`.
   - CoreDNS or cluster `/etc/hosts` resolves `*.apps.cluster.local` to Traefik's internal cluster IP, establishing centralized observability, rate limiting, and access audit logging.

2. **mTLS Handshake Enforcement (`RequireAndVerifyClientCert`):**
   - When `Service-A` establishes an HTTPS connection to Traefik, Traefik initiates a TLS 1.3 handshake and requests a client certificate (`CertificateRequest`).
   - `Service-A` presents its X.509 certificate signed by the corporate internal Root CA (`internal-ca-secret`).
   - Traefik cryptographically verifies the entire certificate chain against `internal-ca-secret`. If the client certificate is missing, expired, or signed by an untrusted CA, the connection is instantly aborted with a TLS alert (`handshake failure`).

3. **Anti-Spoofing & Cross-Namespace Domain Validation:**
   - To prevent a rogue pod in an unprivileged namespace from intercepting or spoofing `service-b.apps.cluster.local`:
     - **SNI & Host Header Enforcement:** Traefik verifies that the TLS Server Name Indication (SNI) matches the HTTP `Host` header (`service-b.apps.cluster.local`). Mismatches result in an immediate `400 Bad Request`.
     - **IP Perimeter Filtering (Solution A Middleware):** Traefik applies the `middleware-internal-east-west-allowlist` ensuring the calling pod originates strictly within the OpenShift OVN-Kubernetes cluster pod network (`10.128.0.0/14`) or the AWS VPC subnet (`10.0.0.0/16`). External internet IPs hitting the internal listener are discarded.
     - **Gateway API Route Attachment Controls (Solution B):** The `aws-edge-gateway` explicitly restricts East-West listener attachments using `allowedRoutes.namespaces.from: Selector` with matching labels `solution: solution-b-gateway-api`. Workloads in unauthorized namespaces cannot bind routes to the internal domain.

4. **Zero-Trust Backend Encryption (Traefik to Upstream Pod):**
   - **Solution A (`ServersTransport`):** Traefik initiates an encrypted HTTPS connection to `service-b:8443`. It validates the backend pod's certificate against `internal-ca-secret` and verifies that the certificate's Subject Alternative Name (SAN) matches `service-b.apps.cluster.local`.
   - **Solution B (`BackendTLSPolicy`):** Gateway API's declarative `BackendTLSPolicy` configures Traefik to enforce strict TLS verification against `service-b`, anchoring trust directly to `internal-ca-secret` and rejecting any untrusted upstream endpoint.

---

## Production Deployment Steps (OpenShift CLI `oc`)

Follow these concrete, step-by-step commands to deploy and test both solutions on a Red Hat OpenShift v4.14+ cluster on AWS.

### Step 1: Clone Repository & Login to OpenShift

```bash
# Clone the repository (local path context: /home/inaki/github/traefik-fqdn-management-poc-openshift-aws)
cd /home/inaki/github/traefik-fqdn-management-poc-openshift-aws

# Authenticate with your OpenShift cluster
oc login --server=https://api.my-cluster.example.opentlc.com:6443 -u admin
```

### Step 2: Provision Namespaces, RBAC, and OpenShift SCCs

```bash
# Deploy namespaces, ServiceAccount, ClusterRole, and ClusterRoleBinding
oc apply -f manifests/common/00-namespaces-rbac-scc.yaml

# Grant the Traefik ServiceAccount permission to use OpenShift nonroot-v2 SCC
oc adm policy add-scc-to-user nonroot-v2 -z traefik-ingress-controller -n traefik-system

# Verify namespace creation and pod security labels
oc get namespaces -l app.kubernetes.io/part-of=traefik-platform
oc get namespaces -l app.kubernetes.io/part-of=traefik-poc
```

### Step 3: Deploy Mock Workloads and TLS Certificates

```bash
# Provision mock backend workloads (service-a, service-b) and TLS secrets
oc apply -f manifests/common/01-mock-microservices.yaml

# Wait for backend deployments to become ready in both namespaces
oc rollout status deployment/service-b -n traefik-crd-poc --timeout=120s
oc rollout status deployment/service-b -n traefik-gateway-poc --timeout=120s
```

### Step 4: Deploy Traefik Proxy v3.0+ Controller & AWS NLB

```bash
# Deploy Traefik Proxy v3.0+ with dual CRD and Gateway API providers enabled
oc apply -f manifests/common/02-traefik-controller-deployment.yaml

# Verify Traefik deployment rollout
oc rollout status deployment/traefik-ingress-controller -n traefik-system --timeout=120s

# Retrieve the AWS NLB external hostname provisioned by AWS Load Balancer Controller
export NLB_HOSTNAME=$(oc get svc traefik-loadbalancer -n traefik-system -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Provisioned AWS NLB Hostname: ${NLB_HOSTNAME}"
```

### Step 5: Deploy Solution A (Traefik CRD Stack)

```bash
# Apply Solution A Middlewares (HSTS, CORS, Host mutations, IP allowlists)
oc apply -f manifests/solution-a-traefik-crds/02-middleware-security.yaml

# Apply Solution A North-South IngressRoute (captures company.com & api.company.com)
oc apply -f manifests/solution-a-traefik-crds/01-ingressroute-north-south.yaml

# Apply Solution A East-West IngressRoute & TLSOption (strict mTLS)
oc apply -f manifests/solution-a-traefik-crds/03-ingressroute-east-west.yaml

# Inspect Solution A IngressRoute statuses
oc get ingressroutes -n traefik-crd-poc
oc get middlewares -n traefik-crd-poc
oc get tlsoptions -n traefik-crd-poc
```

### Step 6: Deploy Solution B (Kubernetes Gateway API Stack)

```bash
# Ensure Kubernetes Gateway API CRDs are installed on OpenShift 4.14+
oc get crd gateways.gateway.networking.k8s.io || \
  oc apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.1.0/standard-install.yaml

# Apply GatewayClass targeting Traefik Gateway Controller
oc apply -f manifests/solution-b-gateway-api/01-gateway-class.yaml

# Apply AWS Edge Gateway resource in traefik-system
oc apply -f manifests/solution-b-gateway-api/02-gateway-aws.yaml

# Apply Solution B North-South HTTPRoute with native filters (RequestHeaderModifier, URLRewrite)
oc apply -f manifests/solution-b-gateway-api/03-httproute-north-south.yaml

# Apply Solution B East-West HTTPRoute and BackendTLSPolicy
oc apply -f manifests/solution-b-gateway-api/04-httproute-east-west.yaml

# Inspect Gateway API statuses
oc get gatewayclass
oc get gateway aws-edge-gateway -n traefik-system
oc get httproutes -n traefik-gateway-poc
```

---

## Validation & Verification Testing

### 1. Validating North-South HTTP to HTTPS Redirection
```bash
# Test HTTP redirect on company.com (Resolution A / B via AWS NLB)
curl -sI -H "Host: company.com" "http://${NLB_HOSTNAME}" | grep -E "HTTP/|location:"
# Expected Output:
# HTTP/1.1 301 Moved Permanently
# location: https://company.com/
```

### 2. Validating Solution A (Traefik CRDs) HTTPS Route & Security Headers
```bash
# Test API route on api.company.com for Solution A
curl -k -sI -H "Host: api.company.com" "https://${NLB_HOSTNAME}/api/data" | grep -E "HTTP/|strict-transport-security|x-served-by|access-control-allow-origin"
# Expected Output:
# HTTP/2 200
# strict-transport-security: max-age=31536000; includeSubDomains; preload
# x-served-by-proxy: Traefik-v3-CRD-IngressRoute
# access-control-allow-origin: https://company.com
```

### 3. Validating Solution B (Gateway API) Native Header Filters & Rewrites
```bash
# Test API endpoint on Solution B verifying RequestHeaderModifier and ResponseHeaderModifier
curl -k -sI -H "Host: api.company.com" "https://${NLB_HOSTNAME}/api/v1/resource" | grep -E "HTTP/|strict-transport-security|x-gateway-engine|access-control-allow-origin"
# Expected Output:
# HTTP/2 200
# strict-transport-security: max-age=31536000; includeSubDomains; preload
# access-control-allow-origin: https://company.com
```

### 4. Validating East-West Mutual TLS (mTLS) Enforcement
```bash
# Execute internal test from Service-A Pod in traefik-crd-poc namespace
CLIENT_POD=$(oc get pod -l app=service-a -n traefik-crd-poc -o jsonpath='{.items[0].metadata.name}')

# Attempt 1: Call Service-B WITHOUT client certificate (Must Fail Handshake)
oc exec -n traefik-crd-poc "${CLIENT_POD}" -- \
  curl -k -s -o /dev/null -w "%{http_code}\n" "https://service-b.apps.cluster.local:8443/api/v1/internal" || echo "Handshake rejected as expected"

# Attempt 2: Call Service-B WITH valid internal client certificate (Must Succeed 200 OK)
oc exec -n traefik-crd-poc "${CLIENT_POD}" -- \
  curl -k -s --cert /var/run/secrets/tls/client.crt --key /var/run/secrets/tls/client.key \
  -H "Host: service-b.apps.cluster.local" \
  "https://traefik-loadbalancer.traefik-system.svc.cluster.local:8443/api/v1/internal"
```

---

## Architectural Comparison & Ingress Decision Flow

To guide platform engineering teams on whether to adopt Traefik CRDs (Solution A), Kubernetes Gateway API (Solution B), or stick with Native OpenShift Routes, evaluate the following architecture decision framework:

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 440, "nodePadding": 30, "diagramPadding": 32}}}%%
flowchart TD
    Start(["&nbsp;&nbsp;&nbsp;<b>Ingress Architecture Decision Flow</b>&nbsp;&nbsp;&nbsp;<br/>&nbsp;&nbsp;&nbsp;Red Hat OpenShift 4.14+ on AWS&nbsp;&nbsp;&nbsp;"])

    D1("<b>Step 1: OpenShift Native Ingress Fit</b><br/><br/>Do standard OpenShift Routes satisfy all basic<br/>ingress, wildcard FQDN & security needs?<br/>&nbsp;")

    NativeRoute["<b>Use Native OpenShift Routes</b><br/><br/>• Out-of-the-box Ingress Operator management<br/>• Zero additional controller overhead<br/>• Default *.apps cluster wildcard domain<br/>&nbsp;"]

    D2("<b>Step 2: Multi-Cloud Parity & Portability</b><br/><br/>Is cross-platform manifest portability across<br/>AWS EKS, GKE, or On-Prem required?<br/>&nbsp;")

    D3("<b>Step 3: Enterprise Multi-Tenancy & RBAC</b><br/><br/>Is strict Platform Admin vs. App Developer<br/>tri-persona separation mandatory?<br/>&nbsp;")

    SolB["<b>Solution B: Kubernetes Gateway API</b><br/><br/>• CNCF de jure standard (v1.x specification)<br/>• Role-oriented Gateway vs. HTTPRoute boundaries<br/>• Complete zero vendor lock-in across clouds<br/>&nbsp;"]

    SolA["<b>Solution A: Traefik Proxy CRDs</b><br/><br/>• Battle-tested IngressRoute & Middleware CRDs<br/>• High delivery velocity for unified engineering teams<br/>• Sub-second dynamic in-memory configuration reload<br/>&nbsp;"]

    Start --> D1
    D1 -->|"Yes: Basic"| NativeRoute
    D1 -->|"No: Advanced"| D2

    D2 -->|"Yes: Multi-Cloud"| SolB
    D2 -->|"No: OCP Only"| D3

    D3 -->|"Yes: Strict RBAC"| SolB
    D3 -->|"No: Unified Tooling"| SolA

    SolB ~~~ PadB[" "]
    SolA ~~~ PadA[" "]

    classDef decision fill:#f3f0ff,stroke:#7c3aed,stroke-width:2px;
    classDef outcome fill:#eef2ff,stroke:#4f46e5,stroke-width:1.5px;
    class D1,D2,D3 decision;
    class NativeRoute,SolA,SolB outcome;
    style PadA fill:none,stroke:none;
    style PadB fill:none,stroke:none;
```

For an exhaustive architectural matrix comparing **Traefik CRDs**, **Kubernetes Gateway API**, and **Native OpenShift Routes (HAProxy)** across security, multi-tenancy, AWS integration, and performance — including our **Multi-Cloud Companion Case Study on Feature-Flagged FQDN & East-West Security in GKE ([jenkins-2026](https://github.com/nubenetes/jenkins-2026))** — see:  
👉 [**COMPARATIVE_MATRIX.md**](./COMPARATIVE_MATRIX.md)

