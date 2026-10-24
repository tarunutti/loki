# Grafana Loki on OpenShift 4.18 --- GitOps Reference

> **Purpose:** A learning and implementation reference for deploying
> Grafana Loki on OpenShift 4.18 using Argo CD, S3-compatible object
> storage, Prometheus monitoring, and an OpenShift Route.
>
> **Status:** Reference scaffold, not a certified or pre-validated
> production distribution. Validate every manifest against the exact
> Loki image version and your cluster policies before deploying it to
> production.

## Contents

-   [1. What is Loki?](#1-what-is-loki)
-   [2. How Loki works](#2-how-loki-works)
-   [3. Architecture in this
    repository](#3-architecture-in-this-repository)
-   [4. Loki deployment modes](#4-loki-deployment-modes)
-   [5. Repository layout](#5-repository-layout)
-   [6. Storage and retention](#6-storage-and-retention)
-   [7. Log ingestion and querying](#7-log-ingestion-and-querying)
-   [8. OpenShift integration](#8-openshift-integration)
-   [9. High availability and scaling](#9-high-availability-and-scaling)
-   [10. Security considerations](#10-security-considerations)
-   [11. GitOps with Argo CD](#11-gitops-with-argo-cd)
-   [12. Deploy and validate](#12-deploy-and-validate)
-   [13. Troubleshooting](#13-troubleshooting)
-   [14. Production readiness
    checklist](#14-production-readiness-checklist)
-   [15. Official documentation](#15-official-documentation)

## 1. What is Loki?

Grafana Loki is a horizontally scalable log aggregation system. It
receives logs from agents or applications, associates each log stream
with labels, stores compressed log data in chunks, and stores an index
that helps locate the relevant chunks when a query is run.

Loki is often compared with Prometheus because both use labels to
identify streams of data. Prometheus stores time-series metrics; Loki
stores log lines. Grafana can query and display both, but they are
different systems with different storage and query models.

### Why use Loki?

-   **Label-based indexing:** Loki indexes stream metadata rather than
    indexing every word in every log line. This can reduce index size
    and storage cost.
-   **Object storage:** Log chunks and index data can be stored in
    S3-compatible object storage.
-   **LogQL:** Loki provides a query language for selecting streams and
    filtering or parsing log lines.
-   **Grafana integration:** Logs can be explored alongside metrics and
    traces.
-   **Horizontal scaling:** Loki can run as a single process for small
    deployments or as separate distributed components for larger
    workloads.

### Loki is not a log collector

Loki stores and queries logs; a separate agent or pipeline must collect
logs and send them to Loki. Examples include Grafana Alloy, Fluent Bit,
Vector, or an OpenShift-supported log collection pipeline. Choose one
compatible with your environment and Loki version.

## 2. How Loki works

### Write path

1.  A collector reads application or infrastructure logs.
2.  The collector batches logs and sends them to Loki's write endpoint
    using the supported push API.
3.  The write-side components validate requests and route each stream to
    ingesters.
4.  Ingester replicas process log entries and build compressed chunks.
5.  Data is persisted according to Loki's write and storage
    configuration.
6.  Completed chunks and index data are stored in object storage.

The exact component boundaries depend on the selected deployment mode.
In distributed mode, the distributor and ingester are separate targets.
In Simple Scalable Deployment (SSD), those responsibilities are grouped
behind the `write` target.

### Read path

1.  Grafana or another client sends a LogQL query to the query endpoint.
2.  The gateway routes the request to the read-side services.
3.  Query frontend/scheduler components may split and schedule large
    queries.
4.  Queriers retrieve recent data from ingesters and older data from
    object storage/indexes.
5.  Loki combines the query results and returns matching log entries.

The exact read-path components vary by deployment mode and Loki release.

### Labels, streams, chunks, and index

-   **Label:** Metadata such as `cluster`, `namespace`, `app`, or
    `environment`.
-   **Stream:** A unique combination of tenant and label set.
-   **Chunk:** Compressed log entries belonging to a stream over a time
    range.
-   **Index:** Metadata used to locate chunks that may contain matching
    logs.

Use labels for bounded, useful metadata. Avoid high-cardinality labels
such as request IDs, trace IDs, user IDs, or unbounded URLs.
High-cardinality labels can create many streams and increase memory,
index, and query costs. Keep such values in the log body and parse them
at query time when appropriate.

## 3. Architecture in this repository

The target environment described by the original request is:

-   OpenShift 4.18
-   Three control-plane nodes and six worker nodes
-   Production-oriented high availability
-   GitOps reconciliation through Argo CD
-   S3-compatible object storage
-   Prometheus metrics collection
-   OpenShift Route with TLS termination

The intended traffic flow is:

``` text
Applications / cluster log collector
              |
              v
       Loki write endpoint
              |
              v
       Write-side components
              |
              v
   S3-compatible object storage

Grafana / LogQL client
              |
              v
        OpenShift Route
              |
              v
        Loki gateway
              |
              v
        Read-side components
              |
              v
   Ingester data + object storage

Prometheus ----scrapes----> Loki metrics endpoints
Argo CD -------reconciles--> Kubernetes manifests
```

This is a conceptual flow, not a guarantee that every component in the
diagram is deployed by the current YAML set. Check the actual files and
the selected Loki version. A gateway is not automatically an
authentication system; configure and verify authentication and
authorization separately.

### Important architecture note

The requested `read`, `write`, and `backend` targets represent Loki's
**Simple Scalable Deployment (SSD)**. Grafana documents SSD as
deprecated for a future Loki 4.0 release. For a new long-lived
production deployment, evaluate Loki's distributed/microservices mode or
the OpenShift Loki Operator before committing to SSD. If your
organization explicitly standardizes on SSD, pin and validate a
supported Loki version and plan the future migration path.

For OpenShift environments that need OpenShift-integrated log collection
and operator lifecycle management, evaluate the Loki Operator and its
`LokiStack` custom resource. Do not mix operator-managed LokiStack
resources with a separately managed Loki installation unless you have
deliberately designed and tested that arrangement.

## 4. Loki deployment modes

Loki's main deployment choices are:

-   **Monolithic/single binary:** Most Loki components run in one
    process. Useful for development, small environments, and evaluation;
    it has fewer independently scalable parts.
-   **Simple Scalable Deployment (SSD):** Components are grouped into
    `read`, `write`, and `backend` targets. This is the layout
    represented by the requested manifest names, but it is deprecated
    for a future Loki 4.0 release.
-   **Distributed/microservices mode:** Components run as separate
    targets and can be scaled independently. This gives more operational
    control but increases deployment and troubleshooting complexity.
-   **Loki Operator / LokiStack on OpenShift:** The operator manages
    LokiStack resources and associated configuration. Confirm support
    and compatibility for the specific OpenShift, operator, and Loki
    versions in use.

Do not copy configuration from a different Loki release without checking
the configuration reference for the exact image version.

## 5. Repository layout

The requested layout may look like this:

``` text
loki/
├── 00-namespace.yaml
├── 01-serviceaccount-rbac.yaml
├── 02-loki-secret-template.yaml
├── 03-loki-config.yaml
├── 04-loki-write.yaml
├── 05-loki-read.yaml
├── 06-loki-backend.yaml
├── 07-loki-gateway.yaml
├── 08-loki-route.yaml
├── 09-loki-service-monitor.yaml
├── 10-loki-alerting-rules.yaml
├── 11-loki-read-hpa.yaml
├── 12-loki-gateway-hpa.yaml
├── 13-loki-pdb.yaml
└── 14-loki-network-policy.yaml
```

The names above describe the intended resource roles. If your repository
uses different names, update this table to match the committed files. A
file being present in a repository does not mean it is required or safe
to apply as-is.

  --------------------------------------------------------------------------------
  File                             Responsibility          Required?
  -------------------------------- ----------------------- -----------------------
  `00-namespace.yaml`              Creates a namespace for Usually yes, unless
                                   Loki                    managed elsewhere

  `01-serviceaccount-rbac.yaml`    Defines service account Only permissions
                                   and any required        actually needed by the
                                   permissions             workload

  `02-loki-secret-template.yaml`   Documents the required  A real Secret is
                                   object-store secret     required; never commit
                                   keys                    real credentials

  `03-loki-config.yaml`            Loki configuration,     Yes
                                   schema, storage,        
                                   limits, and retention   

  `04-loki-write.yaml`             Write target workload   Only for SSD
                                   and Service             architecture

  `05-loki-read.yaml`              Read target workload    Only for SSD
                                   and Service             architecture

  `06-loki-backend.yaml`           Backend target workload Only for SSD
                                   and Service             architecture

  `07-loki-gateway.yaml`           Gateway and internal    Required only if using
                                   routing                 this gateway design

  `08-loki-route.yaml`             Exposes a service       Only if external Route
                                   through an OpenShift    access is needed
                                   Route                   

  `09-loki-service-monitor.yaml`   Prometheus Operator     Only if your monitoring
                                   scrape configuration    stack supports/selects
                                                           it

  `10-loki-alerting-rules.yaml`    Prometheus alert rules  Recommended; verify
                                                           rule names and metric
                                                           availability

  `11-loki-read-hpa.yaml`          Read-target autoscaling Optional; needs working
                                                           metrics and resource
                                                           requests

  `12-loki-gateway-hpa.yaml`       Gateway autoscaling     Optional

  `13-loki-pdb.yaml`               Limits voluntary        Optional, recommended
                                   disruption of selected  for HA when correctly
                                   pods                    designed

  `14-loki-network-policy.yaml`    Restricts network       Optional, recommended
                                   traffic                 if compatible with
                                                           cluster networking
  --------------------------------------------------------------------------------

Some configurations require additional resources that may not be
represented by this list, such as cache services, runtime overrides, a
query scheduler, TLS trust bundles, or persistent storage for component
state. Review the actual configuration and release-specific
documentation before calling the set complete.

## 6. Storage and retention

### S3-compatible object storage

Loki can use S3-compatible object storage for durable log chunks and
index data. The endpoint may be AWS S3 or an on-premises/object-storage
product implementing the required S3 API. Compatibility, TLS, path-style
access, and authentication details vary by provider.

Before deployment, determine:

-   Object-store endpoint and region
-   Bucket name and access policy
-   Whether path-style access is required
-   TLS certificate chain and whether a custom CA must be trusted
-   Credentials or workload identity mechanism
-   Network connectivity from all Loki components
-   Object-store availability, capacity, request limits, and
    backup/recovery expectations

Use a dedicated bucket or carefully isolated prefixes and a
least-privilege identity. Do not put access keys in a ConfigMap, Git
history, README, or plain-text manifest.

### Schema and index

For modern Loki releases, TSDB with schema `v13` is the recommended
index store. The schema configuration includes a start date (`from`),
store type, object store, schema version, and index period. Schema
entries are migration-sensitive: choose the initial date correctly and
do not casually edit an existing schema entry after data has been
written. Follow the version-specific upgrade guidance when adding a
future schema period.

### Retention

Retention must be explicitly configured. For TSDB, Loki uses the
Compactor to apply retention and remove eligible stored data. Retention
configuration typically involves:

-   Enabling retention in the Compactor
-   Setting a global retention period and, if needed, stream-specific
    policies
-   Ensuring the index period is 24 hours for supported retention
    configurations
-   Providing durable storage for deletion marker files, or configuring
    the supported object-store marker location
-   Ensuring object-store lifecycle policies do not delete data earlier
    than Loki expects

A broad object-store lifecycle rule can delete chunks that Loki still
references and cause query failures. Loki's own retention mechanism
should be the primary policy; use bucket lifecycle rules only when their
scope and timing are deliberately coordinated.

Choose retention based on compliance, incident investigation needs,
storage cost, and privacy requirements. Verify actual deletion in a
non-production environment before relying on it.

## 7. Log ingestion and querying

### Ingestion

Loki does not automatically collect every OpenShift pod log just because
Loki is running. Install and configure a collector pipeline separately.
The collector must attach useful labels and send logs to the correct
push endpoint.

For OpenShift, choose an approach that is supported by your
organization's OpenShift Logging/Loki architecture. Avoid deploying
multiple collectors to read the same files unless duplicate ingestion is
intended.

### LogQL examples

Select logs for a namespace:

``` logql
{namespace="payments"}
```

Select logs from an application and search for errors:

``` logql
{namespace="payments", app="payment-api"} |= "error"
```

Exclude health-check noise:

``` logql
{namespace="payments"} != "/health"
```

Count matching log lines over five minutes:

``` logql
sum(count_over_time({namespace="payments"} |= "error" [5m]))
```

These examples work only if the collector actually supplies the labels
shown. Label names depend on the collection pipeline.

### Grafana

Configure Grafana with the Loki query endpoint appropriate to the
deployment. For internal Grafana, prefer the internal Service address
and enforce network access. For external clients, use the Route only
after confirming TLS, authentication, and authorization. The Route's TLS
termination setting determines where encryption ends; it does not by
itself guarantee end-to-end TLS.

## 8. OpenShift integration

### Routes and TLS

An OpenShift Route can expose the gateway through the cluster's ingress
controller. Choose the TLS termination mode deliberately:

-   **Edge:** TLS terminates at the router; traffic from router to the
    service is normally HTTP unless re-encryption is configured
    elsewhere.
-   **Re-encrypt:** TLS terminates at the router and a new TLS
    connection is made to the backend.
-   **Passthrough:** TLS terminates at the backend service.

Use the mode that matches the gateway's listener and security
requirements. Confirm the generated hostname, certificate chain, and
client authentication design. Do not publish a write endpoint publicly
without a strong reason and appropriate access controls.

### Prometheus

A `ServiceMonitor` is discovered only when the Prometheus instance's
service-monitor selector matches it and the relevant namespace selection
allows it. OpenShift's user-workload monitoring and platform monitoring
are distinct configurations; verify which Prometheus instance should
scrape Loki.

Confirm that: - The ServiceMonitor selects the correct Service labels. -
The Service exposes the correct metrics port and named port. - The
endpoint path and TLS settings match the target. - Prometheus can reach
the endpoint under the cluster's network and security policy. - The
metrics are actually exposed by the chosen Loki version and target.

### Security context and SCC

OpenShift commonly runs containers under restricted security policies
with an assigned non-root UID. Do not assume an image can write to any
arbitrary filesystem path. Validate volume ownership, writable
directories, capabilities, seccomp, and the selected Security Context
Constraints (SCC). Avoid granting `anyuid`, privileged access, or broad
RBAC just to make a pod start; solve permissions using supported image
and volume configuration.

## 9. High availability and scaling

A three-control-plane/six-worker cluster provides cluster capacity, but
does not automatically make Loki highly available. Loki's replica count,
storage, scheduling, network paths, and object store must all be
designed for failure.

### Scheduling

-   Run replicas across separate worker nodes where possible.
-   Use pod anti-affinity or topology spread constraints.
-   Set realistic CPU and memory requests and limits based on load
    tests.
-   Check node capacity and avoid scheduling all Loki pods onto one
    worker.
-   Consider dedicated worker pools and taints/tolerations only if your
    platform standard requires them.

### PodDisruptionBudget

A PDB is not required for a pod to start. It can limit voluntary
disruptions during maintenance, but it cannot prevent an involuntary
node failure and does not replace replicas or anti-affinity. Ensure
`minAvailable` or `maxUnavailable` is compatible with the replica count;
a PDB that allows no disruption can block node drains.

### Horizontal Pod Autoscaler

An HPA is optional. It needs resource requests and a functioning metrics
API. CPU-based scaling is only useful if CPU is a meaningful signal for
the workload. Tune scaling behavior, min/max replicas, and capacity
limits from measured ingestion and query demand. Do not assume
autoscaling alone guarantees performance or data durability.

### State and write durability

Validate replication factor, ring/KV-store configuration, ingester
lifecycle, WAL behavior, persistent volume requirements, and shutdown
handling for the exact Loki version and deployment mode. Object storage
provides durable long-term storage but does not remove the need to
design for in-flight data, component restarts, and network outages.

## 10. Security considerations

-   **Never commit real credentials.** Add secret files containing real
    credentials to `.gitignore`; preferably use an approved secret
    manager or External Secrets workflow. If a secret has ever been
    committed, rotate it; deleting it in a later commit does not remove
    it from Git history.
-   **Restrict S3 access.** Use a dedicated least-privilege identity and
    a bucket policy limited to the required bucket/prefix and
    operations.
-   **Use TLS.** Validate certificates and CA trust for object storage
    and external endpoints.
-   **Protect the gateway.** A gateway is a routing component, not
    automatically an identity provider. Configure authentication,
    authorization, and tenant-header handling deliberately.
-   **Plan tenancy.** In multi-tenant mode, Loki uses the
    `X-Scope-OrgID` header. Do not trust arbitrary client-supplied
    tenant IDs without an authenticated proxy that enforces tenant
    identity.
-   **Restrict network paths.** Apply NetworkPolicies only after
    understanding the CNI behavior and allowing all required peer, DNS,
    metrics, and object-store traffic.
-   **Minimize RBAC.** Grant only the permissions required by the
    workload. Loki generally should not need broad access to Kubernetes
    API resources simply to store and query logs.
-   **Pin images.** Use organization-approved images and preferably
    immutable digests; keep image provenance and vulnerability review in
    your release process.
-   **Protect logs as data.** Logs may contain tokens, personal data,
    request bodies, or secrets. Redact sensitive fields at source and
    set retention and access controls appropriately.

## 11. GitOps with Argo CD

Store reviewed manifests in Git and let Argo CD reconcile the desired
state into the cluster.

Recommended workflow:

1.  Keep separate directories or overlays for development, test, and
    production.
2.  Pin image versions and record the intended Loki version.
3.  Keep credentials out of the repository. Reference secrets created by
    an approved secret-management mechanism.
4.  Review diffs before merging changes.
5.  Apply changes first in a non-production environment.
6.  Monitor Argo CD sync and health status after merge.
7.  Test rollback procedures. Be aware that reverting a manifest does
    not necessarily reverse data-schema changes or restore deleted
    object-store data.

A GitOps controller does not validate whether a Loki configuration is
semantically compatible with a binary version. Add configuration
validation and cluster smoke tests to the pipeline where possible.

## 12. Deploy and validate

> Run these commands only against the intended test or production
> cluster. Replace the namespace, route, and resource names if your
> repository differs.

Inspect manifests locally:

``` bash
find loki -maxdepth 1 -type f -print
```

If `kubeconform` is part of your validation toolchain, use it for
Kubernetes schema checks, while remembering that CRDs and custom
resources need the relevant schemas:

``` bash
kubeconform -strict -summary loki/
```

Review changes before applying:

``` bash
oc diff -f loki/
```

For a manually managed deployment (not an operator-managed LokiStack),
apply in a test environment only after required Secrets and
configuration are ready:

``` bash
oc apply -f loki/
```

Inspect workloads:

``` bash
oc get pods,svc,route -n loki
oc get events -n loki --sort-by=.lastTimestamp
oc get endpoints -n loki
```

Inspect a failing pod:

``` bash
oc describe pod <pod-name> -n loki
oc logs <pod-name> -n loki
oc logs <pod-name> -n loki --previous
```

Check readiness and metrics endpoints using the actual Service names and
ports from your manifests. Validate the Route with the hostname and TLS
mode you configured. Do not assume a successful `oc apply` means Loki is
healthy or accepting logs.

### Minimum acceptance tests

-   All expected pods become Ready.
-   Loki's readiness endpoint returns success.
-   The gateway can reach the intended read/write services.
-   A test log can be pushed and queried back.
-   Logs remain queryable after a pod restart.
-   Data is written to the expected object-store bucket/prefix.
-   Retention behavior is tested in a non-production bucket.
-   Prometheus discovers Loki metrics and alert rules load.
-   A worker drain does not cause an unacceptable outage.
-   Authentication, authorization, tenant isolation, and TLS behavior
    are verified.

## 13. Troubleshooting

  -----------------------------------------------------------------------
  Symptom                             What to check
  ----------------------------------- -----------------------------------
  Pods are `Pending`                  Resource requests, node capacity,
                                      affinity/spread, taints, PVC
                                      binding

  Pods are `CrashLoopBackOff`         Container logs, Loki config
                                      validation, command-line flags,
                                      filesystem permissions

  `AccessDenied` from object storage  Credentials, IAM/bucket policy,
                                      bucket and prefix, endpoint/region

  TLS or certificate errors           CA bundle, certificate SAN,
                                      endpoint hostname, trust
                                      configuration

  Gateway returns `502`/`503`         Service selectors/endpoints,
                                      backend readiness, ports, gateway
                                      upstream configuration

  Logs are not visible                Collector status, push endpoint,
                                      labels, tenant ID, timestamps,
                                      query time range

  `ServiceMonitor` is ignored         Prometheus selectors, namespace
                                      selection, Service labels, named
                                      metrics port

  HPA has unknown metrics             Metrics API, resource requests,
                                      metrics availability, target
                                      resource

  Node drain is blocked               PDB selector and availability
                                      budget, replica readiness

  Retention is not deleting logs      Compactor configuration, retention
                                      enablement, schema/index period,
                                      marker storage, policy scope
  -----------------------------------------------------------------------

Useful diagnostic commands:

``` bash
oc get pods -n loki -o wide
oc get events -n loki --sort-by=.lastTimestamp
oc describe pod <pod-name> -n loki
oc logs deploy/<deployment-name> -n loki
oc get servicemonitor,prometheusrule -n loki
oc get route -n loki
```

Adjust resource types and names to the actual manifests. A StatefulSet
pod's logs are not retrieved using a Deployment name.

## 14. Production readiness checklist

-   [ ] Exact OpenShift, Loki, and collector versions are recorded and
    compatible.
-   [ ] Deployment mode is selected intentionally; SSD deprecation has
    been considered.
-   [ ] Manifests are validated against the selected Loki release.
-   [ ] S3-compatible storage endpoint, bucket, TLS, and permissions are
    tested.
-   [ ] No live credentials or private certificates are committed to
    Git.
-   [ ] Authentication and authorization are configured and tested.
-   [ ] Tenant-header trust and tenant isolation are understood.
-   [ ] Retention and deletion are tested with non-production data.
-   [ ] Resource requests, limits, replicas, and scheduling rules are
    load-tested.
-   [ ] WAL and in-flight write recovery have been tested.
-   [ ] Prometheus scrapes the intended metrics endpoints.
-   [ ] Alerts are tested and route to an owned notification
    destination.
-   [ ] Pod disruption budgets are compatible with maintenance
    procedures.
-   [ ] NetworkPolicies permit required traffic and block unnecessary
    paths.
-   [ ] Backups/recovery, upgrades, rollback, and object-store failure
    procedures are documented.
-   [ ] A representative log ingestion and query test passes after
    deployment.

## 15. Official documentation

Use documentation that matches the exact Loki release and OpenShift
version deployed.

-   [Loki
    architecture](https://grafana.com/docs/loki/latest/get-started/architecture/)
-   [Loki deployment
    modes](https://grafana.com/docs/loki/latest/get-started/deployment-modes/)
-   [Loki configuration
    reference](https://grafana.com/docs/loki/latest/configure/)
-   [Loki
    storage](https://grafana.com/docs/loki/latest/configure/storage/)
-   [Loki
    retention](https://grafana.com/docs/loki/latest/operations/storage/retention/)
-   [Loki LogQL](https://grafana.com/docs/loki/latest/query/)
-   [OpenShift
    documentation](https://docs.openshift.com/container-platform/4.18/welcome/index.html)
-   [OpenShift Logging
    documentation](https://docs.openshift.com/container-platform/4.18/observability/logging/cluster-logging.html)
-   [Argo CD documentation](https://argo-cd.readthedocs.io/en/stable/)

------------------------------------------------------------------------

## Disclaimer

This repository is an educational and operational starting point, not a
substitute for release-specific validation, security review, capacity
planning, or production testing. Kubernetes accepts syntactically valid
manifests that may still contain incompatible Loki flags, incomplete
networking, insecure exposure, or incorrect storage behavior. Verify the
exact configuration and test failure recovery before production use.
