# Azure DevOps, Docker, and Kubernetes / AKS

This guide brings together Azure deployment and CI/CD, Docker/container, Kubernetes, and AKS interview preparation from the Questpond interview materials. Related prompts from separate interviews are grouped by topic; overlapping questions are consolidated without discarding distinct operational details.

## 1. Azure DevOps and CI/CD

### What is Azure DevOps?
**Answer:** Azure DevOps is a suite of development and delivery services. Azure Repos hosts Git repositories; Azure Boards tracks work; Azure Pipelines automates build, test, and deployment; Azure Artifacts hosts package feeds; and Azure Test Plans supports test planning and execution. Teams can use individual services or integrate them into one delivery workflow.

### What is CI/CD?
**Answer:** Continuous integration (CI) frequently integrates changes and runs automated validation. Continuous delivery/deployment (CD) packages validated changes and promotes them through environments, with manual approvals where required or automatic deployment when policy allows.

### How would you implement CI/CD for a .NET application using Azure DevOps?
**Answer:** Define a YAML pipeline in source control that restores dependencies, builds, runs unit and integration tests, performs code/dependency/security checks, and publishes a versioned artifact or container image. Deploy the same immutable artifact to test/staging/production environments, apply approvals and health checks, and support rollback or slot-based deployment. Use least-privilege service connections, federated identity where available, and secure variable groups/Key Vault integration for secrets.

### What is the difference between a build pipeline and a release/deployment pipeline?
**Answer:** A build stage validates source and creates a versioned artifact. Deployment stages take that artifact and promote it into one or more environments with environment-specific configuration, approvals, health verification, and rollback. Keeping the artifact immutable avoids rebuilding different binaries for each environment.

### What should a CI/CD pipeline do for a containerized AKS workload?
**Answer:**
1. Restore, build, test, and scan application dependencies/source.
2. Build a Docker image from the tested source and scan it.
3. Tag the image with a unique build/release identifier and push it to Azure Container Registry (ACR).
4. Validate Kubernetes manifests or Helm charts, including environment-specific values.
5. Deploy the exact image digest to a non-production environment and run smoke/integration checks.
6. Promote through controlled environments, verify readiness and telemetry, and retain a rollback path.

Do not place long-lived credentials in pipeline YAML or images. Restrict service connections and registry/cluster access to the minimum required permissions.

### How should a pipeline deploy an Azure Function?
**Answer:** Build and test the Function, publish it through a CI/CD pipeline or the supported Functions deployment tooling, configure app settings and identity/network access outside the artifact, deploy to the target environment (or staging slot where supported), run a smoke test, then promote and monitor failures and latency.

### How do you manage secrets and deployment configuration?
**Answer:** Keep environment-specific configuration outside the immutable build artifact. Use managed identities/workload identity and a secret store such as Key Vault where supported; grant least-privilege access. Avoid committing secrets, echoing secret values in pipeline logs, or baking credentials into a Docker image.

### How do you monitor a deployment and roll back?
**Answer:** Check health/readiness, errors, latency, saturation, dependency failures, and application logs/traces after deployment. Use a staged rollout, deployment slots or progressive traffic shifting where supported, and a previously known-good artifact/image digest for rollback. Define actionable alerts and an explicit rollback owner/process before release.

## 2. Docker and container fundamentals

### What is the difference between a container image and a container?
**Answer:** An image is a packaged, versioned filesystem and configuration template. A container is a running instance of an image. Containers isolate processes and user-space dependencies while sharing the host kernel; they are not full virtual machines.

### What is a Dockerfile, and how do you create and run an image?
**Answer:** A Dockerfile declares how to build an image, including its base image, files, build steps, runtime configuration, and entry point. Common commands are:

```sh
docker build -t my-service:dev .
docker run --rm -p 8080:8080 my-service:dev
docker tag my-service:dev myregistry.azurecr.io/my-service:build-id
docker push myregistry.azurecr.io/my-service:build-id
```

For a .NET service, use a multi-stage build: restore/build/publish with an SDK image, then copy only the published output into a supported runtime image. Exclude unnecessary files with `.dockerignore`.

### How do you choose a base image for a microservice?
**Answer:** Match the supported runtime and CPU architecture; prefer a trusted, maintained, minimal image compatible with the application's native dependencies and operational requirements. Consider patch cadence, image size, debugging needs, and organization policy. Pin a production image version or digest for reproducibility; do not select an image solely by its tag.

### Are Docker tags immutable?
**Answer:** Not necessarily. A tag is a human-readable reference and registries can allow it to be moved to a different image. Use unique build tags and deploy by immutable digest when you need to ensure the exact tested image is promoted. Avoid relying on `latest` for production releases.

### What are Docker image security best practices?
**Answer:** Use trusted, patched base images; minimize installed packages and build context; run as a non-root user; avoid embedding secrets; scan dependencies and images; pin dependencies where appropriate; restrict capabilities; and retain image provenance/signature information when the release process supports it.

### What is a container registry, and what is ACR?
**Answer:** A registry stores and distributes container images. Azure Container Registry (ACR) is Azure's managed registry for container images and related artifacts. A pipeline can push a versioned image to ACR, and AKS can pull it using a properly scoped identity and registry permissions.

### How should application configuration be provided to a container?
**Answer:** Keep environment-specific configuration outside the image and inject it at deployment/runtime. Use configuration values or mounted configuration for non-secret settings and a managed secret mechanism/identity for credentials. Do not bake environment secrets into image layers or publish them as plain text in manifests.

## 3. Kubernetes concepts

### What is Kubernetes?
**Answer:** Kubernetes orchestrates containerized workloads. It schedules pods onto nodes and provides declarative workload management, service discovery, health-based replacement, rollout, and scaling. It adds operational complexity and is not automatically the right choice for every application.

### What is the difference between a container and a pod?
**Answer:** A container runs a process from an image. A pod is Kubernetes' smallest schedulable unit and contains one or more co-located containers. Containers in a pod share the pod's network namespace/IP and can share mounted volumes; separate unrelated applications into separate pods so they can scale and roll out independently.

### What is a Deployment?
**Answer:** A Deployment manages a desired number of interchangeable pod replicas and coordinates updates and rollbacks. It creates and manages ReplicaSets; the desired replica count and rollout strategy are declared rather than managed by manually creating individual pods.

### What is a Kubernetes Service?
**Answer:** A Service provides a stable network identity and load-balances traffic to a set of matching ready pods. Pods may be replaced and receive different IPs, while clients continue to use the Service's stable name/address.

### What is Ingress, and how does external HTTP traffic reach a pod?
**Answer:** An ingress controller or other cluster entry point accepts external traffic and applies host/path/TLS routing rules. It forwards traffic to a Kubernetes Service, which selects ready pod endpoints; cluster networking routes it to a pod. Configure TLS, timeouts, authentication, health checks, and observability. Ingress is not the application Service itself.

### Can one pod contain multiple different containers?
**Answer:** Yes, when the containers are tightly coupled and need to share lifecycle, network, or volumes—for example, an application with a supporting sidecar. They can use different images. Do not put unrelated independently scalable services in one pod.

### What is the sidecar pattern?
**Answer:** A sidecar is a companion container deployed in the same pod as the application to provide a supporting capability such as proxying, telemetry, or certificate handling. It shares the pod's network and may share volumes. It can reduce application-specific integration code but adds resource and operational overhead.

### What are readiness and liveness probes?
**Answer:** A readiness probe indicates whether a pod should receive traffic. A liveness probe indicates whether a container is unhealthy enough to restart. Configure probes to represent actual health without making transient downstream outages cause restart loops; startup probes can protect slow-starting containers from premature liveness failures.

### How should Kubernetes workloads scale?
**Answer:** Set realistic CPU/memory requests and limits, then choose replica counts and autoscaling from observed workload metrics and service objectives. A Horizontal Pod Autoscaler can adjust replicas; cluster/node autoscaling may add capacity when needed. Scaling policies should be tested under load and account for downstream capacity and startup time.

### What is a Kubernetes namespace?
**Answer:** A namespace partitions namespaced Kubernetes objects within a cluster, helping organize teams, applications, and environments. It can be combined with RBAC, resource quotas, and network policies, but is not by itself a strong security boundary between hostile tenants.

### What is a ConfigMap vs. a Secret?
**Answer:** A ConfigMap holds non-sensitive configuration. A Secret is intended for sensitive values, but access still requires RBAC and secure cluster configuration; do not assume that a Secret object alone provides adequate encryption or secret management. Prefer workload identity or an integrated secret store when appropriate.

### What are resource requests and limits?
**Answer:** Requests describe resources Kubernetes should reserve/consider when scheduling a pod. Limits cap consumption for supported resources. Set them based on measurement: incorrect requests can waste capacity or cause scheduling problems, while overly low limits can throttle CPU or terminate memory-heavy containers.

## 4. AKS, Helm, networking, and identity

### What is AKS?
**Answer:** Azure Kubernetes Service (AKS) is Azure's managed Kubernetes service. Azure manages parts of the control-plane experience; teams remain responsible for workload configuration, node/workload capacity, access control, networking, deployment, and operations according to their chosen AKS configuration.

### How would you design AKS in a VNet? How many node pools and microservices?
**Answer:** Choose node pools from workload, resource, availability, and isolation needs—not from a fixed number of services. A separate system pool and user pool is a common starting point; isolate specialized, bursty, or regulated workloads only when justified. Plan VNet/subnet capacity, routes, DNS, private connectivity, identity, and network policy. Each workload should declare resources, probes, scaling, disruption behavior, ingress, configuration, and deployment ownership.

### What does Helm do?
**Answer:** Helm packages Kubernetes manifests into charts with configurable values and release/version management. Charts allow repeatable deployments while keeping common templates separate from environment-specific configuration. Review rendered output and values carefully; Helm does not replace understanding the Kubernetes resources being installed.

### How do you validate a Helm chart locally?
**Answer:**

```sh
helm lint ./my-chart
helm template my-release ./my-chart -f values-dev.yaml
helm install my-release ./my-chart --dry-run --debug -f values-dev.yaml
```

`helm lint` checks chart structure, `helm template` renders manifests locally, and a dry run checks a proposed release. Docker builds container images; it does not validate Helm charts.

### How does a pod access Azure Storage or Key Vault?
**Answer:** Prefer AKS Workload Identity/federated identity to obtain short-lived tokens for a least-privilege identity. Configure the necessary Azure role permissions and network path; use private endpoints/private DNS where required. Avoid embedding account keys or client secrets in images or plain manifest values.

### What is the role of Azure networking for private AKS workloads?
**Answer:** Ensure the workload has a valid route to the resource and that DNS, firewall/NSG rules, private endpoint configuration, and identity permissions are all correct. A private endpoint does not by itself grant access; diagnose name resolution, routing, network policy, and authorization separately.

## 5. Integrated deployment walkthrough

For a .NET microservice on AKS, a defensible interview walkthrough is:

1. Commit application code, Dockerfile, Helm chart/manifests, and pipeline definition to source control.
2. CI restores, builds, tests, and scans the application and dependencies.
3. A multi-stage Docker build produces a minimal runtime image; CI scans it and pushes it to ACR under a unique identifier.
4. CI validates Helm/manifests and deploys the tested image digest to a test environment.
5. Kubernetes applies a Deployment, Service, configuration, identity, probes, and scaling policy; ingress exposes only the intended routes.
6. Verify rollout/readiness, run smoke tests, and inspect logs, metrics, traces, and dependency health.
7. Promote the same immutable image to the next environment with approvals as required; retain the previous release for rollback.

Clearly distinguish what the candidate personally implemented from team or platform responsibilities.

## 6. Interview questions collected from the source material

- How do you implement CI/CD for a .NET application in Azure?
- Have you used AKS? Describe your role with Helm, ingress, configuration, and CI/CD.
- What is ACR used for?
- What is the difference between a container and a pod?
- What do Helm charts do, and why are they useful?
- Which CLI tools can validate a Helm chart locally?
- How do you design Kubernetes in a VNet: node pools, services, networking, and deployment?
- Can a pod contain multiple containers of different images?
- Trace an external HTTP request through ingress, a Service, and a pod.
- How does a pod connect to Azure Storage or Key Vault?
- Have you worked with Dockerfiles? How do you build and deploy a microservice?
- How do you choose a base image, and what is the role of image tags?
- How do you configure AKS workload health, scaling, and rollback?

When answering experience questions, be precise about hands-on responsibility and do not claim work you did not perform.

## Source coverage

Questpond sources consolidated into this guide:

- `B:\TUTS\Questpond\07-azure.md` — Azure deployment and CI/CD question (PDF p. 11).
- `B:\TUTS\Questpond\20-epam.md` — AKS deployment with Helm/ingress/CI/CD, ACR, container vs. pod, Helm validation, and VNet questions (PDF pp. 44–45).
- `B:\TUTS\Questpond\30-virtusa.md` — Azure Functions and Dockerfile/microservice deployment questions (PDF p. 55).
- `B:\TUTS\Questpond\30-wolters.md` — Kubernetes/AKS in a VNet, pods, ingress, Azure identity/storage, sidecars, Docker images, and container questions (PDF pp. 59–61).
- `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf` — AKS, Helm, ACR, ingress, pod/storage, Docker, and Kubernetes design prompts (PDF pp. 45–46, 54–55).
- `B:\TUTS\Interview Prep\source\senior_dotnet_architect_viva_prep.md` — Docker/Kubernetes, deployment pipeline, image publishing, scanning, and rollout guidance.

The practice Q&A and walkthrough organize the source prompts into one study sequence; supplementary explanations clarify related fundamentals where the source contains prompts without answer text.
