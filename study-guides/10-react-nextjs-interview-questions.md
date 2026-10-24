# React.js and Next.js Interview Questions

## React.js questions

The first questions below consolidate material found in the Questpond sources. The supplementary section adds core React topics that were not covered there.

### 1. How do you handle exceptions in React?
**Answer:** Use error boundaries to handle errors thrown while rendering, in lifecycle methods, and by constructors of descendant components. They do not catch errors in event handlers or arbitrary asynchronous callbacks, so handle those explicitly with `try`/`catch` and report errors at an appropriate boundary.

### 2. What is an alternative to React `useEffect`?
**Answer:** There is no universal replacement. Derive values during render, use `useMemo` for pure expensive calculations, and handle user actions in event handlers. For server state, prefer framework data-fetching features or a query library; use `useSyncExternalStore` to subscribe to external stores. Use effects when synchronizing with external systems.

### 3. How can you optimize a React application with heavy database interactions?
**Answer:** Reduce unnecessary requests and over-fetching, batch or paginate data as appropriate, and optimize the API/database query and response payload. On the React side, avoid duplicate requests and use a server-state cache with explicit freshness and invalidation rules. Measure the end-to-end bottleneck before and after changes.

### 4. How do you manage caching in a React front end?
**Answer:** Use HTTP caching where appropriate and a server-state cache for request deduplication, freshness, and invalidation. Define how mutations refresh or invalidate cached data, and ensure user-specific or authorization-sensitive responses are never shared across users.

### 5. What is printed by these JavaScript expressions?
```js
console.log(true + 1);
console.log(1 + '2');
console.log('2' + 1);
```
**Answer:** `2`, `"12"`, and `"21"`, respectively. Boolean is converted to a number for numeric addition; when string concatenation is selected, the other operand is converted to a string.

## Supplementary React.js questions

These additional questions are not attributed to Questpond.

### 1. What is a React component?
**Answer:** A component is a reusable unit of UI. Modern React applications commonly define components as JavaScript or TypeScript functions that receive props and return React elements. Components can compose other components to build a UI tree.

### 2. What is JSX?
**Answer:** JSX is a syntax extension that lets UI markup be written alongside JavaScript expressions. A build tool transforms it into React element creation calls. JSX is not HTML: use `className` for CSS classes, close tags, and embed expressions in braces.

### 3. What is the difference between props and state?
**Answer:** Props are inputs passed to a component by its parent and should be treated as read-only. State is data managed by a component or a state owner; updating it schedules a render. Pass callbacks through props when a child needs to request a parent-owned state change.

### 4. What causes a React component to render again?
**Answer:** A component can render when its state changes, when its parent renders, or when a consumed context value changes. A render computes the next UI; React may then commit only the necessary DOM changes. Rendering should remain pure because React may call it more than once.

### 5. Why are `key` props needed when rendering a list?
**Answer:** Keys let React match list items between renders so it can preserve the right component state and efficiently apply insertions, removals, and reordering. Use a stable, unique identifier from the data; array indexes can cause incorrect state preservation when items move or are removed.

### 6. What is the difference between controlled and uncontrolled inputs?
**Answer:** A controlled input gets its value from React state and reports changes through an event handler. An uncontrolled input keeps its current value in the DOM and can be read through a ref or form submission. Controlled inputs are useful when the UI must respond to or validate each change; uncontrolled inputs can simplify cases where the value is only needed on submit.

### 7. What are the Rules of Hooks?
**Answer:** Call Hooks only at the top level of a function component or custom Hook—not inside conditions, loops, or nested functions—and only from React function components or custom Hooks. This keeps Hook call order stable between renders.

### 8. What is a custom Hook?
**Answer:** A custom Hook is a function whose name starts with `use` and that composes built-in or other Hooks to share stateful logic between components. It shares logic, not one shared state instance: each component calling it has its own Hook state unless the Hook connects to a shared store or context.

### 9. What does `useState` do, and why use its functional updater form?
**Answer:** `useState` adds state to a function component and returns the current value and a setter. Use the functional form, such as `setCount(current => current + 1)`, when the next value depends on the previous value; this avoids relying on a potentially stale value captured by a render.

### 10. What is `useEffect` for, and when does its cleanup run?
**Answer:** `useEffect` synchronizes a component with an external system, such as a subscription, timer, browser API, or network connection. React runs cleanup before re-running the Effect when its dependencies change and when the component is removed. Effects are not needed for values that can be calculated during rendering or for logic that belongs directly in an event handler.

### 11. How should Effect dependencies be selected?
**Answer:** Include every reactive value from the component that the Effect reads. The dependency list describes the values the Effect uses; it is not a manual scheduling switch. Restructure the code or extract stable logic when dependencies cause unwanted repetition instead of suppressing dependency checks without understanding the behavior.

### 12. What is the difference between `useMemo` and `useCallback`?
**Answer:** `useMemo` caches the result of a calculation between renders when dependencies are unchanged. `useCallback` caches a function identity. They are performance optimizations, not correctness guarantees; first keep rendering simple, measure, and use memoization when it meaningfully avoids work or stabilizes a dependency.

### 13. What is `React.memo`?
**Answer:** `React.memo` can skip rendering a function component when its props compare equal to the previous render. By default, it compares each prop with `Object.is`. It does not prevent renders caused by the component's own state or consumed context, and is most useful when profiling identifies avoidable expensive renders.

### 14. What is Context, and when should it be used?
**Answer:** Context lets a subtree read a value supplied by an ancestor without passing it through every intermediate component. It is useful for values such as theme, locale, or a stable service. Frequently changing, large application state may be better managed with more focused state boundaries or a dedicated store to limit broad re-renders.

### 15. When should state be lifted up?
**Answer:** Move state to the closest common ancestor of the components that need to read or update it. Keep state as local as possible; share it only where coordination is required. This makes data flow explicit without turning every piece of state into global state.

### 16. What is reconciliation in React?
**Answer:** Reconciliation is React's process for comparing the newly described element tree with the previous one and determining what work is needed to update the UI. Element types and keys help React decide whether to preserve or replace component instances and their state.

### 17. What is the difference between an event handler and an Effect?
**Answer:** An event handler runs in response to a particular user or browser event, such as a click or form submission. An Effect runs after rendering to synchronize with an external system as the component's reactive values change. Put user-triggered actions in handlers; reserve Effects for synchronization.

### 18. What does `useRef` do?
**Answer:** `useRef` returns a stable object whose `.current` value can hold data across renders without causing a render when changed. It can also reference a DOM node. Use state when a changed value should affect rendered output; use refs for imperative handles or non-rendering information such as timer IDs.

### 19. What are portals?
**Answer:** Portals render a React subtree into a DOM node outside its parent DOM hierarchy, which is useful for overlays such as dialogs and tooltips. The content still participates in the React tree, so context remains available and events propagate according to React's tree.

### 20. What are error boundaries, and what errors do they catch?
**Answer:** Error boundaries are components that display fallback UI when a descendant throws during rendering or related React-managed lifecycle work. They do not catch errors in event handlers, server-side rendering, or asynchronous callbacks such as timers; handle those cases explicitly or route errors through an appropriate boundary.

### 21. What is hydration?
**Answer:** Hydration attaches React's client-side behavior and event handling to HTML that was already rendered on the server. The initial client render must be compatible with the server output; mismatches can cause warnings and, depending on the case, client-side recovery or re-rendering.

### 22. What is the difference between client-side rendering and server-side rendering?
**Answer:** With client-side rendering, the browser runs JavaScript to produce the UI. With server-side rendering, the server generates initial HTML for a request and the browser can hydrate it for interaction. Server rendering can improve initial content availability, but requires attention to server data access, per-request state, caching, and hydration consistency.

### 23. What is React Strict Mode?
**Answer:** Strict Mode is a development-only tool that helps reveal unsafe patterns and missing cleanup. In applicable development configurations, React may intentionally re-run render-related logic or Effect setup and cleanup to expose code that is not resilient to repeated execution. It does not enable these checks in production.

### 24. How can React performance problems be investigated?
**Answer:** Reproduce the slow interaction and profile it with React Developer Tools and browser performance tools. Identify whether the cost is unnecessary rendering, expensive calculations, large lists, JavaScript bundle size, network latency, or backend work. Optimize the measured cause—using techniques such as virtualization, code splitting, memoization, or request caching only where they address that cause—and remeasure.

## Next.js questions

The Questpond materials in the supplied folder contain no Next.js-specific interview questions. The following supplementary questions fill that gap and are not attributed to Questpond.

### 1. What is Next.js?
**Answer:** Next.js is a React framework for building web applications. It adds file-system routing, server rendering and static rendering options, server-side capabilities, and production tooling around React.

### 2. What is the difference between the App Router and Pages Router?
**Answer:** The Pages Router uses files under `pages/` and APIs such as `getServerSideProps` and `getStaticProps`. The App Router uses `app/`, nested layouts, Server Components by default, and newer conventions such as route handlers and server actions. Both are supported; choose based on the application and migrate incrementally when needed.

### 3. What are Server Components and Client Components?
**Answer:** Server Components render on the server and can access server-side resources without shipping their component code to the browser. Client Components are required for browser APIs, event handlers, state, and effects; mark their entry point with `"use client"`. Keep client boundaries narrow to reduce browser JavaScript.

### 4. Explain SSR, SSG, ISR, and client-side rendering.
**Answer:** Server-side rendering (SSR) generates a response on the server for a request. Static-site generation (SSG) generates pages ahead of time. Incremental Static Regeneration (ISR) refreshes generated output after a configured revalidation condition or interval. Client-side rendering (CSR) renders or updates the UI in the browser. Select based on freshness, latency, personalization, and caching needs.

### 5. What are layouts and nested routes in the App Router?
**Answer:** Folders define route segments, while a `page` file exposes a route. A `layout` wraps pages and nested segments and can preserve shared UI across navigation. Nested layouts let sections have their own persistent shells; special files can provide loading, error, and not-found UI.

### 6. How does data fetching work in the App Router?
**Answer:** Server Components can call asynchronous data-access functions directly and render the result. Use the framework's current caching and revalidation APIs deliberately; use a Client Component when interaction or browser state is needed. Handle loading and errors at the appropriate route or component boundary.

### 7. What are Route Handlers?
**Answer:** Route Handlers define HTTP endpoints using route files in the `app` directory and standard request/response APIs. They can implement methods such as `GET` and `POST`, but should validate inputs, enforce authorization, and avoid exposing secrets or internal errors.

### 8. What are Server Actions?
**Answer:** Server Actions are server-executed functions that a UI can invoke, often for form submissions and mutations. Treat each action as a public entry point: validate input, authenticate and authorize the caller, and revalidate or update affected data after a successful mutation.

### 9. How do you handle dynamic routes?
**Answer:** Use bracketed route segments such as `[id]` for one dynamic parameter and `[...slug]` for catch-all segments. The route receives the parameter and should validate it before using it to fetch or mutate data.

### 10. How do you protect secrets in a Next.js application?
**Answer:** Keep credentials and privileged operations in server-only code and environment variables; do not expose secrets through variables intended for the browser. Validate and authorize requests on the server, since hiding UI or guarding client-side navigation is not an authorization boundary.

### 11. How does caching affect Next.js applications?
**Answer:** Caching can apply at multiple levels, including rendered output, data requests, and browser/CDN responses. Decide freshness and invalidation behavior explicitly, especially after mutations and for user-specific data. Framework caching defaults can change between versions, so verify the behavior for the version being used.

### 12. What is middleware in Next.js used for?
**Answer:** Middleware runs before a request reaches a route and can perform lightweight tasks such as redirects, rewrites, or request checks. It is not a replacement for authorization inside server-side data access or route handlers, and its runtime constraints should be considered.

### 13. How do you optimize images and metadata?
**Answer:** Use the framework's image component where it fits to support responsive sizing and image optimization, and define page metadata using the supported route metadata APIs. Supply meaningful alternative text, dimensions or sizing hints, and page-specific titles and descriptions.

### 14. How should authentication and authorization be implemented?
**Answer:** Use a trusted authentication approach to establish identity, then check authorization on every protected server action, route handler, and data-access path. Use secure cookie settings where applicable, avoid trusting client-provided roles or identifiers, and ensure cached responses cannot leak data between users.

## Questpond source coverage

- `RealTimeInterviewQuestions.pdf` — React exception handling, `useEffect`, and JavaScript output questions (pp. 52–53).
- `30-virtusa.md` — React question answers and output questions.
- `02-csharp.md` — React front-end performance scenario (PDF p. 6).
- `05-dotnet-core-web-api.md` — React caching question (PDF p. 10).
