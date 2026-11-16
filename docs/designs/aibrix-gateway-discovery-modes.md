# AIBrix Gateway Discovery Modes

| Field | Value |
| --- | --- |
| **Document ID** | STF-DES-0021 |
| **Type** | Design spec |
| **Author** | Andrea Abellonio |
| **Created** | 2026-10-29 |
| **Status** | Active |
| **Classification** | Public |
| **Canonical URL** | https://github.com/abe-servescale/aibrix/blob/main/docs/designs/aibrix-gateway-discovery-modes.md |

*Copies of this document obtained from any URL other than the canonical one above may be outdated or unofficial.*

> **Confidentiality notice** — This document is the property of servescale.ai. Its contents may not be disclosed, copied, or distributed without prior written authorization from servescale.ai, except as permitted by the classification level above.

## Changelog

| Date | Author | Description |
| --- | --- | --- |
| 2026-11-17 | Andrea Abellonio | Move the design to the AIBrix fork; preserve its document ID and technical content. |
| 2026-11-10 | ssai-mdb-agent1 | Approve; clarify scope/opt-in, placement, routing/RBAC, route ownership/content; deduplicate; rename requests/refine dependencies; limit grants to Envoy Gateway; add sidecar inference acceptance. |
| 2026-11-09 | ssai-mdb-agent1 | Remove product text; define confinement purpose, modes, principles and phases; adopt AIBRIX_INFERENCE_NAMESPACE; add consolidation appendix; specify discovery, RBAC, reporting and scope changes. |
| 2026-11-08 | ssai-mdb-agent1 | Relocate into supporting-docs/aibrix; preserve substantive design content. |
| 2026-11-02 | Andrea Abellonio | Use one optional namespace parameter, separate model-list work, and replace staged Helm retention with migration during a service interruption; retain deployment confinement and exclude allowlists. |
| 2026-10-29 | Andrea Abellonio | Initial draft covering cluster and namespace discovery, cache ownership, model listing, scoped permissions, and guarded migration. |

## Purpose and scope

This design enables the AIBrix gateway to operate when an application installation's access to inference workloads is confined to one namespace with RBAC. Its ServiceAccount may read those workloads in that namespace while cluster-wide workload LIST/WATCH requests are forbidden. This use case applies on a dedicated cluster as well as a shared one.

The gateway must discover eligible workloads within its allowed scope and build its routing state without requesting workloads elsewhere. The optional `AIBRIX_INFERENCE_NAMESPACE` environment variable selects that workload scope. Omitted or empty input preserves the existing cluster-wide behavior. Merely placing Pods in a namespace, without enforcing those permission restrictions, does not exercise this use case.

### Discovery modes

The gateway has two modes for Kubernetes workload discovery, derived from `AIBRIX_INFERENCE_NAMESPACE`:

| Discovery mode | Selected when | Discovery scope |
| --- | --- | --- |
| **Cluster-wide discovery mode (default)** | AIBRIX_INFERENCE_NAMESPACE is unset or empty. | Requests workloads across all namespaces. |
| **Namespace-restricted discovery mode** | AIBRIX_INFERENCE_NAMESPACE contains one valid namespace name. | Requests inference workloads only in that namespace. |

Discovery mode describes what the gateway requests, while placement describes where the gateway plugin Pod and inference workloads run. Namespace-restricted deployments support two placements:

- **Colocated placement:** The gateway plugin Pod and inference workloads share one namespace.
- **Split placement:** The gateway plugin Pod is in one namespace, and inference workloads are in another.

### Namespace permissions, application confinement, and multitenancy

These are three different concerns:

| Concern | Evidence and relevance |
| --- | --- |
| Using namespaces with RBAC | [Kubernetes RBAC guidance](https://kubernetes.io/docs/concepts/security/rbac-good-practices/#least-privilege) recommends namespace-scoped permissions through RoleBindings wherever possible. This is general least-privilege guidance. |
| Confining an application installation with namespace-scoped RBAC | [Kubernetes ServiceAccount guidance](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#service-account-permissions) recommends application-specific accounts and shows a namespace-scoped RoleBinding. The [GitLab Runner Helm chart](https://docs.gitlab.com/runner/install/kubernetes_helm_chart_configuration/#limit-the-permissions-the-chart-grants) illustrates this best practice by using namespace-scoped RBAC for the application. |
| Designing AIBrix multitenancy | Outside this design's scope. Supporting namespace-scoped API permissions does not define tenant identities or routing, guarantee independence between AIBrix installations, or establish network, storage, or node isolation. |

The namespace input tells the gateway which API requests to make; [Kubernetes RBAC](#authorization) enforces what its ServiceAccount may access. The gateway must choose the intended request scope explicitly, rather than infer it by attempting cluster-wide requests and reacting to permission errors. If scope were inferred from permissions, an old cluster-wide binding could silently select cluster-wide discovery. An explicit value such as `AIBRIX_INFERENCE_NAMESPACE=inference` keeps discovery in `inference`, while missing required access causes startup to fail. The [gateway-only first contribution](#component-sequence-and-limits) does not yet confine every AIBrix component. Any required cluster-scoped operations need separately justified, narrow RBAC permissions.

## Separate gateway requests

Four gateway or deployment defects and one sidecar enhancement will be reported separately. They do not require AIBRIX_INFERENCE_NAMESPACE to be set. The dependency column distinguishes namespace prerequisites from related work and the requirement for sidecar public model-list qualification.

| Type | Separate request | Required behavior | Dependency |
| --- | --- | --- | --- |
| Bug | HTTPRoute lookups hard-code aibrix-system instead of using existing Pod metadata. | Use the gateway plugin Pod's namespace from the POD_NAMESPACE environment variable for HTTPRoute lookups; use aibrix-system when POD_NAMESPACE is unset. | Namespace prerequisite |
| Bug | The gateway model-list endpoint rejects the trailing-slash form already accepted by the public metadata endpoint. | Accept both /v1/models and /v1/models/, including query strings, through the same gateway handler without redirects. This preserves public URL compatibility when a deployment routes model-list requests to the gateway. | Namespace prerequisite; also required by the sidecar enhancement in both discovery modes. |
| Bug | Kustomize leaves example names embedded in Envoy configuration when the installation namespace or Gateway name changes. | Derive listener, route-configuration, and extension-filter references from the rendered names. Changing resource metadata alone does not update these embedded strings, leaving patches aimed at the wrong configuration. | Namespace prerequisite |
| Bug | Packaged gateway RBAC includes unused write permissions. | Remove unused write verbs from the existing cluster-wide Helm and Kustomize gateway roles, retaining required reads and Pod patch for ModelClaim wake. Default discovery retains its cluster-wide workload access. | Related work; [restricted Roles](#authorization) have their own required verbs. |
| Enhancement | Sidecar Envoy lacks a public model-list route independent of inference routing. | Route both model-list URL forms to the local gateway HTTP listener at 127.0.0.1:8080 before inference routes, bypassing external processing and requiring no inference model or routing-strategy headers. | Sidecar public model-list qualification only. |

The [v0.8.0 ModelRouter controller](https://github.com/vllm-project/aibrix/blob/453ab36db68ac859f86ac94e32d0e02cec4c1505/pkg/controller/modelrouter/modelrouter_controller.go) creates HTTPRoutes in `aibrix-system` and attaches them to Gateway `aibrix-system/aibrix-eg`. The gateway lookup fix does not change that placement. The gateway-only contribution uses the [per-model route setup](#coverage-and-placement); ModelRouter placement support elsewhere requires separate assessment.

Accepting both gateway URL forms does not choose the public route's backend; see [Model identity and public listing](#model-identity-and-public-listing).

## Contribution principles

- **Explicit opt-in:** Changes introduced by namespace support require a nonempty `AIBRIX_INFERENCE_NAMESPACE`. With omitted or empty input, namespace support changes nothing in discovery, public routing, or RBAC packaging. Pod placement alone does not activate namespace-restricted discovery.
- **Minimum necessary changes:** Make only changes required for the supported namespace-restricted workflow. Retain existing behavior wherever it works; exclude unrelated cleanup, generic hardening, and features.
- **Consolidate over time:** Propose only high-priority opportunities to share behavior between namespace-restricted mode and the default cluster-wide mode in the [appendix](#appendix-consolidation), with a concrete maintenance benefit and compatibility effect.

## Coverage and placement

In namespace-restricted mode, the gateway contribution covers workload discovery and cache, matching RBAC manifests and installation configuration, startup checks, and [runtime mode inspection](#runtime-visibility). It also includes [Envoy Gateway public model-list routing](#model-identity-and-public-listing) backed by the routing cache and [scoped HTTPRoute read permissions](#authorization) in the gateway plugin Pod's namespace (`POD_NAMESPACE`). The lookup behavior itself comes from the prerequisite fix. Discovery includes Pods, enabled ModelAdapters, and optional ModelClaims.

Several namespace-related concepts have different purposes:

| Concept | Meaning |
| --- | --- |
| Gateway plugin Pod namespace | Where the plugin runs; exposed through the Downward API as `POD_NAMESPACE`. |
| ServiceAccount identity | The account's name and namespace, used as the binding subject. Kubernetes uses the RBAC bindings to determine the account's permitted access. |
| Gateway/HTTPRoute placement | Where the routing configuration objects reside. This is distinct from workload placement and from the actual Envoy proxy Pod and Service placement. |
| Inference workload namespace | The namespace configured through `AIBRIX_INFERENCE_NAMESPACE` for inference workload discovery. It may differ from the gateway plugin Pod's namespace. |

For ordinary HTTPRoute-based routing in the gateway-only contribution, the administrator or their model-deployment tooling supplies each model's HTTPRoute in the gateway plugin Pod's namespace, attached to the packaged Gateway. Routes follow the existing `ModelRouterName` naming rule in [aibrix/pkg/utils/model_router.go](https://github.com/vllm-project/aibrix/blob/453ab36db68ac859f86ac94e32d0e02cec4c1505/pkg/utils/model_router.go). Each route matches the exact model name in the `model` header on inference paths and forwards requests to the model's Service and port, following the existing [ModelRouter route builder](https://github.com/vllm-project/aibrix/blob/453ab36db68ac859f86ac94e32d0e02cec4c1505/pkg/controller/modelrouter/modelrouter_controller.go#L346-L413). For Envoy Gateway deployments, AIBrix's installation manifests supply the shared Gateway and shared routes, plus the split-placement ReferenceGrant described under [Helm and Kustomize](#helm-and-kustomize). This design adds no model catalog or per-model route generator to the installation packaging.

### Component sequence and limits

1. **Gateway-only first contribution:** Start with the gateway and its required supporting configuration. Administrators can supply workloads and routing objects directly, without the AIBrix workload controllers. This establishes gateway behavior, not namespace-restricted operation of every supporting component.
2. **Workload-controller namespace support:** Extend StormService, RoleSet, and PodSet in a companion contribution so those workload creation and management paths can operate with namespace-scoped permissions.
3. **Other components:** Consider further controllers and caches only with their own access assessment and supported feature combinations. Their implementation is not a commitment of the initial gateway contribution.

Controller-manager discovery remains cluster-wide until separately covered. Neither contribution claims namespace-confined operation of every AIBrix component. Selecting the gateway's namespace does not disable other deployments or controllers; components that remain enabled require their own permission assessment. Namespace allowlists, installation labels, model registries, Redis partitioning, and complete isolation between installations are outside scope.

## Configuration

### Runtime API

Read `AIBRIX_INFERENCE_NAMESPACE` once at startup as the sole selector of the [discovery mode](#discovery-modes) and inference workload namespace. There is no namespace CLI argument or separate mode setting. The inference workload namespace may differ from the namespace of the gateway plugin Pod and ServiceAccount. `POD_NAMESPACE` remains hosting-Pod metadata and does not activate namespace-restricted mode.

Validate nonempty values without trimming, normalization, or substitution, before Redis initialization or discovery. Whitespace-only strings, padded names, and comma-separated namespace lists are invalid, not empty. Invalid input or denied required workload access must fail startup without falling back to cluster-wide discovery. In namespace-restricted mode, log the resolved mode and namespace and include the namespace in access errors. Preserve existing cluster-wide logging. Standalone file discovery bypasses Kubernetes namespace validation and makes no Kubernetes requests.

Installation tooling or the administrator creates namespaces; the gateway does not. The gateway does not read Namespace objects to validate existence. A successful workload LIST can be empty without establishing that the Namespace object exists, so startup does not promise to distinguish a missing namespace from an existing namespace with no eligible workloads. The latter is valid. This removes the need for a cluster-scoped Namespace reader.

Retain the existing source switches: `--watch-model-adapters` and `--watch-model-claims` default to `true` in the CLI. Disabled sources issue no requests and have no informer or synchronization requirement; disabling claims also excludes their runtime bindings from model mappings.

### Runtime visibility

Expose the resolved discovery scope as one information gauge on the gateway's existing HTTP `/metrics` endpoint, registered only in namespace-restricted mode:

```text
# HELP aibrix_gateway_discovery_scope_info Resolved Kubernetes workload discovery scope.
# TYPE aibrix_gateway_discovery_scope_info gauge
aibrix_gateway_discovery_scope_info{discovery_mode="namespace-restricted",inference_namespace="inference"} 1
```

Populate the gauge once from the same resolved scope used by discovery/cache before successful startup completes. It requires no inference traffic, new listener, separate API, diagnostic command, feature flag, or Kubernetes request. The value is always `1`; its labels report configuration independently of health or authorization. Do not reread the environment when serving metrics. Omitted or empty namespace input and standalone discovery register no new metric and add no scope-specific startup log. Preserve the existing optional early HTTP-listener behavior.

An administrator can inspect the report directly without installing Prometheus. For example, with gateway HTTP port 8080 already forwarded to local port 18080:

```bash
curl http://127.0.0.1:18080/metrics
```

The reported namespace is the inference workload namespace, even when the HTTP listener's Pod lives elsewhere. This follows the [Prometheus information-metric convention](https://prometheus.io/docs/practices/naming/#metric-names). Broader reporting remains a separate [consolidation proposal](#appendix-consolidation).

### Helm and Kustomize

The Helm chart accepts the string `gatewayPlugin.inferenceNamespace`, defaulting to empty. Empty renders the existing cluster-wide RBAC resources and omits `AIBRIX_INFERENCE_NAMESPACE` from the gateway environment. A nonempty value must be a valid namespace name; the chart quotes it in the rendered environment and supplies the restricted RBAC resources described in [Authorization](#authorization). Reject non-string values. Do not substitute `.Release.Namespace` for an empty value or add a separate chart mode setting.

Default Kustomize manifests also omit `AIBRIX_INFERENCE_NAMESPACE`. A restricted overlay renders that environment setting and the [workload Role/RoleBinding](#authorization) for the same concrete namespace. Explicitly colocated overlays may populate `AIBRIX_INFERENCE_NAMESPACE` through the Downward API from `metadata.namespace`. Split-placement overlays supply the workload namespace explicitly. Rendering the workload RoleBinding there must preserve the gateway ServiceAccount's rendered name and namespace in its subject, as specified in [Authorization](#authorization).

Keep the existing Downward API injection of the gateway plugin's actual Pod namespace as `POD_NAMESPACE` in both packagings. This namespace is used for [HTTPRoute lookups](#separate-gateway-requests) and [restricted HTTPRoute read permissions](#authorization).

For namespace-restricted split placement with Envoy Gateway, the Helm and Kustomize installation manifests include a [ReferenceGrant](https://gateway-api.sigs.k8s.io/reference/api-types/referencegrant/) in the inference Service's namespace, permitting HTTPRoutes from the gateway namespace to reference those Services. Gateway API requires this for cross-namespace backend references, so the grant is applied with the other routing resources during normal installation. Colocated routes and Services need no grant for that reference. The gateway does not read this resource or gain additional RBAC permissions. Helm sidecar deployments render no ReferenceGrant because they use no Gateway API routes; namespace selection adds no Gateway API CRD requirement. Kustomize has no sidecar variant.

Apply the restricted workload configuration to both Envoy Gateway and Helm sidecar deployments, preserve common RBAC labels, and leave default quickstart workloads in their documented namespaces. Deployment tests record actual Envoy proxy Pod and Service placement separately from the Gateway object's namespace; sidecar Envoy runs in the gateway Pod.

## Provider and cache

### Scope resolution

Resolve the namespace once and give the gateway's selected Kubernetes discovery provider that fixed scope. The cache consumes that provider, and runtime reporting uses the same resolved value. Avoid a second cache-level namespace setting and rules for reconciling two namespace inputs. Preserve existing provider-construction defaults and source compatibility for cluster-wide callers; add namespace selection without changing their behavior.

### Cache ownership

Keep the existing one-cache-per-process initialization. The gateway initializes it at startup; changing scope requires a new gateway process. This contribution adds no general initialization-ownership comparison framework, checked cache-initialization API, cache reset protocol, or controller-initialization guard. Controller-manager discovery remains separate; its namespace support follows the [component sequence](#component-sequence-and-limits). Standalone discovery uses its file provider.

### Informers and startup

Use existing client-go shared informer factories, typed clients, and Kubernetes namespace validation. Select the resolved namespace through the informer factories' namespace options. No replacement informer framework or dependency upgrade is required.

All Pod, enabled adapter, optional claim, and existing verification requests use the resolved scope, including every page of a verification LIST. Namespaced API requests return objects only from the requested namespace, so event delivery, tombstone handling, and adapter replay retain their existing behavior without an extra event filter. Existing namespace/name Pod identity and wake behavior are unchanged.

Preserve upstream informer synchronization. Namespace selection does not add a mandatory wait for every event handler to finish processing its initial list, in either mode. Pods and enabled adapters remain required. Claims remain optional: a missing CRD or denied claim LIST may skip claim watching, and claim synchronization does not block Pod-based serving.

Before starting restricted informers, check required LIST and WATCH access. This makes incomplete namespace grants a startup error: upstream cache synchronization requires LIST success but can complete even when WATCH is denied. For Pods and enabled ModelAdapters, issue a one-item LIST and open then immediately stop a WATCH in the selected namespace. Each request uses a 30-second context timeout, matching the existing optional claim LIST probe. A failed required probe prevents startup and reports the selected namespace. The optional claim probe retains its existing timeout and skip/retry behavior, with only its request namespace changed.

Informer retries and lifetime behavior remain unchanged in both modes. In the existing Helm/Kustomize deployments, Kubernetes restarts a gateway that fails its liveness probe during startup; namespace selection adds no separate synchronization deadline. Cluster-wide mode gains no new access probes.

Ongoing access errors are reported and existing retries stay within the selected namespace. Losing access loses the gateway's sole configured workload scope; this design promises neither continued service from stale cache data nor a new inference-health gate. Existing optional freshness verification continues to invalidate its proof on required watch errors without widening discovery.

## Model identity and public listing

Pods use the nonempty `model.aibrix.ai/name` label before the corresponding annotation. Preserve exact names such as `vendor/Exact-Model.v1`. ModelAdapters use their object name; enabled claim runtime bindings may add mappings. Existing names and pooling behavior remain within the configured workload scope.

A gateway's public model list should follow its configured workload scope. The metadata service currently discovers models across namespaces independently of the gateway. Its results can therefore include models outside a restricted gateway's scope. The gateway already has a model-list endpoint backed by its discovery cache; using that endpoint keeps listing aligned with the selected workloads and avoids maintaining a second discovery configuration.

For Envoy Gateway deployments in namespace-restricted mode, the Helm and Kustomize HTTPRoute manifests configure Envoy to send both `/v1/models` and `/v1/models/`, including query strings, to the gateway plugin's HTTP Service, normally port 8080. Omitted or empty input retains the metadata-service public model-list route.

Once the [sidecar enhancement](#separate-gateway-requests) is available, sidecar deployments route public model-list requests to the local gateway handler in both discovery modes. Namespace selection controls the workloads in the gateway cache; it does not enable the sidecar route.

Preserve the existing response format and list-mode semantics. Normal listing can return an empty cached set; it supplies no new freshness guarantee. If existing optional verification is enabled, its existing unavailable response remains HTTP 503. `/v1/files` and `/v1/batches` retain their current destination. Routing model lists to the gateway does not replace the metadata service's other functions or disable its deployment. Its own Kubernetes discovery is unchanged and requires separate permission assessment if used under restricted RBAC; these contributions do not establish namespace-confined operation of that service.

New listing modes, paginated freshness comparisons, proof-age/retry policies, and early HTTP availability gates remain separate work. Qualify public listing and inference routing together under the [acceptance criteria](#acceptance-criteria); discovery-only checks do not establish that integration.

## Authorization

Kubernetes authorizes gateway requests using the ServiceAccount's effective RBAC permissions. AIBrix's Helm templates and Kustomize overlays supply the packaged RBAC resources listed below; administrators apply those manifests and manage other bindings. Changing `AIBRIX_INFERENCE_NAMESPACE` alone changes the gateway's request scope, not its RBAC permissions.

Cluster-wide packaging uses a ClusterRole/ClusterRoleBinding for namespace-spanning workload access. Restricted packaging uses the following namespaced RBAC resources:

| Resource | Required verbs | Placement |
| --- | --- | --- |
| Pods | `get,list,watch,patch` | Role/RoleBinding in the inference workload namespace |
| ModelAdapters and ModelClaims | `get,list,watch` | Same workload Role/RoleBinding |
| HTTPRoutes | `get` | Separate Role/RoleBinding in the gateway plugin Pod's namespace, using `POD_NAMESPACE`, or `aibrix-system` when it is unset |
| Namespaces | None | No Namespace-reader resource |

The rendered binding subjects identify the actual gateway ServiceAccount name and namespace, including across namespaces. Workload Roles/RoleBindings use the same namespace configured for discovery. Source switches do not require another packaged-RBAC configuration mechanism.

Discovery reads resources. The gateway's existing ModelClaim wake path patches a Pod using its cached namespace and name; RBAC cannot restrict that permission to the wake annotation. The gateway needs no Pod create, update, or delete permission.

Prometheus access is unchanged in both discovery modes. Incoming gateway metrics scraping uses the existing HTTP endpoint; deployment configuration determines scrape targets and ServiceMonitor placement/selectors.

Outgoing Prometheus queries use the existing `PROMETHEUS_BASIC_AUTH_SECRET_NAME` and `PROMETHEUS_BASIC_AUTH_SECRET_NAMESPACE` settings, or existing environment credentials when no Secret is selected. When a Secret is selected, administrators supply it and the existing named-Secret `get` permission in its namespace; this design adds no Secret permission to the packaged gateway RBAC. The Secret namespace defaults to `aibrix-system`, so a Secret elsewhere requires setting `PROMETHEUS_BASIC_AUTH_SECRET_NAMESPACE`. The workload selector does not override that location. No Prometheus source change or additional namespace input is required.

Other bindings can allow broader access even when the gateway requests only its selected namespace. Administrators own removal of obsolete installation-owned workload bindings outside that namespace, while preserving legitimate routing/Secret access and other installations' resources. Deployment tests verify effective permissions independently of the gateway; the gateway receives no administrative permission to inspect or clean up RBAC.

## Changing discovery scope

A scope change requires the administrator to restart the gateway with a fresh cache, so routing state from the previous scope is not reused. Restarting for scope changes keeps deployment requirements small. Scope changes accept a service interruption: stop the old instance before starting its replacement. To return to cluster-wide discovery, unset or clear `AIBRIX_INFERENCE_NAMESPACE` and restart the gateway. Failure handling never widens scope automatically.

Administrators use ordinary Helm/Kustomize deployment procedures and manage effective permissions as described in [Authorization](#authorization). This contribution adds no migration controller. Same-scope upgrades retain the existing rollout procedure.

## Design choices

Use `AIBRIX_INFERENCE_NAMESPACE` to match AIBrix's existing `AIBRIX_` configuration naming pattern. The [v0.8.0 KV-cache watcher](https://github.com/vllm-project/aibrix/blob/453ab36db68ac859f86ac94e32d0e02cec4c1505/cmd/kvcache-watcher/main.go#L356-L361) already uses `AIBRIX_KVCACHE_WATCH_NAMESPACE` for namespace-scoped Pod discovery.

## Acceptance criteria

| Area | Required coverage |
| --- | --- |
| Prerequisites | Verify the changes marked as namespace prerequisites in [Separate gateway requests](#separate-gateway-requests) before namespace qualification. Qualify sidecar public model listing against its listed dependencies; sidecar listing does not block namespace discovery/inference or Envoy Gateway public listing. |
| Input and rendering | Unset/empty AIBRIX_INFERENCE_NAMESPACE preserves cluster-wide mode; one valid nonempty value selects restricted mode. Invalid names, whitespace, lists, and non-string Helm values fail validation without substitution or fallback. Verify environment-only selection, unchanged POD_NAMESPACE meaning, Helm/Kustomize environment and RBAC resources, source switches, sidecars, ServiceAccount subjects, and common labels. Verify Helm sidecar fixtures install without Gateway API CRDs and render no ReferenceGrant, Gateway, GatewayClass, or HTTPRoute in either placement. |
| Scoped discovery and identity | Required and enabled sources request only the selected namespace, including verification pages. Preserve existing event delivery, tombstone handling, replay, exact model names, pooling, namespace/name Pod identity, and scoped claim wake. Disabled sources issue no requests. |
| Startup and lifetime | Required LIST or WATCH denial prevents startup; successful LIST alone cannot pass a WATCH-denied case. Exercise bounded pre-start LIST/WATCH probes and their request timeouts, existing synchronization/replay and lifetimes, optional/disabled sources, and an existing namespace with no workloads. Successful workload reads make no Namespace-existence guarantee; verify no Namespace GET is issued or needed. |
| Working inference and public listing | Under enforced RBAC, a non-default colocated Envoy Gateway deployment with [per-model HTTPRoutes](#coverage-and-placement) supplied in the gateway namespace handles ordinary HTTPRoute routing and a strategy request that reaches the selected inference Pod. For both Helm and Kustomize, also exercise split placement: POD_NAMESPACE equals the gateway Pod's metadata.namespace, the HTTPRoute exists in that namespace, HTTPRoute GETs target it, and an ordinary routed request reaches a Pod in a different inference namespace. The fixture applies the ReferenceGrant rendered by each packaging in the inference namespace, permitting HTTPRoutes from the gateway namespace to reference its Services, together with rendered RBAC for HTTPRoute reads in the gateway namespace and workload access in the inference namespace. Verify that colocated routes and Services need no ReferenceGrant for that reference and split placement needs no manually created ReferenceGrant. In namespace-restricted mode with enforced RBAC, a strategy request through Helm sidecar Envoy reaches the selected inference Pod in both colocated and split placement. Both model-list URL forms and query strings reach the same scoped cache through Envoy Gateway without extra inference headers or redirects. Once the sidecar enhancement is available, apply the same public-listing checks through sidecar Envoy. Preserve response/list-mode behavior, empty lists, and existing optional verification errors. |
| Runtime mode inspection | After startup, directly scrape /metrics and verify the exact information gauge and labels against the resolved restricted scope, without inference traffic or Prometheus. Split placement reports the workload namespace. Health/access changes do not change the report. Existing normal and optional early HTTP paths use the same registry. |
| Default compatibility | With input omitted/empty, retain cluster-wide discovery, synchronization, retries, RBAC scope, and the public model-list routing specified for each deployment type. Export no new scope metric or scope-specific log. Preserve default quickstart workloads and existing claim wake for Pods in default. Standalone discovery remains independent of Kubernetes. |
| Permissions and supporting packages | Record which supporting components the deployment fixture enables or omits. Apply rendered restricted manifests and use a real Kubernetes authorizer to prove required reads/Pod wake and denied foreign/all-namespaces workload access, Namespace reads/enumeration, and unrelated Pod writes. Qualify colocated and split Helm/Kustomize bindings, embedded Envoy names, and actual proxy Pod/Service placement. With administrator-supplied Prometheus Secret credentials and read permission, verify their configured location and successful queries; incoming scraping and ServiceMonitor selection remain separate checks. |
| Scope changes | Exercise administrator-controlled scope changes and failed changes with a fresh gateway cache. No old-scope gateway remains running before its replacement starts. Failure handling never widens scope automatically. Unset/empty input on restart returns to cluster-wide mode. Same-scope upgrades keep existing behavior. Verify the separately applied RBAC using the Permissions and supporting packages checks. |

These are requirements for later implementation qualification. Run scoped discovery/cache Go and race checks when those paths change, and qualify rendering and real API/RBAC behavior. Build both documentation languages when RST files change. This document records no runtime qualification results.

<a id="appendix-consolidation"></a>
## Appendix — High-priority consolidation proposals

These proposals address correctness and maintenance costs directly related to namespace support. They require separate consideration and do not change default behavior as part of this contribution.

- **One model-list source for both discovery modes:** After scoped public listing is established, consider serving model lists from the gateway's routing index in default-mode Envoy Gateway deployments too. This could remove separate model-discovery logic and reduce disagreement between advertised models and routing. It would change the current metadata-service route, so model membership, response compatibility, and failure behavior need separate assessment.
- **One scope-reporting contract for both modes:** Consider extending runtime mode/scope reporting and scope-specific startup logging to cluster-wide mode. A shared diagnostic contract would let administrators use the same checks in either mode and reduce mode-specific reporting code. This would add diagnostics to default installations, so interface exposure and logging compatibility need separate assessment.
