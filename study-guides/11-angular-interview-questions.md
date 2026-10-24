# Angular

## Theory questions

### 1. Why Angular? (PDF p. 8)
**Answer:** Angular provides an integrated TypeScript framework with routing, forms, dependency injection, HTTP, testing support, and tooling. It suits structured applications and teams that benefit from consistent conventions; suitability depends on project needs.

### 2. Which is the latest version going on? (PDF p. 8)
**Answer:** Check Angular's official release/support page at interview time because releases change regularly. State the exact major version you have verified and distinguish it from the version used in your project.

### 3. Difference between JavaScript and Angular (PDF p. 8)
**Answer:** JavaScript is a programming language. Angular is a framework, primarily written in TypeScript, for building web applications; it uses JavaScript/TypeScript and browser APIs rather than replacing the language.

### 4. How do you install Angular? (PDF p. 8)
**Answer:** Install a supported Node.js version, then install the Angular CLI with `npm install -g @angular/cli` if global CLI use is desired. Create a project with `ng new app-name`, enter the directory, and run it with `ng serve`.

### 5. What is AOT? (PDF p. 8)
**Answer:** Ahead-of-Time compilation compiles Angular templates and components during the build rather than in the browser at runtime. It can improve startup and catch template errors earlier; production builds use AOT by default.

### 6. What are decorators? (PDF p. 8)
**Answer:** Decorators attach metadata to classes or members for Angular's compiler and dependency-injection system. Examples include `@Component`, `@Injectable`, `@Input`, and `@Directive`.

### 7. How do you create a component? (PDF p. 8)
**Answer:** Use `ng generate component component-name` (or `ng g c component-name`), then configure its template, styles, and class. In modern Angular, components are standalone by default in new projects and can be imported directly where needed.

### 8. How does an Angular app start? (PDF p. 8)
**Answer:** The browser loads the generated entry HTML and JavaScript bundle. The bootstrap entry point (for example, `bootstrapApplication(AppComponent, ...)` in modern standalone apps, or `bootstrapModule(AppModule)` in module-based apps) creates the root component and renders it.

### 9. `package.json` vs. `package-lock.json` (PDF p. 8)
**Answer:** `package.json` declares project metadata, scripts, and dependency ranges. `package-lock.json` records the resolved dependency tree and versions so installs can be reproducible; commit both and use `npm ci` in CI.

### 10. How do you download or install other packages? (PDF p. 8)
**Answer:** Run `npm install package-name` to add a dependency (or `npm install -D package-name` for a development dependency). This updates the manifest and lockfile; use `npm ci` for reproducible clean installs.

### 11. What is `angular.json`? (PDF p. 8)
**Answer:** It is the Angular CLI workspace configuration, describing projects, build/serve/test targets, assets, styles, and configuration options.

### 12. What are modules? (PDF p. 8)
**Answer:** An NgModule groups declarations, imports, providers, and bootstrap configuration in module-based Angular applications. Standalone components/directives/pipes reduce the need for NgModules in modern Angular, though NgModules remain supported.

### 13. What is `ngModel`? (PDF p. 8)
**Answer:** `ngModel` is a FormsModule directive used for two-way binding and form state in template-driven forms, commonly written `[(ngModel)]="name"`. Import `FormsModule`; use reactive forms for more explicit, scalable form models.

### 14. What is routing and how is it implemented? (PDF p. 8)
**Answer:** Routing maps URLs to views/components without a full-page navigation. Configure route definitions with `provideRouter(routes)` (or `RouterModule` in module-based apps), place a `router-outlet`, and navigate using links or the Router service.

### 15. Lazy loading in routing (PDF p. 8)
**Answer:** Lazy loading defers downloading a route's code until needed, reducing initial bundle work. Configure routes with `loadComponent` or `loadChildren`, and split features according to usage and bundle measurements.

### 16. What are directives and their types? Explain with real examples (PDF p. 8)
**Answer:** Directives add behavior to elements. Components are directives with templates; structural directives control rendered structure (for example, `@if`/`@for`, or legacy `*ngIf`/`*ngFor`); attribute directives affect appearance/behavior (for example, `ngClass` or a custom highlight directive).

### 17. How do you create custom directives? (PDF p. 8)
**Answer:** Generate one with `ng generate directive`, then define a directive class with `@Directive({selector: ...})` and inject the host element or use Angular host bindings/listeners. For structural changes, use the appropriate template APIs; prefer host bindings for safe attribute behavior.

### 18. Types of data binding in Angular with examples (PDF p. 8)
**Answer:** Interpolation displays text (`{{title}}`); property binding sets a DOM/component property (`[disabled]="busy"`); event binding handles events (`(click)="save()"`); two-way binding combines property and event binding (`[(ngModel)]="name"`).

### 19. What is string interpolation? (PDF p. 8)
**Answer:** Interpolation embeds a template expression in text using `{{ expression }}`, such as `{{ user.name }}`. Angular evaluates it and updates the text when the value changes.

### 20. Difference between string interpolation and property binding (PDF p. 8)
**Answer:** Interpolation writes an expression's string representation into text content. Property binding assigns a value to a DOM or component property, preserving non-string values such as booleans; use it for attributes/inputs rather than textual display.

### 21. What is an HTTP interceptor? How do you implement it? (PDF p. 8)
**Answer:** An interceptor observes or transforms outgoing HTTP requests and incoming responses, often for authentication headers, logging, or centralized error handling. Implement `HttpInterceptorFn` (modern functional style) or `HttpInterceptor`, register it with `provideHttpClient(withInterceptors([...]))` or the module-based provider configuration, and clone immutable requests when changing them.

### 22. What are services in Angular? (PDF p. 8)
**Answer:** Services are injectable classes that encapsulate reusable business logic, data access, or shared state outside the view. Mark one with `@Injectable` and provide it at the appropriate scope, commonly `providedIn: 'root'`.

### 23. Dependency injection in Angular (PDF p. 8)
**Answer:** Angular's injector creates and supplies dependencies declared by a class, typically through constructor parameters or `inject()`. Providers define how tokens map to values/classes and their scope; this improves reuse, configurability, and testability.

### 24. Ways to pass data from one component to another (PDF p. 8)
**Answer:** Use input/output bindings for parent-child communication, shared services with observable or signal state for unrelated components, router parameters/state for navigation, and a state-management library for broad application state. Avoid direct coupling to another component's internals.

### 25. List lifecycle hooks in sequence with examples (PDF p. 8)
**Answer:** For a component, the usual sequence is constructor, `ngOnChanges` (when inputs change, including initial inputs), `ngOnInit`, `ngDoCheck`, content hooks (`ngAfterContentInit`, `ngAfterContentChecked`), view hooks (`ngAfterViewInit`, `ngAfterViewChecked`), and `ngOnDestroy`; `afterNextRender`/`afterEveryRender` are newer render callbacks, not class lifecycle hooks. Use `ngOnInit` for initialization and `ngOnDestroy` for cleanup; checked hooks run frequently.

### 26. Constructor vs. `ngOnInit` (PDF p. 8)
**Answer:** The constructor is called by JavaScript when the class is instantiated and is best for dependency injection and simple field setup. `ngOnInit` runs once after Angular has initialized inputs and is appropriate for component initialization that depends on them.

### 27. `ngOnChanges` vs. `ngDoCheck` (PDF p. 8)
**Answer:** `ngOnChanges` runs when Angular detects changes to input-bound values and provides previous/current values. `ngDoCheck` runs during change detection to support custom checks; it runs often and should be used sparingly.

### 28. What is `ViewChild`? (PDF p. 8)
**Answer:** `@ViewChild` or the `viewChild()` query API obtains a reference to a child component, directive, or element in the component's view. Access timing depends on query configuration; prefer input/output contracts over manipulating child internals.

### 29. How do you create API calls in Angular? (PDF p. 8)
**Answer:** Inject `HttpClient` into a service, call methods such as `get`/`post`, and consume the returned `Observable` (for example, with `async` pipe or `firstValueFrom` where appropriate). Configure `provideHttpClient()` or import `HttpClientModule` in older module-based apps.

### 30. What are HTTP filters in Angular? (PDF p. 8)
**Answer:** Angular has no built-in feature formally called “HTTP filters” equivalent to ASP.NET filters. The likely intended concept is HTTP interceptors, which can cross-cut requests/responses for headers, logging, and errors; server-side filtering belongs in API query parameters.

### 31. How do you add HTTP headers or authorization in a POST call? (PDF p. 8)
**Answer:** Pass an `HttpHeaders` object in the request options, or add the authorization header in an interceptor. Example: `http.post(url, body, { headers: new HttpHeaders({ Authorization: \`Bearer ${token}\` }) })`; never hard-code secrets in a frontend bundle.

### 32. What are pipes and their types? (PDF p. 8)
**Answer:** Pipes transform values for display in templates. Angular provides built-ins such as `date`, `currency`, `number`, `percent`, `json`, `async`, and `slice`; pipes can be pure (default) or impure, with impure pipes requiring care because they run frequently.

### 33. How do you create pipes? (PDF p. 8)
**Answer:** Generate with `ng generate pipe`, implement `PipeTransform.transform`, and declare/use the pipe (or make it standalone and import it). Keep transformations deterministic and inexpensive; do not use a pipe for side effects.

### 34. Promise vs. Observable (PDF p. 8)
**Answer:** A Promise represents one eventual success/failure and begins its work when created. An Observable can emit zero or more values, can be lazy and cancellable by unsubscribing, and supports composition with RxJS operators.

### 35. Observable vs. Subject (PDF p. 8)
**Answer:** An Observable is a consumer-facing stream that subscribers observe. A Subject is both an Observable and an Observer, so it can receive values and multicast them; expose it as an Observable when consumers should not publish values.

### 36. Different types of Subject (PDF p. 8)
**Answer:** `Subject` has no retained initial/latest value; `BehaviorSubject` requires an initial value and immediately emits the current one to new subscribers; `ReplaySubject` replays a configured buffer; `AsyncSubject` emits its final value when completed.

### 37. Can asynchronous programming be done in Angular? How? (PDF p. 8)
**Answer:** Yes. Use Promises or, more commonly for HTTP and event streams, RxJS Observables; compose work with operators and let Angular's change-detection integration update the view.

### 38. How do you implement asynchronous programming in Angular? (PDF p. 8)
**Answer:** Return Observables from services, compose them with operators such as `switchMap` and `catchError`, and subscribe via the `async` pipe or manage subscriptions with lifecycle-aware cleanup. Use `async`/`await` for one-shot Promise-based APIs when it makes the flow clearer.

### 39. Template-driven forms vs. reactive forms (PDF p. 8)
**Answer:** Template-driven forms describe controls and validation mostly in the template and suit simple forms. Reactive forms build an explicit model in TypeScript, making complex validation, dynamic controls, and unit testing easier; both support validation and submission.

### 40. What is an auth guard in Angular? (PDF p. 8)
**Answer:** A route guard determines whether navigation is allowed, redirected, or deferred based on application logic such as authentication state. It improves client-side navigation behavior but is not a security boundary; the server must authorize every protected request.

### 41. Types of auth guards in Angular (PDF p. 8)
**Answer:** Angular route guards include `CanActivate`, `CanActivateChild`, `CanDeactivate`, `CanMatch`, and (in older APIs) `CanLoad`. They address entering routes/children, leaving a route, matching a route, and loading lazy features; prefer current functional guard APIs where appropriate.

### 42. What is local storage? (PDF p. 9)
**Answer:** `localStorage` is browser key/value storage that persists across browser sessions for an origin. It is synchronous and accessible to JavaScript, so do not store sensitive credentials or secrets there; consider XSS risk and server-managed secure cookies for session tokens.

### 43. Have you used RxJS? Why is it used? (PDF p. 9)
**Answer:** RxJS provides Observables and operators for composing asynchronous streams such as HTTP, user input, and events. A good interview response should describe a real use, such as debouncing a search input and cancelling stale requests with `switchMap`.

### 44. Give some RxJS operators (PDF p. 9)
**Answer:** Common operators include transformation (`map`, `switchMap`, `mergeMap`), filtering (`filter`, `take`, `debounceTime`, `distinctUntilChanged`), combination (`combineLatest`, `forkJoin`), and error/lifecycle handling (`catchError`, `retry`, `takeUntil`).

### 45. What is the `map` operator? (PDF p. 9)
**Answer:** `map` transforms each emitted value into another value while preserving the stream structure. It is for synchronous value transformation; use a flattening operator such as `switchMap` when the transformation returns an Observable.

### 46. What is a template reference variable? (PDF p. 9)
**Answer:** A variable declared with `#name` in a template references an element, directive, or component instance. It can be used elsewhere in that template, for example to read an input's value or call a component's exposed API.

### 47. How do you do testing in Angular? (PDF p. 9)
**Answer:** Unit-test components, services, pipes, and directives with Angular's test utilities and the project's configured runner; mock dependencies and test behavior rather than implementation details. Add integration/end-to-end tests for key user flows and run the suite in CI.

### 48. State management in Angular (PDF p. 9)
**Answer:** Keep local state in the component when possible; share focused state through services using signals or RxJS. For complex cross-feature state and effects, use a structured library such as NgRx or an equivalent, with clear ownership and immutable updates.

### 49. Angular new features (PDF p. 9)
**Answer:** Answer against the current official Angular release notes, since features vary by version. Recent Angular generations have emphasized standalone APIs, signals, functional providers/guards/interceptors, built-in template control flow, and improved build/rendering tools; identify only those supported by the version in question.
