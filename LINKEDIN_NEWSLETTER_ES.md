# 🚀 Traefik Proxy v3 vs. Kubernetes Gateway API en Red Hat OpenShift & AWS: El Fin del Dilema de Ingress Empresarial

*Edición Especial de Arquitectura Cloud-Native & Platform Engineering*  
**Repositorio Oficial en GitHub:** 👉 [**nubenetes/traefik-fqdn-management-poc-openshift-aws**](https://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws)

![LinkedIn Newsletter Cover](./assets/linkedin_newsletter_cover.jpg)

---

## 📌 Introducción: El Gran Cuello de Botella del Ingress en OpenShift

Si lideras o formas parte de un equipo de **Platform Engineering** gestionando **Red Hat OpenShift (ROSA u OCP on AWS)** a escala empresarial, es casi seguro que te has topado con este dilema arquitectónico:

> *"El objeto nativo `Route` de OpenShift (basado en HAProxy) es fantástico para la agilidad del desarrollador en el día 1, pero se convierte en una pared infranqueable cuando entran en juego requerimientos estrictos de Zero-Trust mTLS granular, mutación avanzada de cabeceras, soporte nativo para HTTP/3 (QUIC) y portabilidad multi-cloud sin vendor lock-in."*

¿La consecuencia directa? Muchas organizaciones terminan forzando la instalación de gigantescas mallas de servicios (*Service Meshes* basadas en Istio/Envoy) únicamente para resolver el cifrado inter-servicio (East-West), disparando el consumo de CPU/Memoria por los sidecars y multiplicando la complejidad operativa.

Para responder a este desafío con código y arquitectura real, hemos publicado un repositorio de referencia y prueba de concepto (*PoC*) con grado de producción:  
🔗 **[https://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws](https://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws)**

En este artículo desgranamos el análisis técnico profundo, la comparativa frente a frente entre **Traefik Proxy v3.0+ CRDs** y el nuevo estándar **Kubernetes Gateway API v1.x**, y cómo diseñar una topología de Ingress de alto rendimiento en AWS con cero recargas de proceso.

---

## 🛑 El Dilema de OpenShift: ¿Por qué las Routes Nativas se Quedan Cortas?

El Ingress Operator de Red Hat OpenShift desacopla la infraestructura mediante su router HAProxy. Sin embargo, en arquitecturas cloud modernas sobre AWS, emergen cuatro limitaciones críticas:

1. **Penalizaciones por Recarga de Proceso HAProxy:**  
   Aunque HAProxy ha incorporado endpoints dinámicos, cambios continuos en certificados TLS, rutas complejas o mutaciones de configuración provocan regeneraciones de plantilla y recargas del demonio. A escalas de miles de peticiones concurrentes, esto se traduce en picos de latencia (*p99 jitter*) y micro-cortes TCP.
2. **Carencia de mTLS Granular a Nivel de Ruta:**  
   OpenShift Routes soporta terminación `edge`, `reencrypt` o `passthrough`. En `edge`/`reencrypt`, el router no permite autenticar y verificar de forma declarativa certificados de cliente X.509 (`RequireAndVerifyClientCert`) validados contra Root CAs corporativas específicas por microservicio. En `passthrough`, la conexión se reenvía opaca al Pod, perdiendo toda capacidad de inspección L7, routing por path y manipulación de cabeceras.
3. **Snippets de Configuración Vulnerables:**  
   Para implementar cabeceras complejas (HSTS estricto, CORS corporativo, path regex rewrites), los desarrolladores recurren a la anotación `haproxy.router.openshift.io/snippet`. En clústeres empresariales regulados (banca, salud, seguros), SecOps desactiva estos snippets por riesgo crítico de inyección de configuración arbitraria.
4. **Restricción de Protocolos Cloud de Nueva Generación:**  
   Falta de soporte nativo out-of-the-box para **HTTP/3 sobre QUIC (UDP)** y multiplexación optimizada de streams gRPC.

---

## 🎯 El Desafío de los FQDNs: ¿Justifica la Gestión de Dominios (North-South & East-West) el Salto a Traefik / Gateway API?

Una de las preguntas arquitectónicas más críticas para los líderes de plataforma es:  
**¿Realmente la gestión de nombres de dominio totalmente cualificados (FQDNs) justifica por sí sola reemplazar o complementar el router nativo de OpenShift?**

La respuesta técnica contundente es **SÍ**.  
La razón fundamental radica en la enorme brecha que existe entre cómo OpenShift Routes fue diseñado históricamente y las exigencias de los ecosistemas cloud-native y Zero-Trust modernos.

### 1. La Trampa del Comodín `*.apps` en North-South
El modelo nativo de OpenShift está estructuralmente optimizado para un único dominio comodín (*wildcard*): `*.<subdominio>.apps.<cluster-name>.<baseDomain>`.  
Cuando una organización requiere exponer múltiples marcas, dominios de clientes (*white-label*) o FQDNs corporativos independientes (`api.empresa.com`, `pagos.banco.es`, `partner.portal.io`):
* **Falta de Automatización DNS Nativa:** OpenShift Routes no sincroniza registros DNS individuales contra AWS Route 53 para dominios externos arbitrarios. Obliga a depender de tickets manuales a redes o de scripts externos desacoplados.
* **La Proliferación de IngressControllers (*Sprawl* de Costes en AWS):** Para aislar certificados TLS o aplicar políticas de red diferenciadas por dominio en OpenShift nativo, el patrón oficial de Red Hat exige desplegar múltiples instancias del operador `IngressController`. Cada `IngressController` provisiona un nuevo AWS Network Load Balancer (NLB) y nuevos pods de HAProxy. En organizaciones con decenas de dominios, esto dispara exponencialmente la factura de AWS y el consumo de cómputo del clúster.
* **La Ventaja de Traefik / Gateway API:** Un único despliegue de Traefik o una sola instancia de `Gateway` multiplexa cientos de FQDNs corporativos sobre un único AWS NLB, seleccionando dinámicamente certificados TLS vía SNI y orquestando registros A/Alias en AWS Route 53 en tiempo real mediante ExternalDNS.

### 2. El Vacío Absoluto de FQDNs East-West en OpenShift Routes
El router de OpenShift es un componente **exclusivamente de borde (North-South)**. Las OpenShift Routes no están diseñadas para gobernar ni inspeccionar tráfico interno de servicio a servicio.
* Para llamadas entre microservicios, Kubernetes solo ofrece nombres DNS de ClusterIP (`servicio.namespace.svc.cluster.local`) en capa 4 sin verificación de identidad por cliente.
* Si una aplicación interna necesita invocar a otra usando un FQDN canónico (`servicio-b.apps.cluster.local` o `facturacion.internal.corp`) con autenticación mutua TLS (mTLS), en OpenShift tradicional solo existen dos caminos:
  1. **Hacer *Hairpinning*:** Forzar al tráfico a salir del clúster al balanceador externo de AWS y volver a entrar por el router perimetral. Esto añade latencia inaceptable, costes de transferencia de datos en AWS y riesgos graves de seguridad al exponer APIs internas al perímetro.
  2. **Implantar Red Hat OpenShift Service Mesh (Istio):** Pagar el descomunal peaje de CPU y RAM de inyectar sidecars de Envoy en cada pod (hasta 250MB RAM y 0.2 vCPU por réplica).

---

### 🏢 4 Casos de Uso Empresariales Reales: Cuándo este Requisito es Obligatorio

#### 🔹 Caso 1: Plataformas SaaS B2B Multi-Tenant y White-Label (North-South)
* **Requisito del Negocio:** Una plataforma SaaS financiera sobre OpenShift en AWS da servicio a más de 150 entidades bancarias. Cada cliente exige acceder a la API a través de su propio FQDN personalizado (`api.bancoprimario.com`, `auth.caja-ahorros.es`) con certificados EV/OV corporativos específicos.
* **Por qué falla OpenShift Routes:** Crear 150 rutas manuales con dominios externos sin automatización DNS colapsa el ciclo de vida de certificados y genera fricción operativa inmanejable.
* **Por qué Traefik / Gateway API es mandatorio:** Un único punto de entrada resuelve todos los FQDNs dinámicamente. Al crear un nuevo `HTTPRoute` con su hostname correspondiente, ExternalDNS genera automáticamente el alias en AWS Route 53 y Traefik asocia el Secret TLS adecuado en submilisegundos.

#### 🔹 Caso 2: Auditoría y Cumplimiento Zero-Trust Bancario / Salud (East-West)
* **Requisito del Negocio:** Normativas como **PCI-DSS 4.0, HIPAA o ENS (Nivel Alto)** exigen que toda comunicación entre el servicio de Pedidos (`namespace: e-commerce`) y el servicio de Pagos (`namespace: transacciones`) viaje cifrada con mTLS mutuo, validando que el llamante posee un certificado emitido por la CA de Seguridad y accediendo a través del FQDN interno auditable `pagos.internal.banco.local`.
* **Por qué falla OpenShift Routes:** Las Routes nativas no interceptan tráfico entre namespaces internos.
* **Por qué Traefik / Gateway API es mandatorio:** Traefik expone un EntryPoint interno que evalúa la llamada a `pagos.internal.banco.local`, ejecuta `RequireAndVerifyClientCert` contra `internal-ca-secret`, valida el SNI contra el header Host, e inyecta la identidad del cliente verificado antes de entregar al pod de destino. **Cero sidecars de Envoy, 100% de cumplimiento normativo.**

#### 🔹 Caso 3: Modernización de Monolitos y Fachada de API Canónica (East-West & Edge)
* **Requisito del Negocio:** Durante la migración de un sistema core bancario, componentes legados que aún residen en máquinas virtuales AWS EC2 fuera del clúster deben consumir microservicios en OpenShift bajo un FQDN canónico unificado (`core.internal.corp/api/v2`), requiriendo reescritura transparente de rutas y Canary Splitting (80% a v1, 20% a v2).
* **Por qué falla OpenShift Routes:** Las Routes nativas no soportan división ponderada avanzada de tráfico ni reescritura de paths declarativa sin snippets HAProxy vulnerables.
* **Por qué Traefik / Gateway API es mandatorio:** El middleware de reescritura (`URLRewrite` en Gateway API) y la ponderación nativa (`weight: 80 / weight: 20`) se definen de manera segura y estándar sobre el FQDN canónico.

#### 🔹 Caso 4: Estrategia Multi-Cloud y Recuperación ante Desastres (DR)
* **Requisito del Negocio:** La organización opera su producción principal en OpenShift sobre AWS (ROSA), pero mantiene un clúster secundario de Disaster Recovery en AWS EKS o Google Cloud GKE. Ambos clústeres deben publicar exactamente los mismos FQDNs internos y externos.
* **Por qué falla OpenShift Routes:** Las APIs `route.openshift.io/v1` no existen en EKS ni en GKE. El equipo se ve forzado a mantener dos repositorios de GitOps paralelos con manifiestos divergentes.
* **Por qué Gateway API es mandatorio:** El manifiesto `HTTPRoute` con sus reglas de FQDN es 100% portable. El mismo fichero YAML se aplica sin cambios en OpenShift, EKS o GKE.

---

## ⚔️ La Batalla de Paradigmas: Solución A vs. Solución B

Para superar estas limitaciones sobre OpenShift 4.14+ en AWS, evaluamos e implementamos los dos paradigmas líderes del ecosistema cloud-native:

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 440, "nodePadding": 30, "diagramPadding": 32}}}%%
flowchart TD
    Start(["&nbsp;&nbsp;&nbsp;<b>Ingress Architecture Decision Flow</b>&nbsp;&nbsp;&nbsp;<br/>&nbsp;&nbsp;&nbsp;Red Hat OpenShift 4.14+ on AWS&nbsp;&nbsp;&nbsp;"])

    D1("<b>Paso 1: Ajuste de OpenShift Routes</b><br/><br/>¿Cubren las Routes estándar todas las necesidades<br/>básicas de ingress, FQDN comodín y seguridad?<br/>&nbsp;")

    NativeRoute["<b>Usar Native OpenShift Routes</b><br/><br/>• Gestión estándar con Ingress Operator<br/>• Cero overhead de controladores adicionales<br/>• Dominio wildcard *.apps por defecto<br/>&nbsp;"]

    D2("<b>Paso 2: Paridad y Portabilidad Multi-Cloud</b><br/><br/>¿Se requiere portabilidad declarativa de manifiestos<br/>entre AWS EKS, GKE o entornos On-Prem?<br/>&nbsp;")

    D3("<b>Paso 3: Multi-Tenancy Empresarial y RBAC</b><br/><br/>¿Es mandatoria la separación estricta tri-persona<br/>(Admin de Plataforma vs. Desarrollador de Apps)?<br/>&nbsp;")

    SolB["<b>Solución B: Kubernetes Gateway API</b><br/><br/>• Estándar CNCF de jure (especificación v1.x)<br/>• Desacoplamiento de roles Gateway vs. HTTPRoute<br/>• Cero vendor lock-in entre proveedores cloud<br/>&nbsp;"]

    SolA["<b>Solución A: Traefik Proxy CRDs</b><br/><br/>• CRDs probadas en batalla (IngressRoute, Middleware)<br/>• Máxima velocidad para equipos con tooling unificado<br/>• Recarga dinámica en memoria en submilisegundos<br/>&nbsp;"]

    Start --> D1
    D1 -->|"Sí: Básico"| NativeRoute
    D1 -->|"No: Avanzado"| D2

    D2 -->|"Sí: Multi-Cloud"| SolB
    D2 -->|"No: Solo OCP"| D3

    D3 -->|"Sí: RBAC Estricto"| SolB
    D3 -->|"No: Herramientas Unificadas"| SolA

    SolB ~~~ PadB[" "]
    SolA ~~~ PadA[" "]

    classDef decision fill:#f3f0ff,stroke:#7c3aed,stroke-width:2px;
    classDef outcome fill:#eef2ff,stroke:#4f46e5,stroke-width:1.5px;
    class D1,D2,D3 decision;
    class NativeRoute,SolA,SolB outcome;
    style PadA fill:none,stroke:none;
    style PadB fill:none,stroke:none;
```

---

### 🅰️ Solución A: Traefik Custom Resource Definitions (CRDs)

*Basado en:* `IngressRoute`, `Middleware`, `TLSOption`, `ServersTransport`

* **Mayor Fortaleza:** **Velocidad de entrega y madurez operacional inmediata.**  
  Las CRDs de Traefik han sido probadas en batalla durante años. La reactividad de Go permite hot-swapping de tablas de enrutamiento en milisegundos directamente en memoria, sin recargas de proceso ni conexiones perdidas.
* **Seguridad Declarativa con Middlewares:**  
  Permite encadenar tuberías de middleware reutilizables para inyección/mutación de cabeceras, CORS whitelisting, forzado de HSTS con preload y restricción perimetral de rangos CIDR (`ipAllowList`) para OVN-Kubernetes (`10.128.0.0/14`) y subnets de AWS VPC (`10.0.0.0/16`).
* **Compromiso / Riesgo:**  
  *Vendor lock-in*. La configuración está atada a las APIs propietarias de Traefik (`traefik.io/v1alpha1`). Cambiar de controlador de Ingress en el futuro requiere reescribir todos los manifiestos.

---

### 🅱️ Solución B: Kubernetes Gateway API (CNCF Standard v1.x)

*Basado en:* `GatewayClass`, `Gateway`, `HTTPRoute`, `BackendTLSPolicy`

* **Mayor Fortaleza:** **Estándar de la industria *de jure* y portabilidad absoluta.**  
  Especificación oficial de Kubernetes SIG-Network (`gateway.networking.k8s.io`). El plano de datos subyacente puede reemplazarse (de Traefik a Envoy Gateway, Cilium o F5) sin alterar ni una sola línea de los manifiestos de los equipos de aplicación.
* **Separación de Roles Tri-Persona en RBAC:**  
  Resuelve de raíz el conflicto de gobernanza en grandes organizaciones:
  1. *Infrastructure Provider:* Define el `GatewayClass`.
  2. *Platform Admin:* Controla el objeto `Gateway` (listeners NLB, puertos, secretos TLS maestros, namespaces autorizados).
  3. *Application Developer:* Gestiona sus propios `HTTPRoute` (reglas de matching, reescritura de URLs y backend refs) sin privilegios sobre los balanceadores ni la infraestructura de red.
* **Compromiso / Curva:**  
  Mayor abstracción inicial y necesidad de dominar el uso de `ReferenceGrant` para delegación segura de secretos entre namespaces.

---

## 📊 Matriz Comparativa de Ingeniería

| Dimensión de Evaluación | Solución A: Traefik CRDs (`IngressRoute`) | Solución B: Gateway API (`Gateway` / `HTTPRoute`) | OpenShift Native Routes (HAProxy) |
| :--- | :--- | :--- | :--- |
| **Estandarización & Lock-in** | ⚠️ **Propietario.** Fuerte acoplamiento a Traefik Proxy. | 🛡️ **Estándar CNCF v1.x.** Cero lock-in; portabilidad entre OCP, EKS y GKE. | ⚠️ **Propietario Red Hat.** No portable a Kubernetes upstream vanilla. |
| **Integración AWS (NLB + Route 53)** | Directa vía anotaciones en Service `LoadBalancer` + ExternalDNS. | Asignación nativa de infraestructura en el recurso `Gateway`. | Gestión automática vía Ingress Operator para el comodín `*.apps`. |
| **Mutación de Cabeceras / URLs** | Extensa y desacoplada mediante CRD `Middleware`. | Estandarizada mediante filtros nativos (`RequestHeaderModifier`, `URLRewrite`). | Limitada o dependiente de snippets HAProxy no declarativos. |
| **Aislamiento Multi-Tenancy & RBAC** | Fragmentado; requiere RBAC estricto o webhooks validadores. | **Óptimo.** Desacoplamiento estricto en 3 capas de responsabilidad. | Centrado en el namespace del desarrollador; políticas poco delegables. |
| **Tráfico East-West y mTLS** | Ligero vía `TLSOption` y `ServersTransport` (sin sidecars). | Estandarizado vía `BackendTLSPolicy` (`gateway.networking.k8s.io`). | Requiere OpenShift Service Mesh (Istio sidecars), alto overhead. |
| **Penalización por Cambios** | **0 ms.** Actualización concurrente en memoria en Go. | **0 ms.** Reconciliación dinámica en memoria. | Recarga periódica del proceso HAProxy ante eventos de churn. |

---

## 🌐 Flujo de Tráfico North-South: De Route 53 al Microservicio sin HAProxy

En ambos enfoques demostrados en el repositorio, se implementa un bypass completo del Ingress Operator tradicional de OpenShift para maximizar el rendimiento:

```
[ Cliente / API Consumer ]
            │
            ▼
    AWS Route 53 DNS (Alias Dual-Stack gestionado por ExternalDNS)
            │
            ▼
    AWS Network Load Balancer (NLB L4 TCP)
    • PROXY Protocol v2 activo (Preservación de Real Client IP)
            │
            ▼
    Bypass Directo del Router OpenShift
            │
            ▼
    Traefik Proxy v3.0+ Controller (Namespace: traefik-system)
    • Cumplimiento SCC OpenShift: nonroot-v2 / restricted-v2 (UID 65532)
    • Puertos de entrada: web (:8000), websecure (:8443)
            │
    ┌───────┴──────────────────────────────┐
    ▼                                      ▼
[ Solución A: IngressRoute ]          [ Solución B: HTTPRoute ]
  • Validación Host (company.com)       • Filtros de Header Request/Response
  • Middleware HSTS + CORS              • URL Rewrite / Prefix Matching
  • Enrutamiento al Service backend     • Enrutamiento al Service backend
    │                                      │
    └──────────────────┬───────────────────┘
                       ▼
          [ Backend Pod Microservice ]
```

### Claves de Ingeniería Implementadas:
* **PROXY Protocol v2:** El NLB de AWS entrega el tráfico TCP en capa 4 inyectando el encabezado PROXY v2. Traefik está configurado con `trustedIPs` hacia la VPC para recuperar la IP de origen real del cliente antes de cualquier evaluación de reglas.
* **Seguridad en OpenShift (SCC):** El despliegue de Traefik no requiere privilegios de `root` ni SCC `privileged`. Cumple estrictamente con el SCC `nonroot-v2` / `restricted-v2` de OpenShift ejecutándose bajo el UID no privilegiado `65532`.

---

## 🔒 Tráfico East-West y Zero-Trust: mTLS Sin el Overhead de un Service Mesh

Uno de los mayores hallazgos de este PoC es cómo lograr **mTLS mutuo criptográficamente riguroso entre microservicios en distintos namespaces** sin pagar el peaje de Istio:

* **El Problema Típico:** Para cumplir normativas como PCI-DSS, HIPAA o ENS Alto, los equipos despliegan Envoy sidecars en cada Pod. Esto consume 100MB–250MB de RAM y 0.2 vCPU adicionales por cada réplica, sumando gigabytes de desperdicio en clústeres con cientos de microservicios.
* **La Realidad de CoreDNS en OpenShift:** En OpenShift 4.x, los equipos de aplicaciones **no pueden modificar las tablas ni zonas del CoreDNS interno**. El ConfigMap `dns-default` en el namespace `openshift-dns` está estrictamente reconciliado por el **OpenShift DNS Operator** (`dns.operator.openshift.io`), que revierte cualquier cambio manual en segundos. Además, declarar zonas DNS a nivel de clúster (`dnses.operator.openshift.io/default`) exige privilegios de `cluster-admin` (inaccesibles para desarrolladores en entornos multi-inquilino) y genera el peligro de *Split-Brain DNS*.
* **La Solución Demostrada en el Repo: Split-Horizon Ingress con Manifiestos IngressRoute + Middleware:**  
  En lugar de hackear DNS o inyectar sidecars, Traefik actúa como plano de enrutamiento interno mediante el tándem declarativo **`IngressRoute` + `Middleware`**.

  > 💡 **¿Qué FQDN se utiliza para el tráfico East-West? (Los 3 Escenarios Reales):**  
  > En este PoC empleamos didácticamente `service-b.apps.cluster.local` para mimetizar la convención de subdominio `apps` de OpenShift. No obstante, en producción las arquitecturas empresariales suelen adoptar uno de estos tres patrones:  
  > 1. **El estándar nativo y universal de Kubernetes/OpenShift:** `<servicio>.<namespace>.svc.cluster.local` (por ejemplo: `service-b.traefik-crd-poc.svc.cluster.local` o `service-b.backend.svc.cluster.local`). Es el FQDN que **CoreDNS resuelve por defecto de forma 100% nativa y sin intervención de ningún operador**.  
  > 2. **Zonas privadas corporativas (AWS Route 53 Private Hosted Zones):** Dominios como `service-b.internal.empresa.com` o `*.corp.local`, asociados a la VPC donde reside el clúster.  
  > 3. **El FQDN canónico público unificado (`api.empresa.com`):** Consumido internamente redirigiendo el tráfico localmente mediante `hostAliases` en el Pod consumidor hacia la IP interna de Traefik.

  1. **Resolución Nativa sin Tocar CoreDNS:** `Service-A` inicia una llamada HTTPS utilizando el FQDN de servicio (sea el estándar nativo de Kubernetes `https://service-b.traefik-crd-poc.svc.cluster.local:8443/api/v1/internal`, una zona privada o el alias configurado en el PoC `service-b.apps.cluster.local`), presentando su certificado de cliente emitido por la CA corporativa interna.
  2. **Intercepción y Validación Criptográfica en IngressRoute:** El recurso `IngressRoute` interno de Traefik ([`03-ingressroute-east-west.yaml`](./manifests/solution-a-traefik-crds/03-ingressroute-east-west.yaml)) intercepta el tráfico en su EntryPoint interno. Vincula el objeto `TLSOption` (`strict-mtls-option`), validando el certificado de cliente contra el secreto `internal-ca-secret` con `clientAuthType: RequireAndVerifyClientCert`. Si el certificado falta, expiró o no pertenece a la CA interna, Traefik aborta la conexión inmediatamente en el handshake TLS.
  3. **Cadena de Middlewares de Seguridad y Mutación:** Traefik ejecuta su pipeline de Middlewares antes de entregar la petición:
     - **`middleware-internal-east-west-allowlist`:** Aplica un filtro de capa IP (`ipAllowList`) restringiendo el tráfico exclusivamente al CIDR de Pods de OpenShift (`10.128.0.0/14`) y la VPC (`10.0.0.0/16`), bloqueando accesos no autorizados.
     - **`middleware-forwarded-host-mutation`:** Si el microservicio de destino espera recibir el FQDN corporativo canónico (`Host: api.empresa.com`) para validar tokens JWT o cabeceras CORS, el Middleware de cabeceras de Traefik muta el `Host` e inyecta `X-Forwarded-Host: api.empresa.com` y `X-Forwarded-Proto: https` de forma totalmente transparente.
  4. **Salto Backend Cifrado con ServersTransport:** Finalmente, Traefik abre una conexión HTTPS segura hacia los Pods de `service-b`, validando el SAN y el certificado del pod upstream mediante `ServersTransport` (Solución A) o `BackendTLSPolicy` (Solución B).

**Resultado:** Cifrado Zero-Trust de extremo a extremo, trazable, verificable por auditoría y con **cero sidecars inyectados y cero modificaciones en el operador de DNS de OpenShift**.

---

## 🎛️ El Espejo Multi-Cloud: Cómo lo Resuelve con Feature Flags el Repo Hermano 'jenkins-2026' en GKE

Una de las preguntas más recurrentes al diseñar plataformas cloud-native es:  
**¿Es obligatorio casarse con un modelo único (solo Ingress perimetral vs. Backend TLS vs. Service Mesh completo)? ¿O puede una plataforma ofrecer estas soluciones como capacidades conmutables mediante *Feature Flags*?**

En la organización **nubenetes**, el repositorio complementario de grado empresarial:  
👉 **[github.com/nubenetes/jenkins-2026](https://github.com/nubenetes/jenkins-2026)**  
ofrece una respuesta fascinante a este problema sobre **Google Kubernetes Engine (GKE)**. Mientras que en este repositorio sobre OpenShift y AWS demostramos cómo evitar los cuellos de botella de OpenShift Routes mediante Traefik Proxy v3 y Gateway API, en `jenkins-2026` se estandariza el borde con **Kubernetes Gateway API** (`gke-l7-global-external-managed`) y se orquesta el eje de seguridad intra-clúster mediante **Feature Flags declarativos**, permitiendo alternar entre tres niveles de seguridad sin modificar el código de las aplicaciones.

### 🧩 Los 3 Niveles de Seguridad Activados por Feature Flags en `jenkins-2026`

```mermaid
%%{init: {"flowchart": {"wrappingWidth": 400, "nodePadding": 28, "diagramPadding": 30}}}%%
flowchart LR
    subgraph Client["<b>Cliente Externo</b>"]
      User(["<b>Navegador / API Consumer</b><br/>Tráfico Web & APIs Externas"])
    end

    subgraph Edge["<b>Perímetro: Ingress Gateway</b>"]
      GW["<b>GKE Gateway API (L7)</b><br/>• Clase: gke-l7-global-external-managed<br/>• Terminación TLS 1.3 con Certs Wildcard<br/>• Autenticación Perimetral: Google IAP"]
    end

    subgraph Flag0["<b>Opción 0: Default (none)</b>"]
      Pod0["<b>Pod Backend (HTTP Plano)</b><br/>• Tráfico plano en red privada VPC<br/>• Cifrado transparente WireGuard eBPF<br/>• Segmentación L3/L4 NetworkPolicies"]
    end

    subgraph Flag1["<b>Opción 1: backend-tls (Flag ON)</b>"]
      direction TB
      BTP["<b>BackendTLSPolicy (Gateway API)</b><br/>• Validación estricta contra CA interna<br/>• ConfigMap: ca.crt del clúster<br/>• Re-cifrado HTTPS en salto LB → Pod"]
      Pod1["<b>Pod Backend (HTTPS Seguro)</b><br/>• Servidor HTTPS con certificado propio<br/>• Cero sidecars de Envoy (0% overhead)"]
      BTP --> Pod1
    end

    subgraph Flag2["<b>Opción 2: cloud-service-mesh (Flag ON)</b>"]
      direction TB
      Proxy["<b>istio-proxy Sidecar (CSM)</b><br/>• Identidad criptográfica SPIFFE<br/>• Inyección gestionada por Cloud Fleet<br/>• Puerto LB :8080 en modo PERMISSIVE"]
      Pod2["<b>Pod Backend (mTLS Estricto)</b><br/>• PeerAuthentication STRICT (mTLS)<br/>• Reglas L7 AuthorizationPolicy"]
      Proxy --> Pod2
    end

    User ==>|"HTTPS (TLS 1.3)"| GW
    GW -.->|"HTTP plano"| Pod0
    GW -.->|"HTTPS Re-encrypt"| BTP
    GW -.->|"mTLS SPIFFE"| Proxy

    classDef edge fill:#eef2ff,stroke:#4f46e5,stroke-width:2px;
    classDef opt fill:#f8fafc,stroke:#64748b,stroke-width:1.5px;
    classDef secure fill:#f0fdf4,stroke:#16a34a,stroke-width:2px;
    classDef user fill:#fefce8,stroke:#ca8a04,stroke-width:1.5px;
    class GW edge;
    class Pod0 opt;
    class BTP,Pod1,Proxy,Pod2 secure;
    class User user;
```

#### 1. Nivel 0: `none` (Postura por Defecto / Máxima Simplicidad)
* **Configuración del Flag:** `gateway.backendTls.enabled: false` y `serviceMesh.mode: none`.
* **Tráfico North-South & FQDNs Perimetrales:** El Gateway L7 gestionado por Google (`gke-l7-global-external-managed`) termina el TLS perimetral con certificados comodín gestionados para todos los FQDNs públicos (`app.jenkins2026.nubenetes.com`, `jenkins.jenkins2026...`) y valida autenticación corporativa con **Google Identity-Aware Proxy (IAP)**. El salto interno desde el balanceador al pod viaja en HTTP plano dentro de la VPC privada de Google.
* **Tráfico East-West sobre FQDNs de Servicio:** La comunicación interna entre microservicios (de servicio a servicio o desde runners de CI) se realiza directamente a través de los **FQDNs canónicos de Kubernetes** (`<servicio>.<namespace>.svc.cluster.local`) en HTTP plano en Capa 7.
* **Seguridad Subyacente:** La red no está desprotegida: Google cifra el tráfico en la capa de red física, y Dataplane V2 (eBPF Cilium) aplica cifrado transparente nodo a nodo vía **WireGuard** (`in_transit_encryption_config`) junto con `NetworkPolicies` estrictas en modo *default-deny*.
* **Limitación:** El cifrado WireGuard es a nivel de nodo de infraestructura; **no proporciona identidad criptográfica por carga de trabajo** ni verificación en el FQDN de servicio.
* **Veredicto:** Cero sobrecoste operacional, cero consumo extra de memoria/CPU, ideal para entornos de desarrollo, PoCs o plataformas donde la red privada y las NetworkPolicies se consideran frontera de confianza suficiente.

#### 2. Nivel 1: `backend-tls` (Re-cifrado sobre FQDN de Servicio Sin Sidecars)
* **Configuración del Flag:** `gateway.backendTls.enabled: true` (o variable de entorno `JENKINS2026_GATEWAY_BACKEND_TLS_ENABLED=true`).
* **Mecanismo:** Instala automáticamente `cert-manager` y una Autoridad de Certificación (CA) interna (`ClusterIssuer`). Despliega la directiva estándar **`BackendTLSPolicy`** de Gateway API (`gateway.networking.k8s.io`).
* **El FQDN de Servicio como Doble Ancla Criptográfica:**  
  En `jenkins-2026`, el FQDN interno del servicio (`<servicio>.<namespace>.svc.cluster.local`, e.g., `headlamp.headlamp.svc.cluster.local`) cumple una función dual imprescindible en el `BackendTLSPolicy` (`validation.hostname`):
  1. Es el **SNI** que el balanceador L7 de Google envía en el handshake TLS hacia el pod.
  2. Es el **SAN (Subject Alternative Name)** contra el que el balanceador valida el certificado servido por el pod, verificando la cadena contra el ConfigMap `jenkins-2026-backend-tls-ca` (`ca.crt`).
  Esto impide que un pod malicioso o comprometido en otro namespace pueda suplantar el tráfico del servicio.
* **Tráfico East-West:** Los clientes internos del clúster (como Backstage invocando a Grafana o scripts de prueba de integración) pueden invocar el FQDN del servicio directamente en `https://<servicio>.<namespace>.svc.cluster.local:<puerto-tls>` montando el bundle de la CA interna.
* **Limitación East-West:** Es estrictamente un cifrado **unidireccional (solo autenticación del servidor)**. El cliente valida el FQDN del pod de destino, pero el servidor no valida criptográficamente la identidad del cliente (no hay mTLS mutuo ni autorización L7 por ruta HTTP).
* **Veredicto:** Re-cifrado y autenticación de host robustos sobre el FQDN interno **sin penalizar el clúster con sidecars de Envoy ni consumir vCPU/RAM adicional por réplica**.

#### 3. Nivel 2: `cloud-service-mesh` (Zero-Trust Integral con mTLS y FQDNs East-West)
* **Configuración del Flag:** `serviceMesh.mode: cloud-service-mesh` (o variable de entorno `JENKINS2026_SERVICE_MESH_MODE=cloud-service-mesh`).
* **Mecanismo:** Activa Google Cloud Service Mesh (CSM) bajo el SKU *standalone* (a la carta por cliente de malla). El plano de control gestionado inyecta automáticamente sidecars de `istio-proxy` en los namespaces de aplicación (`istio.io/rev=asm-managed`).
* **Tráfico East-West sobre FQDNs de Servicio:** Todas las llamadas internas de microservicio a microservicio que resuelven FQDNs internos (`gateway` $\rightarrow$ `backend.microservices.svc.cluster.local`) son interceptadas transparentemente por los sidecars de Envoy.
* **Identidad Criptográfica SPIFFE & mTLS:** Se establece **mTLS mutuo estricto** con certificados emitidos por Google Mesh CA portando identidades SPIFFE de carga de trabajo (`spiffe://<project-id>.svc.id.goog/ns/<ns>/sa/<sa>`), gobernado por `PeerAuthentication STRICT`.
* **Autorización Granular L7:** Se aplican reglas de `AuthorizationPolicy` que restringen qué llamadas HTTP (métodos y rutas) se permiten entre FQDNs de microservicios (e.g., solo el ServiceAccount del Gateway puede invocar endpoints de negocio en el Backend).
* **Trampas Operacionales Reales Resueltas en `jenkins-2026`:**
  * *La Trampa de los Clientes No-Mesh en East-West (`curl exit 56`):* Cuando un componente no mesheado (como el balanceador de GKE en el puerto `:8080` o un runner de CI en el namespace `jenkins`) intentaba invocar el FQDN de un pod mesheado en modo `STRICT`, la conexión era abortada por fallo de handshake TLS. `jenkins-2026` resolvió esto aplicando directivas `PeerAuthentication` con `portLevelMtls: PERMISSIVE` exclusivamente en el puerto de entrada/salud, manteniendo el resto del tráfico East-West en `STRICT`.
  * *La Trampa de Cuotas de Cómputo por Sidecar:* Los sidecars de Istio exigían por defecto 2 vCPU de límite, provocando rechazo de pods en actualizaciones continuas por `exceeded quota`. Se solucionó fijando anotaciones declarativas en los pods (`proxyCPULimit: "500m"`, `proxyMemoryLimit: "512Mi"`).
* **Veredicto:** Cobertura Zero-Trust absoluta a nivel de proceso para normativas bancarias o de salud extremas (PCI-DSS 4.0 / SOC 2 Type II), asumiendo el coste operacional de los sidecars pero automatizando el plano de control con Google.

---

### 🔬 Comparativa Profunda: Los 3 Enfoques de `jenkins-2026` vs. El PoC de OpenShift + Traefik

| Dimensión Técnica | Nivel 0: `none` (Default) | Nivel 1: `backend-tls` (Feature Flag) | Nivel 2: `cloud-service-mesh` (Feature Flag) | Enfoque OpenShift + Traefik v3 (`traefik-fqdn-management`) |
| :--- | :--- | :--- | :--- | :--- |
| **Repositorio / Ecosistema** | `nubenetes/jenkins-2026` (GKE) | `nubenetes/jenkins-2026` (GKE) | `nubenetes/jenkins-2026` (GKE) | `traefik-fqdn-management-poc-openshift-aws` (ROSA) |
| **Punto de Entrada Ingress** | GKE Gateway API (`gke-l7`) + IAP | GKE Gateway API (`gke-l7`) + IAP | GKE Gateway API (`gke-l7`) + IAP | AWS NLB L4 + Traefik Proxy v3 (`Gateway` / `IngressRoute`) |
| **Salto Ingress/LB → Pod** | HTTP plano (VPC + WireGuard) | HTTPS re-cifrado y validado (`BackendTLSPolicy`) | mTLS gestionado (puerto edge PERMISSIVE) | HTTPS con SNI y `ServersTransport` / `BackendTLSPolicy` |
| **Tráfico East-West (Pod ↔ Pod)** | Cilium L3/L4 NetworkPolicies | Cilium L3/L4 NetworkPolicies | **mTLS mutuo estricto (SPIFFE)** + AuthZ L7 | **mTLS perimetral interno** vía FQDN canónico (`*.apps.cluster.local`) |
| **Presencia de Sidecars** | **Ninguno (0 sidecars)** | **Ninguno (0 sidecars)** | Sí (`istio-proxy` por pod) | **Ninguno (0 sidecars)** |
| **Overhead de CPU / RAM por Pod** | 0% | 0% (solo handshake TLS en pod) | +100-250 MB RAM y +0.1-0.2 vCPU / pod | 0% adicional en pods de aplicación |
| **Identidad Criptográfica** | N/A (Aislamiento de red) | CA interna de clúster (cert-manager) | SPIFFE ID por ServiceAccount (Mesh CA) | Certificados X.509 de CA interna verificados en Traefik |
| **Complejidad Operacional** | Mínima | Baja (gestión de CA interna) | Media-Alta (malla gestionada pero sidecars) | Baja (controlador Traefik unificado) |

---

### 🛡️ Lecciones de Arquitectura y Buenas Prácticas para Feature Flags en Redes

La experiencia en el repositorio `jenkins-2026` deja tres patrones de diseño esenciales para cualquier equipo de Platform Engineering:

1. **Exclusividad Mutua Estricta (Guardrail Anticolisión):**  
   Las opciones `backend-tls` y `cloud-service-mesh` son **mutuamente excluyentes**. Cuando una malla de servicios está activa, el Mesh CA y el sidecar de Envoy asumen el control del ciclo de vida TLS del pod. Si una política de `BackendTLSPolicy` intentara validar simultáneamente contra la CA de `cert-manager` en el mismo puerto, se produce un conflicto en el handshake que degenera en errores HTTP 502 Bad Gateway. En `jenkins-2026`, los scripts de configuración (`lib/config.sh`) abortan el despliegue con un error explícito si ambos flags están activos, y los workflows de GitHub Actions presentan un selector único (`intra_cluster_tls: [none, backend-tls, cloud-service-mesh]`), haciendo físicamente imposible una colisión.
2. **Sondeo Activo de Capacidades (Capability-Gated Probing):**  
   Ningún componente confía ciegamente en el valor del flag booleano. Antes de reconfigurar los pods para servir TLS o inyectar sidecars, las sondas de plataforma (`j2026_backend_tls_active` y `j2026_service_mesh_active`) verifican que el clúster soporte físicamente la API requerida (que la CRD `BackendTLSPolicy` exista en el clúster o que el webhook de inyección de Istio esté disponible). Si el clúster es antiguo o el plano de control aún está convergiendo, el sistema **degrada de forma segura y consistente a HTTP plano con un warning**, eliminando por completo caídas de servicio durante rollouts progresivos.
3. **La Promesa Cumplida de Gateway API entre Nubes:**  
   Lo más revelador al contrastar ambos repositorios es que, a pesar de que este proyecto utiliza **OpenShift en AWS con Traefik Proxy v3** y `jenkins-2026` utiliza **GKE en Google Cloud con Cloud Service Mesh**, las definiciones de enrutamiento de aplicación (`HTTPRoute`) son **100% interoperables**. Un desarrollador puede aplicar prácticamente el mismo manifiesto de enrutamiento en ambos clústeres, demostrando que Gateway API no es una promesa futura, sino el estándar de facto que unifica la gestión de tráfico cloud-native hoy.

---

## 🔄 Estrategia de Migración con Cero Downtime: Coexistencia Híbrida

Una de las grandes ventajas de **Traefik Proxy v3.0+** es su arquitectura multi-proveedor. Puedes activar simultáneamente en el controlador:

```yaml
--providers.kubernetescrd=true
--providers.kubernetesgateway=true
```

Esto habilita una transición sin riesgos:
1. **Fase 1 (Día 1):** Migrar cargas críticas existentes desde OpenShift Routes hacia la **Solución A (Traefik CRDs)** para ganar estabilidad inmediata, eliminación de caídas por recargas y soporte de Middlewares.
2. **Fase 2 (Día 2):** Adoptar progresivamente la **Solución B (Gateway API)** para nuevos servicios, delegando `HTTPRoutes` en los equipos de desarrollo.
3. **Fase 3 (Convergencia):** Trasladar los `IngressRoute` a `HTTPRoute` de forma transparente sobre el mismo clúster, sin reemplazar controladores ni alterar la configuración de los balanceadores de AWS.

---

## 🛠️ Manifiestos de Producción Listos para Usar

En el repositorio abierto encontrarás los 10 manifiestos listos para desplegar con `oc apply -f`:

```
traefik-fqdn-management-poc-openshift-aws/
├── README.md                                    # Guía técnica exhaustiva
├── COMPARATIVE_MATRIX.md                        # Matriz profunda de 6 dimensiones
├── LINKEDIN_NEWSLETTER_ES.md                    # Este informe en español
├── LINKEDIN_NEWSLETTER_EN.md                    # English newsletter edition
└── manifests/
    ├── common/
    │   ├── 00-namespaces-rbac-scc.yaml         # RBAC, ServiceAccount y OpenShift SCCs
    │   ├── 01-mock-microservices.yaml          # Cargas de prueba (service-a, service-b) y CAs
    │   └── 02-traefik-controller-deployment.yaml # Controlador Traefik v3 + AWS NLB
    ├── solution-a-traefik-crds/
    │   ├── 01-ingressroute-north-south.yaml    # IngressRoute Edge con Route 53
    │   ├── 02-middleware-security.yaml         # Middlewares de HSTS, CORS e IP allowlist
    │   └── 03-ingressroute-east-west.yaml      # Enrutamiento mTLS interno con TLSOption
    └── solution-b-gateway-api/
        ├── 01-gateway-class.yaml               # Definición del GatewayClass
        ├── 02-gateway-aws.yaml                 # Gateway para AWS NLB y listeners
        ├── 03-httproute-north-south.yaml       # HTTPRoute Edge con filtros nativos
        └── 04-httproute-east-west.yaml         # HTTPRoute con BackendTLSPolicy mTLS
```

---

## 💡 Conclusión y Recomendación para Arquitectos

### 🎯 La Decisión por Patrón de Uso:
* **¿Cuándo mantener OpenShift Routes?** Si tus aplicaciones son monolitos o servicios web básicos dentro del comodín `*.apps`, sin exigencias de mTLS entre microservicios ni cabeceras complejas.
* **¿Cuándo elegir la Solución A (Traefik CRDs)?** Si tu equipo ya cuenta con automatizaciones en Ansible/Terraform/ArgoCD para Traefik y prioriza velocidad operativa inmediata sobre portabilidad estricta.
* **¿Cuándo elegir la Solución B (Gateway API)?** Si operas entornos híbridos (OpenShift + EKS/GKE), buscas estandarización oficial de la CNCF a 5-10 años vista y necesitas un modelo de gobernanza limpio entre administradores de plataforma y desarrolladores de aplicaciones.

---

### ⚡ El Factor Decisivo: Misma FQDN en North-South y East-West Sin Hairpinning

Existe un patrón arquitectónico cada vez más extendido en plataformas empresariales modernas donde **Traefik y Gateway API resultan estrictamente obligatorios frente a OpenShift Routes**:  
👉 **El escenario en el que una misma API o servicio debe ser consumido bajo exactamente el mismo FQDN canónico (`api.empresa.com` o `pagos.plataforma.io`) tanto por clientes externos (North-South) como por microservicios internos (East-West), evitando a toda costa el *Traffic Hairpinning*.**

#### ¿Por qué Falla OpenShift Routes ante este Escenario?
En OpenShift tradicional, una `Route` está vinculada estructuralmente a un IngressController perimetral expuesto a AWS. Si un microservicio interno (e.g. un frontend con Server-Side Rendering (SSR), un orquestador de pedidos o un webhook local) realiza una llamada hacia `https://api.empresa.com`, la resolución DNS lo envía al balanceador público externo de AWS (NLB/ALB) y el tráfico vuelve a entrar al clúster por el router perimetral.  
Este fenómeno de **Hairpinning** genera graves penalizaciones de arquitectura:
1. **Penalización Severa de Latencia:** Añade saltos de red innecesarios fuera del tejido SDN del clúster (OVN-Kubernetes) atravesando interfaces públicas de AWS (15–40 ms de degradación añadida).
2. **Costes Inútiles de Transferencia de Datos en AWS:** Se factura tráfico de salida y entrada (egress/ingress data transfer) en balanceadores y NAT Gateways para comunicaciones entre pods vecinos.
3. **Pérdida de la Identidad Zero-Trust:** El tráfico interno sale al perímetro público perdiendo la trazabilidad de IP de origen y la capacidad de autenticar el certificado de cliente interno.
4. **Punto Único de Fallo:** Una degradación en la conectividad externa de AWS o en el gateway de internet puede tumbar llamadas estrictamente internas entre microservicios del mismo clúster.

#### El Mito de "Modificar CoreDNS" en OpenShift: Por Qué Es Inviable para los Equipos de Aplicaciones
Frecuentemente surge la duda: *¿Por qué no simplemente añadimos una entrada en CoreDNS para que `api.empresa.com` resuelva a una IP interna del clúster?*  
En Red Hat OpenShift, **este planteamiento es un antipatrón inviable por diseño**:
* **El OpenShift DNS Operator gobierna CoreDNS:** El ConfigMap `dns-default` en el namespace `openshift-dns` está gestionado activamente por el DNS Operator (`dns.operator.openshift.io`). Cualquier modificación manual a los Corefiles o plugins (e.g. `rewrite` o `hosts`) es revertida automáticamente por el operador en cuestión de segundos.
* **Barrera de Privilegios RBAC (Falta de Autonomía):** La única vía soportada para declarar zonas en OpenShift es parchear el recurso de clúster `dnses.operator.openshift.io/default`. Esto requiere **privilegios de `cluster-admin`**. En organizaciones empresariales con decenas de equipos de desarrollo, los desarrolladores NO tienen ni deben tener permisos de administración de clúster; depender de tickets a soporte de infraestructura para registrar cada FQDN destruye la agilidad de entrega continua.
* **El Riesgo Catastrófico de Split-Brain DNS:** Si se secuestra a nivel de clúster el dominio corporativo público `empresa.com`, cualquier Pod que legítimamente necesite acceder a recursos externos bajo ese dominio (portales corporativos externos, webhooks de terceros, proveedores de identidad OAuth/SAML en la nube) sufrirá caídas inmediatas de resolución.

#### La Solución Elegante: Split-Horizon Ingress con Traefik y Gateway API
En lugar de manipular DNS a nivel de infraestructura, **Split-Horizon Ingress** traslada el desacoplamiento directamente a la **Capa 7 de Ingress en Traefik**:
Traefik permite enlazar el **mismo FQDN corporativo (`api.empresa.com`) a dos horizontes lógicos con políticas totalmente independientes**:
* **Horizonte Exterior (North-South Ingress - EntryPoint `websecure` :8443):** Escucha en el AWS NLB público, aplica filtros WAF perimetrales, CORS para navegadores de internet, rate limiting y termina TLS con certificados públicos corporativos (Let's Encrypt / DigiCert / AWS ACM).
* **Horizonte Interior (East-West Ingress - EntryPoint / Service `ClusterIP`):** Escucha en una dirección IP virtual privada del clúster (solo accesible dentro de la red SDN OVN-Kubernetes).

#### ¿Cómo se Implementa el Enrutamiento Interno sin Tocar CoreDNS? (2 Patrones de Producción)
1. **Patrón Recomendado: IngressRoute Interno + Middleware de Mutación de Cabeceras (`headers`):**
   * El microservicio llamante utiliza el DNS nativo del clúster (`service-b.traefik-crd-poc.svc.cluster.local` o la convención interna elegida como `service-b.apps.cluster.local`), que CoreDNS resuelve de forma inmediata sin requerir ninguna modificación en el operador de DNS.
   * El manifiesto `IngressRoute` interno ([`03-ingressroute-east-west.yaml`](./manifests/solution-a-traefik-crds/03-ingressroute-east-west.yaml)) captura la llamada y aplica el `Middleware` de tipo `headers` ([`02-middleware-security.yaml`](./manifests/solution-a-traefik-crds/02-middleware-security.yaml)): inyecta `Host: api.empresa.com` y `X-Forwarded-Host: api.empresa.com`.
   * El backend recibe la petición con el FQDN canónico esperado para validar tokens JWT y CORS, mientras que el `TLSOption` ejecuta mTLS estricto (`RequireAndVerifyClientCert`) y el middleware `ipAllowList` protege la red. Todo a velocidad de cable local, sin salir a AWS y sin tocar CoreDNS.
2. **Patrón para Aplicaciones Rígidas: Inyección de `spec.hostAliases` a Nivel de Pod (Cero Privilegios):**
   * Si el código fuente o SDK de la aplicación cliente tiene la URL pública `https://api.empresa.com` codificada en duro y no puede alterarse:
   * El equipo de desarrollo añade en su propio manifiesto `Deployment` (sin requerir permisos de `cluster-admin`):
     ```yaml
     spec:
       template:
         spec:
           hostAliases:
             - ip: "172.30.150.10" # ClusterIP del Service interno de Traefik
               hostnames:
                 - "api.empresa.com"
     ```
   * Kubernetes inyecta esta resolución en el `/etc/hosts` local del contenedor en su arranque. Cuando el microservicio invoca `https://api.empresa.com`, el resolver local del Pod lo envía directamente al `ClusterIP` interno de Traefik.
   * Traefik recibe la llamada en su horizonte interno, valida el mTLS con `strict-mtls-option`, aplica los Middlewares y entrega el tráfico al pod backend en la red OVN-Kubernetes.

**Veredicto:** Arquitectura **Split-Horizon Ingress** limpia: el clúster preserva **un único FQDN canónico**, garantizando contratos de API inalterables, **cero hairpinning hacia AWS, latencia en submilisegundos, cero dependencias del operador de DNS de OpenShift y cero consumo de memoria por sidecars**.

---

### 🎬 Sesiones Técnicas en Vídeo en YouTube (@nubenetes)

Si prefieres seguir el despliegue, las pruebas de validación y los análisis arquitectónicos en formato vídeo y audio paso a paso, dispones de una serie técnica completa en [**youtube.com/@nubenetes**](https://youtube.com/@nubenetes):

> 💡 **Nota sobre Idioma y Ajustes de Audio:** Todos los vídeos y shorts cuentan con soporte para **pistas de audio multilingües de YouTube** / doblaje automático (hasta 21 idiomas seleccionables en **Configuración (⚙️) ➔ Pista de audio**). El idioma del audio original coincide con el título: aquellos con título en inglés cuentan con **audio original en inglés 🇺🇸**, mientras que los titulados en español fueron grabados con **audio original en español 🇪🇸**.

1. 🚀 [**OpenShift con FQDN en north-south y east-west: Traefik vs Gateway API**](https://www.youtube.com/watch?v=kIEqhHRf-Ks) *(9m 06s • Audio Original: Español 🇪🇸 • Pistas Multilingües ⚙️)*  
   *Análisis a fondo:* Walkthrough técnico de este repositorio en OpenShift ROSA (AWS). Comparativa de Solución A (`IngressRoute`) vs. Solución B (`HTTPRoute`), integración con AWS NLB y PROXY Protocol v2, mTLS estricto (TLS 1.3) y cumplimiento de SCC `restricted-v2`.
2. 🎯 [**FQDN unificado en OpenShift para north-south y east-west con Traefik y Gateway API**](https://www.youtube.com/watch?v=zUq_CYC7vM8) *(9m 25s • Audio Original: Español 🇪🇸 • Pistas Multilingües ⚙️)*  
   *Análisis a fondo:* Cómo superar el bloqueo estricto del Corefile de CoreDNS en OpenShift con Traefik Proxy v3 y Gateway API. Evaluación práctica de Patrón A (Reenvío con DNS Operator) vs. Patrón B (Split-Horizon Route 53).
3. 🎙️ [**Unified FQDN Routing with Traefik alternatives**](https://www.youtube.com/watch?v=xuDtcUZYeHU) *(8m 14s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*  
   *Análisis a fondo:* Alternativas a Traefik para FQDN unificado: Cilium eBPF en kernel vs. Istio Ambient mode, eliminación del "impuesto de sidecar" (0 MB de RAM por pod) y comparativa de flujos Capa 4 vs. Capa 7.
4. 🎙️ [**Gateway API y FQDNs**](https://www.youtube.com/watch?v=vay32AcPJ9Q) *(8m 44s • Audio Original: Español 🇪🇸 • Pistas Multilingües ⚙️)*  
   *Análisis a fondo:* Evolución del estándar Kubernetes Gateway API v1.1 GA hacia 2026, resolución de FQDNs de doble plano y erradicación del drift de entornos en arquitecturas multicloud.
5. 🎙️ [**Podcast: FQDN Unificado y el Futuro de Ingress/Service Mesh**](https://www.youtube.com/watch?v=pUilWzKDgFQ) *(15m 55s • Audio Original: Español 🇪🇸 • Pistas Multilingües ⚙️ • Formato: Podcast de Audio sin vídeo/slides)*  
   *Análisis a fondo:* Podcast técnico en formato solo audio que analiza el antipatrón de forzar URLs duales (`INTERNAL_URL` vs. `EXTERNAL_URL`) en aplicaciones de IDP, la consolidación de FQDN unificado en OpenShift y AWS con Traefik v3 y Gateway API, superación del bloqueo de CoreDNS, alternativas sidecarless y reflexiones sobre la cultura técnica frente al "teatro de Jira".
6. 🎙️ [**Podcast: Beyond Ingress with Dual Plane FQDN**](https://www.youtube.com/watch?v=ID0YEJzv_4E) *(59m 29s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️ • Formato: Podcast de Audio sin vídeo/slides)*  
   *Análisis a fondo:* Podcast técnico de 1 hora en formato solo audio que profundiza en la arquitectura Dual Plane FQDN, eliminación de penalizaciones por recarga de HAProxy, enrutamiento transparente de microservicios sin peaje de memoria de sidecars (0 MB de RAM overhead), DNS Split-Horizon en AWS y defensa del rigor técnico en ingeniería de plataformas.

#### ⚡ YouTube Shorts Relacionados (Píldoras Arquitectónicas de 60–90 Segundos)

- ⚡ [**How to Route East West unified FQDNs on OpenShift with Traefik or Gateway API**](https://www.youtube.com/shorts/_YufQ7kv2xM) *(1m 23s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*: Enrutamiento Split-Horizon en Capa 7 en OpenShift 4.x evitando el hairpinning hacia el NLB público de AWS.
- ⚡ [**Cómo Enrutar Dominios Internos con Traefik con FQDN unificado**](https://www.youtube.com/shorts/ZNo0BCIXlbA) *(1m 34s • Audio Original: Español 🇪🇸 • Pistas Multilingües ⚙️)*: Resolución canónica para microservicios sin costes de sidecar ni alteraciones en CoreDNS.
- ⚡ [**The Ghost in the Server: East-West & Split-Brain DNS on Red Hat OpenShift 4.x**](https://www.youtube.com/shorts/LX_SLw5ovVo) *(0m 58s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*: El problema del hairpinning oculto que añade 15–40ms de latencia y costes de egress en AWS.
- ⚡ [**How Split Brain DNS Keeps Traffic Hidden**](https://www.youtube.com/shorts/moT_HjQsuF4) *(1m 09s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*: Inspección perimetral WAF/NLB frente a resolución directa por ClusterIP interno.
- ⚡ [**Routing Internal URLs With Service Mesh**](https://www.youtube.com/shorts/go_sCgyASe4) *(1m 17s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*: Intercepción transparente en Capa 7 con Envoy/Cilium y reescritura dinámica de rutas.
- ⚡ [**Traefik CRDs vs Gateway API on OpenShift**](https://www.youtube.com/shorts/ZykBWmE9Gd8) *(1m 16s • Audio Original: Inglés 🇺🇸 • Pistas Multilingües ⚙️)*: Duelo rápido entre Traefik IngressRoute y Kubernetes Gateway API HTTPRoute en OpenShift ROSA.

---

### 🔗 Explora el Repositorio y Prueba el Despliegue

Te invito a clonar el repositorio, revisar los diagramas de arquitectura en Mermaid y probar los scripts de despliegue en tu propio clúster:

👉 **[github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws](https://github.com/nubenetes/traefik-fqdn-management-poc-openshift-aws)**

¿Qué estrategia de Ingress estás utilizando actualmente en tus clústeres de OpenShift? ¿Has comenzado la adopción de Kubernetes Gateway API o sigues confiando en soluciones basadas en CRDs? ¡Déjame tu opinión en los comentarios! 👇

---
*#Kubernetes #OpenShift #RedHat #AWS #Traefik #GatewayAPI #PlatformEngineering #DevOps #CloudNative #CNCF #ZeroTrust #CyberSecurity*
