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
15. [Questpond employer interviews and full PDF question bank](15-questpond-employer-interviews-and-full-pdf.md)

## Completeness and duplicate handling

The source-directory audit found 45 files: all 40 distinct source documents are included in full under their subject guides, and five exact duplicates are documented below. Every Markdown source and all selectable text from each distinct PDF were checked against the compiled guides. PDF inspection found no embedded images; one page at the end of `C# Data Structures Guide.pdf` (and its duplicate `f.pdf`) is blank. Related documents are grouped together; where non-identical sources cover the same concept, their examples and interview angles are retained rather than silently discarded. Headings are shifted down to make the combined guides navigable; source body text and examples are retained. The sources are left unchanged.

These exact duplicates were omitted because their contents match byte-for-byte or, for PDFs, their extracted text matches exactly:

- `23-design-patterns-csharp-senior-viva (1).md` duplicates `23-design-patterns-csharp-senior-viva.md`.
- `f.pdf` duplicates `C# Data Structures Guide.pdf`.
- `NET middleware for senior devs (2).pdf`, `(3).pdf`, and `(4).pdf` duplicate `(1).pdf`.

`NET middleware for senior devs.pdf` is retained as a separate version because its extracted text differs from the numbered copies.

The JavaScript and Angular guides are copied from the dedicated Questpond question files. The React/Next.js guide consolidates React questions found across the Questpond interview PDF and notes, then adds clearly labeled supplementary React and Next.js Q&A sections because coverage of those frameworks in the Questpond materials is limited.

The Azure DevOps, Docker, and Kubernetes guide consolidates deployment, CI/CD, container, and AKS interview material from Questpond notes and the interview PDF.

The Azure cloud-administration guide uses Azure services and project examples named in the H1B CV, then adds administration context. The backend viva file itself contains backend topics, not an inventory of Azure services.

The Questpond folder has also been audited end-to-end. Its complete topic-specific Markdown question sets are included in their matching guides; all employer-specific interview notes and all 70 extracted pages from `RealTimeInterviewQuestions.pdf` are preserved in the Questpond interview archive. `settings.json` is workspace configuration, not interview-preparation material, so it is not included.

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

### [System Design and Distributed Systems](08-system-design-and-distributed-systems.md)

- `B:\TUTS\Questpond\10-scenario-based-interview-questions.md`
- `high-traffic-dotnet-api-blueprint.md`
- `saga-pattern-dotnet.md`
- `async-messaging-dotnet-viva-guide.md`
- `senior_dotnet_async_locking_distributed_guide.md`

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

### [Questpond Employer Interviews and Full PDF Question Bank](15-questpond-employer-interviews-and-full-pdf.md)

All employer-specific Questpond interview notes are preserved in full here, including multi-topic reports already mined for questions in the focused guides. The complete 70-page `RealTimeInterviewQuestions.pdf` is also included as page-organized extracted text.

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
