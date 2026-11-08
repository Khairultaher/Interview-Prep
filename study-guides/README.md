# Interview Preparation Study Guides

This collection turns scattered interview notes into a practical path from language fundamentals to senior-level architecture and cloud discussions. Each guide explains the ideas, trade-offs, and questions most likely to come up in a real interview.

## Recommended study order

Work through the guides in order, or jump to the area you need to strengthen:

1. [Study roadmap and interview strategy](00-study-roadmap.md) — Start here for a realistic preparation plan, revision rhythm, and advice on presenting your experience clearly.
2. [C# language and code quality](01-csharp-language-and-quality.md) — Reinforce the language fundamentals behind maintainable production code, including collections, generics, memory, and clean coding practices.
3. [Data structures and algorithms](02-data-structures-and-algorithms.md) — Practise the structures, complexity analysis, sorting, searching, heaps, and problem-solving patterns used in coding rounds.
4. [.NET and ASP.NET Core](03-dotnet-and-aspnetcore.md) — Understand the runtime and web framework topics that distinguish a confident .NET developer: DI, middleware, configuration, hosting, and performance.
5. [Web APIs, HTTP, and identity](04-web-apis-http-and-identity.md) — Learn how to design reliable APIs and explain HTTP, REST, authentication, authorization, tokens, and security decisions.
6. [EF Core, LINQ, and SQL Server](05-efcore-linq-and-sql.md) — Prepare for data-access questions with practical coverage of querying, tracking, relationships, transactions, indexes, and SQL performance.
7. [Architecture and design patterns](06-architecture-and-design-patterns.md) — Build a vocabulary for discussing SOLID, layering, common patterns, and the trade-offs behind an architecture.
8. [Concurrency, threading, and async](07-concurrency-threading-and-async.md) — Make asynchronous and concurrent code easier to reason about, from tasks and locks to cancellation and synchronization primitives.
9. [System design and distributed systems](08-system-design-and-distributed-systems.md) — Work through the decisions behind scalable systems: consistency, caching, messaging, resilience, observability, and failure handling.
10. [Microservices, modular monoliths, and distributed systems](09.1-microservices_modular_monolith_interview_tutorial.md) — Compare service boundaries and modular designs so you can explain when complexity is justified and when it is not.
11. [JavaScript interview questions](09-javascript-interview-questions.md) — Review the language and browser concepts that frequently appear alongside backend and full-stack interviews.
12. [React.js and Next.js interview questions](10-react-nextjs-interview-questions.md) — Prepare for component design, rendering, state, data fetching, and the practical trade-offs of modern React applications.
13. [Angular interview questions](11-angular-interview-questions.md) — Cover Angular architecture, components, services, RxJS, forms, routing, and the patterns used in enterprise frontends.
14. [Azure DevOps, Docker, and Kubernetes / AKS](12-azure-devops-docker-kubernetes.md) — Connect application delivery to the platform: containers, pipelines, deployments, orchestration, and operational concerns.
15. [Azure cloud administration for backend developers](13-azure-cloud-administration-for-backend.md) — Get a developer-friendly overview of Azure networking, identity, compute, storage, monitoring, and day-to-day administration.
16. [Questpond employer-specific interview questions](15-questpond-employer-interviews-and-full-pdf.md) — Finish with a broad question bank that connects common employer-style questions to the focused explanations in the other guides.

## How duplicates are handled

**Each concept is explained once.** When several source notes covered the same topic, their unique details were merged into one section, and the repeated explanations were removed. When a topic belongs to another guide, a link points there instead of repeating it. For example:

- **Middleware** (`Use`/`Run`/`Map`, ordering, middleware vs filters) lives in guide 03. The three near-identical ChatGPT middleware transcripts were merged into one "Extra Senior Notes" section.
- **Dependency injection** lives in guide 03. The three DI guides were merged into one section.
- **Collections** (`List`, `Dictionary`, `Stack`, `Queue`, sets) live in guide 01. **Sorting, comparers, heaps, and `PriorityQueue`** live in guide 02.
- **HTTP headers** live in guide 04. Two header guides were merged into one.
- **EF Core** has one main guide in 05. The four overlapping EF Core notes were folded into it, and only the questions not covered there were kept.
- **Async, threads, and locking** (`lock`, `Monitor`, `SemaphoreSlim`, `ConfigureAwait`) live in guide 07. **Distributed jobs, distributed locks, and resilience** live in guide 08.
- **Questpond interview material**: raw page-by-page text is no longer copied. Every question is answered in the employer notes (guide 15) or in the matching topic guide, keeping the content searchable and easier to revise.

Chat transcript labels ("You said", "ChatGPT said") and import labels have been removed. The explanations keep their original simple, friendly tone. The raw sources in `../source/` are unchanged.

## Source coverage

### [Study Roadmap and Interview Strategy](00-study-roadmap.md)

The starting point for planning your preparation, prioritizing topics, and building an effective revision routine.

### [C# Language and Code Quality](01-csharp-language-and-quality.md)

Core C# concepts, collections, generics, and code-quality guidance are consolidated here.

### [Data Structures and Algorithms](02-data-structures-and-algorithms.md)

Interview-ready explanations of common data structures, algorithms, and complexity trade-offs.

### [.NET and ASP.NET Core](03-dotnet-and-aspnetcore.md)

Runtime, dependency injection, middleware, configuration, hosting, and ASP.NET Core fundamentals.

### [Web APIs, HTTP, and Identity](04-web-apis-http-and-identity.md)

API design, HTTP behavior, authentication, authorization, and identity concepts.

### [EF Core, LINQ, and SQL Server](05-efcore-linq-and-sql.md)

Querying, persistence, database design, transactions, and performance-focused data access.

### [Architecture and Design Patterns](06-architecture-and-design-patterns.md)

Design principles and patterns for communicating maintainable architecture choices.


### [Concurrency, Threading, and Async](07-concurrency-threading-and-async.md)

Task parallelism, synchronization primitives, threading, async/await, locking, and `ConfigureAwait` are brought together here for focused revision.

### [System Design and Distributed Systems](08-system-design-and-distributed-systems.md)

Scalability, reliability, messaging, caching, consistency, and operational design.

### [JavaScript Interview Questions](09-javascript-interview-questions.md)

The JavaScript concepts most useful for full-stack and frontend interview discussions.

### [React.js and Next.js Interview Questions](10-react-nextjs-interview-questions.md)

Modern React and Next.js concepts, from components and state to rendering and data fetching.

### [Angular Interview Questions](11-angular-interview-questions.md)

Enterprise Angular concepts including components, services, forms, routing, and RxJS.

### [Azure DevOps, Docker, and Kubernetes / AKS](12-azure-devops-docker-kubernetes.md)

Delivery pipelines, containers, Kubernetes, AKS, and the operational details behind deployments.

### [Azure Cloud Administration for Backend Developers](13-azure-cloud-administration-for-backend.md)

An accessible tour of the Azure services and administration topics backend developers encounter.

### [Questpond Employer-Specific Interview Questions](15-questpond-employer-interviews-and-full-pdf.md)

A broad final question bank that points back to the focused topic guides for deeper revision.
