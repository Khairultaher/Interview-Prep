# Azure Cloud Administration for Backend Developers

This guide combines Azure-service references from the CV in `B:\TUTS\H1B` with Azure administration material relevant to the backend topics in `Backend-Development-Viva-Preparation.md`—ASP.NET Core APIs, EF Core/database access, asynchronous work, REST, and GraphQL.

**Source scope:** The backend viva file does not name Azure products or provide Azure administration questions. The service inventory below is grounded in the CV; the broader administration explanations and interview Q&A are supplementary context. A technology appearing in a CV environment list does not, by itself, establish Azure administrator-level ownership.

## 1. Azure services and cloud technologies in the CV

The following inventory reflects services and technologies explicitly named in `B:\TUTS\H1B\MD_Khairul_Alam_Resume_2027.docx`. Use the project descriptions as interview anchors, and describe only the parts you personally implemented or operated.

### Application hosting and delivery

- **Azure App Service / Azure Web App:** The CV describes migrating an application to Azure App Service and Azure SQL. It also lists Azure Web App/App Service in project environments. This supports discussion of managed web/API hosting and migration; the CV does not specify plan SKU, slots, VNet integration, or deployment configuration.
- **Azure DevOps:** Listed in multiple project environments for delivery work. The CV does not detail particular Repos, Boards, Pipelines, service connections, environments, or YAML tasks; explain only the actual Azure DevOps responsibilities you performed.
- **Docker and Kubernetes:** Listed in an earlier project environment, but not identified there as Azure AKS. Discuss container/Kubernetes experience separately from AKS-specific operational experience unless you can confirm AKS usage.

### Identity, network, and application monitoring

- **Azure Active Directory:** Listed in project environments. This is the former name of Microsoft Entra ID. The CV does not specify whether this work involved workforce identity, app registrations, OAuth/OIDC configuration, RBAC, or workload identities; be precise about your actual role.
- **Azure Firewall:** Listed in environments for multiple projects. The CV does not describe rule design, network topology, policy management, or incident operations; do not imply those responsibilities unless accurate.
- **Azure Application Insights / Application Insights:** The CV states that Azure Application Insights monitoring/logging was implemented for .NET APIs, including APIs using .NET 8/.NET 9. This gives a concrete observability example; be ready to describe which telemetry, alerts, dashboards, and diagnostic workflows you actually configured.

### Databases, storage, and data integration

- **Azure SQL Database / Azure SQL Server:** The CV describes migrating an application's database to Azure SQL and lists Azure SQL across several application environments. Discuss schema/data migration, connection configuration, performance, and secure access only to the extent you owned them.
- **Azure Blob Storage:** Listed for a fault-tolerant file-upload application. The project environment also lists retry policies and large-file upload support; this can anchor discussion of object storage, retry behavior, and upload design.
- **Azure Files / File Share:** Listed in the same file-upload application environment. The CV does not detail share provisioning, identity, mount protocol, network restrictions, or lifecycle administration.
- **Azure Data Factory:** Listed in ETL/data environments. Explain pipeline orchestration, source/target integrations, scheduling, and monitoring only if those were part of your own work.
- **Azure Databricks:** The CV describes leading design/development of an Azure Databricks ETL pipeline. This is a direct project anchor for data engineering and pipeline discussion; cluster policy, workspace administration, and networking are not specified.
- **Microsoft Fabric and Power BI:** The CV describes ETL with Python/Pandas, Azure Functions, and Microsoft Fabric, with operational and analytical reporting in Power BI. Discuss the implemented data flow and your specific Fabric/Power BI role rather than assuming tenant/workspace administration.

### Serverless, communication, and AI

- **Azure Functions:** The CV describes using Azure Functions with Python/Pandas for ETL pipelines and separately lists Functions in a .NET environment. This supports discussion of event-driven or scheduled processing where relevant; the trigger types and hosting plan are not specified.
- **Azure Communication Services:** Listed in a .NET 9 project environment; the CV does not specify which communication channel or configuration was implemented.
- **Azure AI Document Intelligence:** The CV describes document/image reading in an application. Use this as the concrete AI/document-processing example and explain the actual integration and data handling.
- **Azure AI Content Understanding:** The CV describes integrating it to extract structured information from scanned documents and unstructured files.
- **Azure AI Foundry:** The CV says Azure AI Foundry, agentic AI, and MCP server capabilities were introduced for AI-powered workflows. Be prepared to explain which features were prototyped or delivered and how access, evaluation, and data protection were handled.

### CV-grounded project examples

1. **Multi-tenant operations platform:** .NET 9 REST API, tenant-aware workflows, authentication/authorization, Azure Application Insights monitoring, Azure SQL Database, Azure Functions, Microsoft Fabric, Power BI, Azure AI Content Understanding, and Azure AI Foundry. The CV describes tenant isolation and centralized tenant management.
2. **Document and data-processing applications:** .NET 9 API with Azure Application Insights; Azure AI Document Intelligence; Python/Pandas, Azure Functions, and Microsoft Fabric ETL; Power BI reporting. Azure Firewall and Azure DevOps appear in the listed environment.
3. **Azure-hosted application migration:** Application migrated to Azure App Service and database migrated to Azure SQL, with Application Insights and Azure DevOps listed in the environment.
4. **File-upload application:** Fault-tolerant uploads with custom retry policies and large-file support; Azure SQL, Azure Data Factory, Azure Files/File Share, Azure Blob Storage, Azure Active Directory, and Azure DevOps are listed.
5. **Earlier .NET/microservices project:** Docker and Kubernetes are listed in the environment, alongside AWS infrastructure. The CV does not identify this as AKS, so do not present it as AKS experience without confirming.

**Interview framing:** Separate (a) services named in the environment, (b) features you implemented, and (c) infrastructure you personally provisioned or administered. Where the CV only names a service, be candid about that boundary.

## 2. Azure organization and administration

### Microsoft Entra ID
Microsoft Entra ID is Microsoft's cloud identity and access management service (formerly Azure Active Directory). It manages users, groups, application identities, authentication, and access to Azure and connected applications.

### Tenant, management group, subscription, and resource group
- **Tenant:** An organization's Microsoft Entra identity boundary.
- **Management group:** A way to organize subscriptions and apply governance consistently above the subscription level.
- **Subscription:** A billing, quota, and access-control boundary for Azure resources.
- **Resource group:** A lifecycle and management container for related Azure resources; resources in a group can have different regions.

### Azure Resource Manager (ARM)
Azure Resource Manager is the management plane used to deploy and manage Azure resources. ARM templates, Bicep, and Terraform can describe infrastructure as code. Separate resource-management operations from application data-plane access—for example, creating a Storage account versus reading a blob.

### Azure RBAC
Azure role-based access control grants principals permissions on a management scope such as a management group, subscription, resource group, or resource. Assign the least-privilege built-in or custom role at the narrowest practical scope, and review inherited permissions.

### Azure Policy, resource locks, and tags
- **Azure Policy** evaluates or enforces organizational rules such as allowed regions, required tags, or diagnostic settings.
- **Resource locks** help prevent accidental deletion or modification; they do not replace RBAC.
- **Tags** add metadata such as environment, owner, cost center, and application for governance and cost reporting.

### Cost Management
Use budgets, cost alerts, tags, and cost analysis to understand spend by subscription, resource group, service, and workload. Autoscaling, retention, log ingestion, and idle resources can materially affect backend hosting costs.

## 3. Identity and access for applications

### User identity vs. workload identity
- **Human identity:** An employee or administrator signing in through Microsoft Entra ID; secure privileged operations with role-based access and appropriate sign-in controls.
- **Application/workload identity:** An application authenticating to another service without a person present.
- **Managed identity:** An Azure-managed identity for a supported resource. It avoids storing an application credential in configuration.
- **Service principal/app registration:** An application identity used for authentication and permissions; credentials or certificates must be protected and rotated if used.

### System-assigned vs. user-assigned managed identity
A system-assigned identity has the lifecycle of its Azure resource. A user-assigned identity is a separate resource that can be attached to multiple supported resources and can persist when an individual workload is replaced. Choose based on lifecycle and access-boundary needs, then grant only necessary roles.

### Authentication and authorization
Authentication establishes who or what is calling. Authorization decides which resources and operations that identity may use. A valid Entra token does not automatically grant access to every Azure resource; configure the required application permissions or Azure data-plane roles and enforce application-level authorization as well.

## 4. Azure networking

### Virtual Network and subnet
A Virtual Network (VNet) is a private network boundary in Azure. Subnets divide its address space for workloads or network components. Design address ranges and subnet capacity before deployment, especially for private endpoints and AKS.

### Network Security Group
An NSG applies inbound and outbound allow/deny rules to network interfaces or subnets. Review effective rules and avoid assuming that a subnet is private merely because it has an NSG.

### User-defined routes and network egress
User-defined routes (UDRs) can direct traffic through a firewall, network virtual appliance, or other next hop. A NAT Gateway provides controlled outbound connectivity for supported subnet workloads. Validate DNS, routing, and firewall rules together when diagnosing connectivity.

### Private Endpoint and Private DNS
A Private Endpoint gives a supported Azure service a private IP address in a VNet. Private DNS zones provide the expected private name resolution. Private connectivity alone does not authorize requests: identity, service permissions, firewall configuration, DNS, and routing must all be correct.

### Load balancers and application entry points
- **Azure Load Balancer:** Layer-4 traffic distribution for supported network workloads.
- **Azure Application Gateway:** Layer-7 web traffic management, TLS termination, and optional Web Application Firewall (WAF) capabilities.
- **Azure Front Door:** Global edge entry, routing, acceleration, and optional WAF capabilities.
- **Azure API Management (APIM):** An API gateway and API-management platform for publishing APIs and applying policies such as authentication integration, quotas, and transformations.

Choose based on layer, geographic needs, API-management requirements, and operations; these services are not interchangeable.

### VPN Gateway, ExpressRoute, and Bastion
- **VPN Gateway** connects networks using encrypted VPN connectivity.
- **ExpressRoute** provides private connectivity between an organization's network and Azure through a connectivity provider.
- **Azure Bastion** enables browser-based access to supported virtual machines without exposing their management ports directly to the public internet.

## 5. Hosting and compute for backend applications

### Azure App Service
App Service is a managed hosting platform for web applications and APIs. It supports deployment options, app settings, TLS, scaling, deployment slots (depending on plan/features), and platform integration. It is a strong option when a team wants managed web hosting without operating Kubernetes or individual virtual machines.

**Typical use:** Host an ASP.NET Core REST or GraphQL API, connect it to managed data services, use a managed identity for Azure dependencies, and instrument it with Application Insights.

### Azure Container Apps
Azure Container Apps is a managed platform for containerized applications and jobs, with platform features for ingress, scaling, and application environments. It can suit container workloads that need less direct Kubernetes administration than AKS.

### Azure Kubernetes Service
AKS is managed Kubernetes for teams that need Kubernetes orchestration and control over Kubernetes workloads. The team still owns workload manifests/charts, access, capacity, networking, security configuration, observability, and operational practices. Choose it when those capabilities justify the additional complexity.

### Azure Container Instances
Azure Container Instances runs container groups without requiring a Kubernetes cluster. Consider it for suitable isolated or short-lived container workloads; it is not a drop-in replacement for a full orchestration platform.

### Azure Functions
Azure Functions is event-driven serverless compute for triggers such as HTTP requests, timers, queues, and events. It can support background or scheduled work related to an API. Choose a hosting plan and execution model that meet workload duration, scaling, networking, and operational needs.

### Virtual Machines and Virtual Machine Scale Sets
Virtual Machines provide operating-system-level control and flexibility, with corresponding responsibility for patching, hardening, availability, and maintenance. Virtual Machine Scale Sets manage groups of similar VMs for scalable VM-based workloads. Prefer a managed application platform when it meets requirements and reduces operational burden.

### Azure Container Registry
Azure Container Registry (ACR) stores and distributes container images and related artifacts. Use a controlled build pipeline to publish versioned images, scan them, and grant the deployment platform a narrowly scoped pull identity.

## 6. Data, secrets, and messaging

### Azure Key Vault
Key Vault stores and controls access to secrets, keys, and certificates. Prefer managed identity for applications retrieving secrets or using supported key operations. Apply least privilege, auditing, network controls where applicable, and rotation processes.

### Azure Storage
Azure Storage includes Blob, Queue, Table, and Files services. Blob Storage is commonly used for files and objects; Queue Storage offers simple queueing. Use a managed identity and narrowly scoped data roles where supported. Do not expose account keys in application code or container images.

### Azure SQL Database and SQL Managed Instance
Azure SQL Database is a managed relational database service; SQL Managed Instance offers broader SQL Server compatibility and instance-level capabilities. Select based on feature compatibility, migration needs, availability, network access, performance, and cost. EF Core can access these services through supported database providers.

### Azure Service Bus
Service Bus provides enterprise messaging with queues, topics/subscriptions, dead-lettering, and delivery features. Use it to decouple API requests from background processing or distribute messages among independent consumers. Design for retries, duplicate delivery, idempotency, poison messages, and queue-age monitoring.

### Event Grid and Event Hubs
- **Event Grid** routes event notifications to subscribers.
- **Event Hubs** ingests high-volume event streams for independent consumers and processing.

Select based on whether the requirement is event notification/routing, durable business messaging, or high-throughput streaming.

## 7. Monitoring, diagnostics, and security

### Azure Monitor
Azure Monitor is the umbrella monitoring service for metrics, logs, alerts, and related observability capabilities across Azure resources. Configure resource diagnostic settings to send supported logs and metrics to destinations such as Log Analytics.

### Application Insights
Application Insights provides application performance monitoring and distributed tracing for applications. Instrument backend services to capture requests, dependencies, exceptions, traces, and performance data. Use correlation/trace context across service calls, and avoid capturing credentials or sensitive payloads.

### Log Analytics workspace
A Log Analytics workspace stores and queries log data using Kusto Query Language (KQL). It can support centralized operational analysis across applications and Azure resources, subject to configured diagnostic settings, workspace design, access control, and retention.

### Alerts, workbooks, and dashboards
Create alerts from meaningful metrics or log queries, define an Action Group for notification/action, and test that alerts are actionable. Use workbooks and dashboards to review service health and investigate incidents across resources.

### Microsoft Defender for Cloud
Defender for Cloud provides security posture management and workload-protection capabilities for supported environments. Use its recommendations alongside secure configuration, patching, access reviews, and application-specific threat modeling; it does not automatically secure an application.

## 8. Common backend deployment patterns

### Pattern A: Managed API
**App Service** hosts the ASP.NET Core API; **Microsoft Entra ID** provides identity; **managed identity** accesses **Azure SQL** and **Key Vault**; **Application Insights/Azure Monitor** provide telemetry; **APIM** is added if centralized API publishing or gateway policies are needed.

### Pattern B: Containerized API
The build pipeline creates and scans an image, publishes it to **ACR**, then deploys it to **Azure Container Apps** or **AKS**. **Entra/workload identity** secures service access; **VNet/private endpoints** provide private network paths where required; **Azure Monitor/Application Insights** support operations.

### Pattern C: API with asynchronous work
An **App Service** or containerized API accepts a request and enqueues work in **Service Bus**. A worker or **Azure Function** processes the message idempotently, accesses data using managed identity, and reports status. Monitor queue age, failures, retries, dead-letter volume, and end-to-end traces.

## 9. Interview questions to practice

1. How do Entra ID, Azure RBAC, and application authorization differ?
2. When should an application use a managed identity instead of a stored service credential?
3. Explain the Azure hierarchy: tenant, management group, subscription, and resource group.
4. How would you securely connect an App Service API to Azure SQL or Key Vault using private networking?
5. What are the purposes of a VNet, subnet, NSG, route table, private endpoint, and private DNS zone?
6. Compare App Service, Container Apps, AKS, Functions, Container Instances, and Virtual Machines.
7. When would you add APIM, Application Gateway/WAF, Front Door, or a load balancer?
8. How do Application Insights, Azure Monitor, and Log Analytics relate?
9. How would you investigate an API that is slow or failing to reach a private dependency?
10. When do you choose Service Bus, Event Grid, Event Hubs, or Storage Queues?
11. What controls can prevent accidental exposure, excessive permissions, and unexpected resource cost?
12. How would you deploy, monitor, and roll back a backend service in Azure?

For scenario questions, explain the request/data flow, identity, network path, deployment choice, monitoring, failure handling, and trade-offs. State which products you have used directly; do not present study knowledge as hands-on experience.

## Source coverage

- `B:\TUTS\H1B\MD_Khairul_Alam_Resume_2027.docx` — Azure services and project examples in the CV-grounded inventory.
- `B:\TUTS\H1B\Backend-Development-Viva-Preparation.md` — backend concepts used to organize the Azure service mapping. The file itself does not name Azure services.
- The Azure organization, identity, networking, hosting, data, monitoring, and interview sections add supplementary administration context beyond the services explicitly described in the CV.

---

## Additional Questpond question sets

### Azure

###### 1. What is Azure App Service, and how do you deploy a .NET application to it? (PDF p. 11)
**Answer:** Azure App Service is a managed platform for hosting web apps and APIs without managing the underlying VM. Build and publish the application, configure the runtime/settings and deployment identity, then deploy through a supported pipeline, ZIP/package, or container workflow; verify health and logs after release.

###### 2. What are the options to deploy a Web API on Azure? (App Service / AKS / Azure Container Service) (PDF p. 11)
**Answer:** Options include App Service for managed web hosting, Azure Kubernetes Service (AKS) for orchestrated containers, and Azure Container Apps for many containerized workloads; virtual machines are another option when OS-level control is required. “Azure Container Service” is a retired product name; select based on operational needs, scaling, and workload.

###### 3. Why choose App Service to deploy a Web API? (PDF p. 11)
**Answer:** App Service is a good choice when the API needs managed hosting, straightforward deployment, built-in TLS and scaling options, and the team does not need to manage a cluster or VM. It reduces operations compared with self-managed infrastructure, but offers less low-level control than AKS/VMs.

###### 4. Difference between Azure App Service and Azure VM (PDF p. 11)
**Answer:** App Service is a managed application-hosting platform with less OS administration and built-in web deployment features. A VM provides control over the operating system and software stack but requires the team to manage patching, security, availability, and deployment operations.

###### 5. What are Azure Functions? How are they used with .NET? (PDF p. 11)
**Answer:** Azure Functions is a managed event-driven compute service that runs code in response to triggers such as HTTP, timers, queues, or events. A .NET Function implements a trigger binding and can use input/output bindings or SDKs to interact with other services.

###### 6. How will you secure an Azure Function? (PDF p. 11)
**Answer:** Use the least-privilege identity for resource access, authenticate HTTP callers with the platform/identity provider or API gateway, protect secrets in Key Vault, restrict network exposure where possible, validate inputs, and enable monitoring. Function keys are shared secrets, not a replacement for user authentication/authorization.

###### 7. I want to create a storage account and Function App. I don't want to enable the storage account, but the Function should read data; otherwise a hacker can tamper with the storage account. (PDF p. 11)
**Answer:** Azure Functions requires a storage account for host state and certain runtime operations; disabling the required host storage is not a secure design. Separate host storage from business data, use a managed identity with read-only data permissions, disable public access where feasible, use private endpoints/firewall rules, and audit access. If the concern is data tampering, enforce least privilege and data integrity rather than removing required runtime storage.

###### 8. What are Durable Functions? (PDF p. 11)
**Answer:** Durable Functions extend Azure Functions with stateful orchestration for workflows, including durable timers, retries, fan-out/fan-in, and human interaction. The runtime persists orchestration state so workflows can resume after function executions complete.

###### 9. How do you check whether a Function App is healthy? (PDF p. 11)
**Answer:** Use an application health endpoint or synthetic invocation, Azure Monitor/Application Insights availability and failure telemetry, platform health/status, and dependency checks. Alert on meaningful service-level indicators such as failed executions, latency, and queue age, not only process availability.

###### 10. “14. I want to receive an email if the Function App is failing. How do you configure this?” (PDF p. 11)
**Answer:** The source contains an embedded “14.” even though this is its tenth listed item. Configure an Azure Monitor alert on a suitable Function/App Insights metric or log query, attach an Action Group with an email recipient, and test the alert using an actionable failure condition and threshold.

###### 11. What is Azure Key Vault and why is it important? (PDF p. 11)
**Answer:** Key Vault securely stores and controls access to secrets, keys, and certificates, with auditing and rotation/integration options. It reduces hard-coded credentials and centralizes access policy; use managed identity and least privilege to retrieve only what an application needs.

###### 12. What is Managed Identity in Azure? (PDF p. 11)
**Answer:** Managed Identity gives an Azure resource an identity in Microsoft Entra ID so it can obtain tokens for supported services without storing credentials in application configuration. Assign only necessary roles and select system-assigned or user-assigned identity based on lifecycle/reuse needs.

###### 13. What is Key Vault and how do you connect to Key Vault? (PDF p. 11)
**Answer:** Key Vault is Azure's managed secret/key/certificate store. Enable a managed identity on the application, grant it the minimum required Key Vault data-plane role/access, then use the Azure SDK or configuration provider with that identity; do not embed client secrets in source code.

###### 14. How do you implement CI/CD for a .NET project in Azure? (PDF p. 11)
**Answer:** Define a pipeline that restores, builds, tests, scans, and packages the application, then deploys an immutable artifact through environments with approvals and health checks. Use federated/service connections with least privilege, store secrets securely, and support rollback or slot-based deployment.

###### 15. What is Azure Service Bus and how does it work with .NET? (PDF p. 11)
**Answer:** Azure Service Bus is a managed enterprise messaging service supporting queues and topics, durable delivery, dead-lettering, and competing consumers. A .NET service uses `Azure.Messaging.ServiceBus` clients/processors to send and consume messages; handle retries and idempotency because delivery may be at least once.

###### 16. How do you decide whether to use a queue or topic in Azure Service Bus? (PDF p. 11)
**Answer:** Use a queue when one logical consumer group should process each message; competing workers can share the load. Use a topic with subscriptions when a message must be independently delivered to multiple consumer groups, optionally filtered per subscription.

###### 17. What is Event Grid in Azure, and when/why do we use it? (PDF p. 11)
**Answer:** Event Grid is a managed event-routing service for discrete notifications about state changes, such as a blob being created. It routes events from publishers to subscribers with filtering and retry/dead-letter options; use it for event notification rather than long-lived work queues.

###### 18. What is Event Hubs? Explain with an example. (PDF p. 11)
**Answer:** Azure Event Hubs is a high-throughput event-ingestion and streaming platform that partitions ordered event streams and retains them for a configurable period. For example, many devices can stream telemetry to Event Hubs for independent consumer groups to process and analyze.

###### 19. What is a data mart? (PDF p. 11)
**Answer:** A data mart is a curated analytical data store focused on a business area or subject, such as sales. It provides structured, governed data for reporting and analysis and may be sourced from a broader data warehouse/lakehouse.

###### 20. How do you implement logging and monitoring in Azure for .NET applications? (PDF p. 11)
**Answer:** Emit structured logs and telemetry through `ILogger` and Application Insights/OpenTelemetry, propagate correlation/trace IDs, and use Azure Monitor workspaces/dashboards/alerts. Avoid logging secrets or unnecessary personal data, and base alerts on user-impacting signals and actionable thresholds.

###### 21. Azure API Management questions (PDF p. 12)
**Answer:** Azure API Management (APIM) is a managed API gateway and developer/API-management platform. It can publish APIs, apply policies for authentication, transformation, quotas/rate limits, and expose documentation; it should complement, not replace, service-level authorization.

###### 22. How do you secure an ASP.NET Core API in Azure? (PDF p. 12)
**Answer:** Authenticate with Microsoft Entra ID/OIDC or another trusted identity provider; enforce authorization policies in the API; use TLS, managed identity, Key Vault, least-privilege network/data access, and APIM/WAF protections as appropriate. Add input validation, patching, audit logs, and monitoring.

###### 23. Explain some AI services in Azure (PDF p. 12)
**Answer:** Examples include Azure AI Foundry services/models for generative AI, Azure AI Services for capabilities such as vision, speech, language, and document processing, and Azure Machine Learning for model development and deployment. Names and capabilities change, so verify the current product lineup and describe a specific use case.

###### 24. Have you used AKS? (PDF p. 12)
**Answer:** Answer truthfully. If yes, describe your role in deploying and operating workloads, including manifests/Helm, ingress, identity/secrets, autoscaling, and observability. If not, say so and explain relevant container/Kubernetes knowledge without overstating hands-on experience.

###### 25. How is Application Insights configured in your ecosystem? Can logs for all applications be viewed in one place, or must you use separate URLs/tabs? How do you configure that? (PDF p. 12)
**Answer:** Instrument each service with Application Insights/OpenTelemetry and route telemetry to a shared or centrally queried Log Analytics workspace, while retaining per-resource access controls and service/environment dimensions. Use Azure Monitor workbooks/dashboards and shared correlation IDs to view cross-service traces; separate resources can still be queried centrally when configured appropriately.

###### 26. How do you configure deployment parameters (number of pods/max pods/min pods) for an App Service Web App/API? Must every app have separate configuration or can they share one? (PDF p. 12)
**Answer:** The prompt mixes Kubernetes pod terminology with App Service, which scales using App Service plan instances and per-app settings, not Kubernetes pods. For AKS, configure replicas and autoscaling (for example, HPA) in deployment/cluster configuration. Each workload needs an explicit scaling policy, though reusable templates/IaC can centrally define common defaults; validate current App Service scaling limits and plan topology.

---

### Azure interview questions

##### Azure Bus *(PDF p.21)*

The PDF contains the heading “Azure interview questions” followed only by “Azure Bus”; no specific question or product is identified. Assuming this refers to **Azure Service Bus**: it is a managed message broker for reliable asynchronous communication. Queues provide competing-consumer point-to-point delivery; topics/subscriptions provide pub/sub. Use durable messages, dead-letter queues, duplicate detection/idempotency where needed, and lock/settlement handling. If “Azure Bus” meant another service, the source does not clarify.
