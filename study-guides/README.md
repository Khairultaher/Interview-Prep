# Interview Preparation Study Guides

A topic-organized library compiled from the Markdown and PDF materials in `../source/`, with additional frontend, Azure DevOps/container, and Azure administration interview material from `B:\TUTS\Questpond` and `B:\TUTS\H1B`.

## Recommended study order

1. [Study roadmap and interview strategy](00-study-roadmap.md)
2. [C# language and code quality](01-csharp-language-and-quality.md)
3. [Data structures and algorithms](02-data-structures-and-algorithms.md)
4. [.NET and ASP.NET Core](03-dotnet-and-aspnetcore.md)
5. [Web APIs, HTTP, and identity](04-web-apis-http-and-identity.md)
6. [EF Core, LINQ, and SQL Server](05-efcore-linq-and-sql.md)
7. [Architecture and design patterns](06-architecture-and-design-patterns.md)
8. [Concurrency, threading, and async](07-concurrency-threading-and-async.md)
9. [System design and distributed systems](08-system-design-and-distributed-systems.md)
10. [JavaScript interview questions](09-javascript-interview-questions.md)
11. [React.js and Next.js interview questions](10-react-nextjs-interview-questions.md)
12. [Angular interview questions](11-angular-interview-questions.md)
13. [Azure DevOps, Docker, and Kubernetes / AKS](12-azure-devops-docker-kubernetes.md)
14. [Azure cloud administration for backend developers](13-azure-cloud-administration-for-backend.md)
15. [Questpond employer-specific interview questions](15-questpond-employer-interviews-and-full-pdf.md)

## How duplicates are handled

**Each concept is explained once.** When several source notes covered the same topic, their unique details were merged into one section, and the repeated explanations were removed. When a topic belongs to another guide, a link points there instead of repeating it. For example:

- **Middleware** (`Use`/`Run`/`Map`, ordering, middleware vs filters) lives in guide 03. The three near-identical ChatGPT middleware transcripts were merged into one "Extra Senior Notes" section.
- **Dependency injection** lives in guide 03. The three DI guides were merged into one section.
- **Collections** (`List`, `Dictionary`, `Stack`, `Queue`, sets) live in guide 01. **Sorting, comparers, heaps, and `PriorityQueue`** live in guide 02.
- **HTTP headers** live in guide 04. Two header guides were merged into one.
- **EF Core** has one main guide in 05. The four overlapping EF Core notes were folded into it, and only the questions not covered there were kept.
- **Async, threads, and locking** (`lock`, `Monitor`, `SemaphoreSlim`, `ConfigureAwait`) live in guide 07. **Distributed jobs, distributed locks, and resilience** live in guide 08.
- **Questpond `RealTimeInterviewQuestions.pdf`**: its raw page-by-page text is no longer copied. Every question in it is answered in the employer notes (guide 15) or in the matching topic guide. The original PDF is still at `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf`.

Chat transcript labels ("You said", "ChatGPT said") and import labels have been removed. The explanations keep their original simple, friendly tone. The raw sources in `../source/` are unchanged.

## Source coverage

### [Study Roadmap and Interview Strategy](00-study-roadmap.md)

- `senior_dotnet_refined_roadmap.md`

### [C# Language and Code Quality](01-csharp-language-and-quality.md)

- `B:\TUTS\Questpond\01-oop-solid.md`
- `B:\TUTS\Questpond\02-csharp.md`
- `modern-csharp-senior-architect-interview-guide.md`
- `csharp_data_types_and_collections_cheatsheet.md`
- `senior_csharp_code_review_refactoring_interview_guide.md`
- `shallow-vs-deep-clone.md`

### [Data Structures and Algorithms](02-data-structures-and-algorithms.md)

- `B:\TUTS\Questpond\10-scenario-based.md`
- `B:\TUTS\Questpond\12-coding-questions.md`
- `csharp-data-structures-sorting-interview-cheatsheet.md`
- `algorithmic_techniques_guide.md`
- `C# Data Structures Guide.pdf`

### [.NET and ASP.NET Core](03-dotnet-and-aspnetcore.md)

- `aspnetcore_middleware_filters_dotnet8_9.md`
- `NET middleware for senior devs (1).pdf`
- `NET middleware for senior devs.pdf`
- `Middleware vs Action Filters.pdf`
- `aspnet_core_performance_checklist_and_mcqs.md`
- `dotnet-dependency-injection-guide.md`
- `dotnet-dependency-injection-senior-architect-guide.md`
- `dotnet-dependency-injection-through-net-10.md`
- `IIS_Kestrel_Senior_DotNet_Viva.md`
- `signalr_dotnet8_viva_guide.md`

### [Web APIs, HTTP, and Identity](04-web-apis-http-and-identity.md)

- `B:\TUTS\Questpond\05-dotnet-core-web-api.md`
- `B:\TUTS\Questpond\11-api-testing.md`
- `csharp-aspnetcore-rest-api-notes.md`
- `dotnet8-rest-api-viva-notes.md`
- `senior_dotnet_interview_guide.md` (HTTP/API MCQs, standards, tests, and scenarios)
- `common-http-headers.md`
- `http-headers-dotnet-viva.md`
- `senior_dotnet_storage_session_jwt_viva.md`

### [EF Core, LINQ, and SQL Server](05-efcore-linq-and-sql.md)

- `B:\TUTS\Questpond\06-orm.md`
- `B:\TUTS\Questpond\08-sql.md`
- `B:\TUTS\Questpond\11-sql-scenarios.md`
- `ef_core_linq_senior_interview_guide.md`
- `EF_Core_Viva_Preparation.md`
- `ef-core-linq-optimization-senior-viva-guide.md`
- `ef-core-performance-interview-and-benchmark-lab.md`
- `senior_dotnet_linq_efcore_viva.md`
- `sql_server_advanced_grouping_and_window_functions.md`

### [Architecture and Design Patterns](06-architecture-and-design-patterns.md)

- `B:\TUTS\Questpond\09-design-pattern-architecture.md`
- `23-design-patterns-csharp-senior-viva.md`
- `clean-architecture-cqrs-vs-mvc-dotnet8.md`
- `senior_dotnet_architect_viva_prep.md`

### [Concurrency, Threading, and Async](07-concurrency-threading-and-async.md)

- `csharp-concurrency-span-stack-queue.md`
- `csharp-task-parallelism-interview-notes.md`
- `dotnet-threading-async-await-senior-viva.md`
- `senior_dotnet_async_locking_distributed_guide.md` (async/locking part)
- `senior_dotnet_architect_viva_prep.md` (async internals and `ConfigureAwait`)

### [System Design and Distributed Systems](08-system-design-and-distributed-systems.md)

- `B:\TUTS\Questpond\10-scenario-based-interview-questions.md`
- `high-traffic-dotnet-api-blueprint.md`
- `saga-pattern-dotnet.md`
- `async-messaging-dotnet-viva-guide.md`
- `senior_dotnet_async_locking_distributed_guide.md` (distributed part; the async/locking part is in guide 07)

### [JavaScript Interview Questions](09-javascript-interview-questions.md)

- `B:\TUTS\Questpond\03-javascript.md`
- JavaScript output questions from `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf` and the Virtusa interview notes

### [React.js and Next.js Interview Questions](10-react-nextjs-interview-questions.md)

- React questions from `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf`
- `B:\TUTS\Questpond\30-virtusa.md`
- React performance question from `B:\TUTS\Questpond\02-csharp.md`
- React caching question from `B:\TUTS\Questpond\05-dotnet-core-web-api.md`
- No Next.js-specific questions were present in the Questpond materials

### [Angular Interview Questions](11-angular-interview-questions.md)

- `B:\TUTS\Questpond\04-angular.md`

### [Azure DevOps, Docker, and Kubernetes / AKS](12-azure-devops-docker-kubernetes.md)

- `B:\TUTS\Questpond\07-azure.md`
- `B:\TUTS\Questpond\20-epam.md`
- `B:\TUTS\Questpond\30-virtusa.md`
- `B:\TUTS\Questpond\30-wolters.md`
- `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf`
- `B:\TUTS\Interview Prep\source\senior_dotnet_architect_viva_prep.md`

### [Azure Cloud Administration for Backend Developers](13-azure-cloud-administration-for-backend.md)

- `B:\TUTS\Questpond\07-azure.md`
- `B:\TUTS\Questpond\13-azure-interview-questions.md`
- `B:\TUTS\H1B\Backend-Development-Viva-Preparation.md` (backend topics used for context; it does not enumerate Azure services)
- Supplementary Azure identity, networking, compute, data, messaging, governance, monitoring, and security service guide

### [Questpond Employer-Specific Interview Questions](15-questpond-employer-interviews-and-full-pdf.md)

All employer-specific Questpond interview notes, with short answers. The raw text of `RealTimeInterviewQuestions.pdf` is not repeated here because its questions are answered in this guide or the topic guides.

- `B:\TUTS\Questpond\14-globallogic-interview-questions.md`
- `B:\TUTS\Questpond\20-accenture-first-round.md`
- `B:\TUTS\Questpond\20-capgemini-azure-lead.md`
- `B:\TUTS\Questpond\20-capgemini.md`
- `B:\TUTS\Questpond\20-carelon.md`
- `B:\TUTS\Questpond\20-cognizant.md`
- `B:\TUTS\Questpond\20-deloitte.md`
- `B:\TUTS\Questpond\20-epam.md`
- `B:\TUTS\Questpond\20-globaldata.md`
- `B:\TUTS\Questpond\20-lti-mindtree.md`
- `B:\TUTS\Questpond\20-mphasis.md`
- `B:\TUTS\Questpond\20-persistent-architect.md`
- `B:\TUTS\Questpond\20-persistent-azure-dotnet.md`
- `B:\TUTS\Questpond\20-verisk.md`
- `B:\TUTS\Questpond\30-fn-lastletteralphabet.md`
- `B:\TUTS\Questpond\30-infosys.md`
- `B:\TUTS\Questpond\30-tcs.md`
- `B:\TUTS\Questpond\30-virtusa.md`
- `B:\TUTS\Questpond\30-wipro.md`
- `B:\TUTS\Questpond\30-wolters.md`
- `B:\TUTS\Questpond\RealTimeInterviewQuestions.pdf`

### Exact duplicates omitted

- `23-design-patterns-csharp-senior-viva (1).md`
- `f.pdf`
- `NET middleware for senior devs (2).pdf`
- `NET middleware for senior devs (3).pdf`
- `NET middleware for senior devs (4).pdf`
