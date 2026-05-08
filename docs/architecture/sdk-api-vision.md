# X9Ware SDK API — Design Vision

X9Ware LLC • May 8, 2026

## Status

**DRAFT — under active design.** This document is a vision, not a charter. It articulates the destination and the principles that lead the design there, and explicitly does not pin specific class signatures, method names, or implementation sequencing.

## Purpose and scope

This document is a **vision** for what the X9Ware SDK API should look like in 2026 and the principles that govern the design choices to get there. It is one level above a charter: a charter locks rules and conventions for a specific dimension of the work, and one or more charters follow naturally from this vision.

### What this document covers

- The strategic position of the modern SDK API alongside the legacy `x9Sdk`, including the dual-API model that lets both coexist indefinitely
- The dimensions that define a contemporary Java SDK in 2026, beyond fluent grammar
- The principles that govern design choices throughout the work
- The customer-facing posture and the layered access pattern that follows from it
- A worked simplest example as the acceptance test for whether the principles are honored
- The business case for why this work delivers customer and revenue value
- The relationship to Lowell's existing fluent API charter, recorded in `x9Sdk/docs/development/fluent-api/`

### What this document does NOT cover

- Specific class names, interface signatures, or method names for the new API surface — those depend on a substrate review of `x9SysTools`, `x9Sdk`, and the existing engines, and live in companion documents authored after that review.
- Implementation sequencing, release timelines, or staging — owned by an implementation plan that follows the substrate review.
- Detailed conventions for individual Engines (terminal verb, factory shape, etc.) — those live in the existing fluent API charter and any future charters this vision spawns.
- Concrete code beyond the worked simplest example used to demonstrate the principles.

The vision sits one level above charters and specs. A reader looking for specifics on a particular Engine, class, or method should find them in the companion documents the vision points to, not here. The deliberate restraint at this level is what lets the principles stay general enough to govern decisions across the whole API, rather than being tied to one engine's particulars.

## Position in one paragraph

The fluent API charter that ships with `x9Sdk` establishes that every operation in the modernized SDK uses the same fluent grammar — a real consistency win the API has been missing. This document addresses the complementary dimensions that 2026 Java SDK customers also expect: an interface-based public surface, JavaBean data shapes, observability seams, modern threading patterns, and ecosystem-friendliness. The two pieces of work compose. Fluent grammar without the surface modernization reads as polish over the existing surface; surface modernization without fluent grammar reads as a different SDK rather than a coherent one. Together they describe what the modernized X9Ware SDK looks like in 2026, and together they support the dual-API position that legacy `x9Sdk` and the modernized API coexist indefinitely.

## Strategic context

### Where the SDK is today

The X9Ware SDK has carried a decade and a half of organic growth across four payment-file dialects (X9.37, ACH, ISO 20022, CPA 005). The substrate is sound and battle-tested: `x9SdkBase`, `x9SdkIO`, the dialect readers and writers, the validation engines, the trailer managers, the rules infrastructure. Existing customers — overwhelmingly X9.37 commercial users — run the SDK in production today and would be poorly served by any change that broke their integrations.

The recent Engine rename (commit `4bc29141`, "rename 38 SDK core classes to the Engine convention") established a uniform naming convention across the verb classes and added a base-class chokepoint where dialect feature gating, configuration validation, and lifecycle hooks can be enforced consistently for the first time. The work to ship that rename was less expensive than initial estimates suggested, which is itself a useful data point: structural cleanup in the substrate is cheaper than the team has historically assumed.

The fluent API charter, dated April 28, 2026, builds on the Engine rename by committing the conventions every fluent Engine will follow: single terminal verb, dual-overload Engine factory, application-managed logging by default, source/sink as builder methods, dialect dispatch via static factory per dialect, and an escape hatch via the `legacy()` accessor. Those conventions are sound within the scope they address.

### Customer base reality

The customer base picture is unchanged from what the charter's parent design document records. X9.37 carries the entire commercial weight of the SDK today. ACH has approximately two greenfield customers. CPA 005 has zero. ISO 20022 has zero production customers. Any decision about API surface is in practice still a decision that affects X9.37 customers most directly.

This reality cuts in two directions for the modernization conversation. On one hand, no current X9.37 customer has filed a support ticket whose resolution depends on the modern API existing. The fluent grammar will not solve any specific past complaint. On the other hand, the absence of paying customers in ACH, ISO 20022, and CPA 005 is itself the strongest argument for modernization: these are precisely the segments where prospects evaluate the SDK against modern alternatives, and where a contemporary API surface is competitive maintenance, not surface ergonomics theater.

The modernization work therefore is a forward-looking investment in customer acquisition, not a fix for an existing customer-reported problem. Naming that explicitly is what protects the work against the legitimate question "is anyone asking for this?" The honest answer is: no current X9.37 customer is asking for it; ACH and ISO 20022 prospects evaluating the SDK in 2026 are.

### What 2026 customers expect

The 2026 enterprise Java landscape that prospects evaluate the SDK against has converged on a recognizable set of patterns. A new customer evaluating a financial messaging SDK in 2026 has internalized expectations from Spring Boot, Hibernate, Jackson, Apache Commons, and the standard library itself. Among those expectations, the ones most relevant to a vendor SDK:

The public surface a customer interacts with is **interface-based**. Concretes are package-private; the customer's code declares its dependencies in terms of interfaces, which makes mocking, alternative implementations, and refactoring across boundaries practical. A customer who cannot write a unit test for code that uses the SDK because the SDK insists on concrete-class collaborators is a customer who is going to find another SDK.

Data types follow **JavaBean conventions**: getters and setters on encapsulated fields, predictable equals and hashCode, useful `toString()`. This shape is what every introspection-based tool in the Java ecosystem expects — DI containers, JSON serializers, validation frameworks, reflection-driven mappers, ORMs. An SDK whose data types use public fields, or whose accessor naming is inconsistent, fights the ecosystem.

Initialization is **constructor-based and explicit**. Mature libraries have moved away from static state, thread-locals, and side-effect-laden bootstrap in favor of objects that take their dependencies as constructor arguments and document their thread-safety contract per type. This is what makes Spring Boot integration a one-line `@Bean` definition rather than a multi-step setup.

Observability is **first-class, but pluggable**. The de facto Java metric standard is Micrometer; for tracing it is OpenTelemetry. Both follow the SLF4J pattern of "API only — customer brings the binding." A 2026 SDK that does not expose Micrometer-shaped meter hooks and OpenTelemetry-shaped observation seams looks dated against its competitors, even if its core functionality is excellent.

Threading is **documented explicitly per type**, with patterns that compose cleanly with virtual threads when the JDK floor permits. Customers have stopped accepting "this might or might not be thread-safe" as a contract.

The runtime is **container-friendly without depending on any specific container**. Customers expect to drop the SDK into Spring Boot, into a microservice container, into AWS Lambda, into a long-running batch job, and have it work — not because the SDK has Spring code in it, but because it does not get in the way of how those environments expect Java libraries to behave.

### Why a vision now

The Engine rename is complete. The fluent charter is published. The substrate is stable. This is the moment to step back from the per-Engine details and ask the broader question: across all the dimensions a 2026 Java SDK is judged on, which has the work addressed, and which remain to be addressed? A vision document at this moment captures the answer durably, gives Lowell's charter a hierarchy to sit inside, and gives any future charters or specs a screen to be measured against.

## What the charter contributes

Several decisions in Lowell's charter are sound, durable, and stay unchanged in this vision. Naming them explicitly anchors the work — the modernization is *building on* the charter, not replacing it.

The **dual-API model** is the right strategic shape. Legacy `x9Sdk` continues unchanged, modern API ships alongside, neither replaces the other, and customers choose. The IBM CICS macro/command analogy the charter cites is recognizable industry precedent, defends against the migration-pressure trap that kills most modernization projects, and gives existing X9.37 customers a guarantee that nothing under them is moving. Without this, modernization becomes a forced upgrade, and forced upgrades to a banking SDK are how customers leave.

`X9SdkApplication` **as the central abstraction** is the highest-leverage single change in the modernization. The 1:N application-and-base model that x9SdkApplication formalizes is not speculative — `x9UtilBatch` has been an unnamed `x9SdkApplication` for over a decade across approximately fifty utility classes. Lifting that pattern into a customer-accessible class collapses the seventy-line invariant preamble that opens every example program today into a fluent block, and incidentally makes Spring Boot bean lifecycle and container-managed lifecycle natural rather than retrofit. This vision adopts the same shape and the same lifecycle ownership rules.

The **engine-as-noun** architecture is the right organizing principle. Verbs are anchored on Engine classes; new engines compose into the same architecture rather than each one being a new thing to learn. The Engine rename pays structural dividends past consistent naming, demonstrated by `commit b397c8c7` adding cross-engine dialect feature gating in a single base class for the first time. This vision treats Engines as the architectural foundation underneath the public API surface — clients do not necessarily see Engine types directly, but the work the engines do is the work the customer-facing API exposes.

The **fluent grammar conventions** the charter locks — single terminal verb, dual-overload factory, source/sink as builder methods, dialect dispatch via static factory per dialect — are sound and apply to operations of the kind the charter targets. This vision keeps them. Where this vision goes further is not in changing those conventions, but in extending the modernization to dimensions the charter does not address.

The **escape hatch via `legacy()`** is unusually thoughtful. A single named door returning a holder with `sdkBase()`, `sdkIO()`, and `worker()` is the right shape for letting power users reach into the existing substrate without leaving the modern path entirely. The lifecycle rule it carries — "you brought it, you close it" — is the only rule that scales across the migration cases. This vision keeps the escape hatch. The complementary question this vision raises is what the *primary* surface should look like, so that customers reach for `legacy()` only at genuine boundary cases rather than as a routine response to common needs.

**Application-managed logging by default** is the right call. The SDK as good library citizen — never configures SLF4J, never bundles a binding, emits log statements via SLF4J API only — aligns the SDK with how Spring, Hibernate, Jackson, and Apache Commons all behave. Customers integrating into modern container scenarios never have to remember to opt out of unwanted logging configuration. This vision keeps that default and extends the same posture to other observability concerns.

## Complementary dimensions to fluent grammar

The fluent grammar the charter establishes is one stylistic dimension of API modernization. A modernized SDK in 2026 is judged across several other dimensions, and each one has a complementary contribution to make. The dimensions named here are not in tension with the charter — they sit alongside it, addressing concerns the charter does not engage with.

### Interface-based public surface

Every type the customer's code touches should be an interface. Concretes are package-private; the customer's code names interfaces in its declarations, parameters, and field types. This shape is what makes mocking practical for customer unit tests, what makes alternative implementations possible for testing or for specialized deployments, and what makes refactoring inside the SDK safe across an interface contract. It is also what allows the SDK to expose multiple implementations of the same concept — synchronous and asynchronous, eager and lazy, in-memory and persistent — without forcing the customer to rewrite their code at the call site.

The current substrate uses concrete classes throughout the public surface. `x9SdkBase`, `x9SdkIO`, the dialect readers and writers, the validation engines — all concretes that customers reference directly. This was a defensible choice when the SDK was first written; it is no longer the default expectation in 2026 Java. The charter does not engage with this dimension; the modernization does.

The implication for our work: every customer-facing type the modern API introduces is an interface. The internal facades that wrap the existing substrate are package-private implementations of those interfaces. The legacy substrate is reachable through the `legacy()` escape hatch when customers genuinely need it, but the primary surface is interface-only.

### JavaBean conventions for data types

Data types — items, headers, trailers, money, errors, validation reports, write summaries — should follow JavaBean conventions. Encapsulated fields, getter/setter accessors with predictable naming, `equals`/`hashCode` consistent with the type's value semantics, useful `toString()`. The Java ecosystem's introspection-based tooling (DI containers, JSON serializers, validation frameworks, ORMs, configuration mappers) all rely on these conventions. A type that breaks them — public fields, inconsistent accessor naming, mutable state without setters — fights every framework the customer might wire it into.

Some current substrate types follow JavaBean conventions; others mix conventions or expose public mutable fields. The latter pattern is direct evidence that the substrate predates the conventions becoming universal expectations. The charter does not engage with this dimension; the modernization does.

The implication for our work: every public data type follows JavaBean conventions throughout. Mutation is through setters, never through direct field assignment. Setters update observable state immediately, consistent with JavaBean contract — there is no deferred-mutation pattern where the customer has to call a separate "commit" method. Internal byte-level representations are an implementation detail the customer never sees.

### Observability seams

A modernized 2026 Java SDK exposes observability hooks — typed integration points where customers wire metrics, tracing, structured logging, and similar instrumentation. The de facto Java standards are Micrometer for metrics, OpenTelemetry for tracing, and SLF4J for logging. All three follow the same pattern: an API the SDK consumes, a binding the customer brings. This is the well-behaved-library posture the charter already commits to for SLF4J; observability extends that posture to metrics and tracing.

The charter does not engage with metrics or tracing, only with SLF4J logging. The SDK today has no observability surface. A customer running the SDK in a Spring Boot service, a microservice, or a containerized batch job has no built-in way to see operation timings, error rates, throughput, or per-operation traces. They have to instrument the SDK from outside, which means inferring boundaries the SDK could expose internally.

The implication for our work: the modern API exposes pluggable observability seams in the shape of `MeterRegistry`-style hooks for metrics and `Observation`-style seams for tracing. Default implementations are no-op, so customers who do not opt in pay no runtime cost. Customers who do opt in wire Micrometer or OpenTelemetry in a few lines, and the SDK's operations show up in their existing observability stack without further instrumentation.

### Modern threading patterns

The charter records the right per-type thread-safety contract: `X9SdkApplication` is thread-safe and shared across threads; `X9SdkBase` is per-task and thread-confined; Engines are stateful builders configured and run on a single thread. That contract is sound and this vision keeps it. The dimension to extend is composition with the threading patterns customers will be using in 2026.

Java 21 introduced virtual threads as a stable feature. Java 25 (the current LTS at the time of writing) includes structured concurrency. Customers building services on these features expect the libraries they integrate to compose cleanly with virtual threads — meaning the libraries do not pin OS-thread-specific resources, do not spawn their own thread pools without a configurable executor, and do not block in ways that prevent virtual-thread scheduling from doing its job. Even when an SDK targets a Java 8 floor (as ours does), the architectural decisions the SDK makes today should not preclude virtual-thread compatibility for customers running on newer JVMs.

The current substrate uses ad hoc threading in places. Some operations spawn their own threads or thread pools; others document thread-confinement; the patterns are not uniformly applied. The charter does not engage with virtual-thread compatibility specifically.

The implication for our work: any thread-pool a modern API operation creates internally accepts a customer-supplied `Executor` as an override. Daemon threads are the default for any pool the SDK manages. Operations document their blocking behavior explicitly so customers running on virtual threads can plan accordingly. The thread-confinement contract the charter records is preserved for state container types but extended to make sure the *patterns* are compatible with modern JVM features.

### Ecosystem friendliness without ecosystem dependencies

The Java application ecosystem in 2026 is centered on Spring Boot for application frameworks, Spring Cloud for distributed-system patterns, and a constellation of containerization platforms (Kubernetes, Docker, OCI runtimes generally). A 2026 SDK does not have to depend on Spring Boot, but it does have to *compose with* Spring Boot — meaning a customer can register the SDK as a Spring bean without writing adapter glue, can inject SDK types into their service code, can rely on the SDK's lifecycle to align with Spring's `@PreDestroy` or `DisposableBean` machinery.

The charter mentions Spring scenarios in its example sketches but does not commit the design choices that make Spring integration friction-free. The `X9SdkApplication` shape (AutoCloseable, no static state, constructor-based, thread-safe for sharing) gets us most of the way; the remaining work is naming the integration points explicitly and validating them against a reference Spring Boot integration.

The implication for our work: the modern API is designed so a Spring Boot user can register `X9SdkApplication` as a singleton bean with `@Bean(destroyMethod = "close")`, inject it into their `@Service` classes, and call SDK operations with no adapter layer. Container-managed lifecycle works. Configuration externalization (system properties, environment variables, externalized files) follows 12-factor conventions so customers in containerized environments can supply license keys, configuration overrides, and credentials through the mechanisms their deployment platforms already provide.

We do not take a Spring or Lombok dependency. We do not bundle Micrometer or OpenTelemetry. The principle is "friendly to, not dependent on" — the SDK works in any of those environments because its public shape is compatible, not because it carries weight from any of them.

## Principles

Three principles govern the design choices in the modern API surface. Each one is a screen — a yes/no question we ask of every API decision. A choice has to clear all three to enter the design.

### First-impression code is a budget, not an outcome

The simplest, complete, runnable, customer-facing example of using the modern API must fit on a slide. Every line in it is being scrutinized by a prospect deciding in five to ten minutes whether to read further. The line count of that example is a budget, not an outcome.

This principle promotes "the simplest example" from a test result to a design constraint. If the budget is, say, twenty lines and we keep blowing it, that is a signal an abstraction needs to absorb more — not that the example should be written more cleverly while the API stays as-is. The pressure runs from the customer's first impression back into the API design.

The competitive reality this principle reflects: modern developers evaluating an SDK make their decision quickly. Twenty lines of clean, readable customer code is a quickstart; fifty lines is a read-through; eighty lines is a barrier. Every multiplier of size is a multiplier of evaluation attrition. An API that requires sixty lines of client setup before any business work begins is one whose business case for adoption is weaker than its technical merits suggest.

### Layered access: implicit defaults, explicit overrides, full descent when needed

Every customer-facing decision point in the modern API has three forms.

The **implicit form** works without configuration. The 95% case uses this. Convention beats configuration: license loaded from a conventional location, dialect inferred from input, format defaulted appropriately, threading invisible, cleanup automatic.

The **override form** is a single knob — one builder method or method parameter — that changes one decision. The 4% case uses this. The override form does not require the customer to learn the underlying machinery; they name the change they want and the SDK does the rest.

The **replace form** drops to the underlying engines or the legacy SDK for full control. The 1% case uses this. The escape hatch is the path; `engine.legacy()` (per the charter) is the door; the customer who reaches for it has signaled they need full control and the SDK gets out of their way.

This three-form pattern is not novel — Spring Boot's auto-configuration, AWS SDK's defaults-with-overrides, and most modern Java libraries follow it. What this principle does is make it a discipline rather than a happy accident. Every design decision states all three forms explicitly.

### Customer-facing posture: surface area matches the work being done

The default surface area a customer interacts with should match the work they are actually trying to do. If the customer wants to read an x9.37 file, validate it, modify a record, and write it back, the API surface they touch should be limited to the verbs of that work — not the lifecycle of the SDK itself, not the threading model, not the configuration loading mechanism, not the engine machinery underneath.

This is a sharper version of the layered-access principle, focused specifically on what the customer's first lines of code look like. The SDK's internal mechanisms (engines, thread pools, configuration files, error managers, trailer managers) all exist; they just do not have to *appear* at the customer's first call site. They appear when the customer reaches for them through an override or an escape hatch — not before.

The principle protects against a specific failure mode: a well-engineered API that is structurally correct but ergonomically heavy, where every internal mechanism shows up as a named type in the customer's code because the SDK author wanted to be transparent. Transparency at the wrong layer is its own form of friction. The customer does not need to know about thread pools to write a check-processing service; if they did, the SDK is making them carry weight that should be carried by the SDK.

## What this means for the public API surface

The principles applied to a public API surface produce a recognizable shape. This section describes that shape at a high level, without pinning specific class signatures or method names — those depend on the substrate review and live in the companion specs.

The customer's entry point is a single typed handle that owns process-scope lifecycle (license, configuration, dialect bindings) and is `AutoCloseable`. It is constructed from a license source (file, classpath resource, environment variable, or inline string) and yields domain operations through method calls on the handle itself. The handle is the analog of Spring's `ApplicationContext` — once per application, created at bootstrap, shared across threads.

Domain types representing payment-file content (files, items, bundles, trailers, errors, money, validation reports) are interfaces. Concrete implementations are package-private and obtained only through the entry point or through other domain operations. Interfaces follow JavaBean conventions for getters and setters; mutation is immediate and observable through the next getter call. Containers (lists of items, lists of errors) are typed iterables, not raw `List<X>`, so domain logic on the container has a place to live.

Dialect typing is explicit at the call sites where it matters and inferred where the customer's intent is unambiguous. A customer whose entire application is X9.37 declares the dialect once, and subsequent calls infer it. A customer who handles multiple dialects names the dialect per call. Both shapes coexist; neither is privileged.

Operations (read, validate, modify, write, scrub, clone, merge, etc.) are services obtained from the entry point or from a contextual handle. They take their inputs as method parameters and return typed result objects. The fluent grammar conventions the charter establishes apply to these operation services without modification.

Observability seams are exposed as typed integration points on the entry point. A customer wiring Micrometer registers a `MeterRegistry`; a customer wiring OpenTelemetry registers an `Observation`-style hook; a customer wiring neither pays no runtime cost. The seams are interface-based, so the customer can supply test doubles in their own tests.

Threading is documented per type and the patterns compose with virtual threads. Any executor the SDK creates internally accepts a customer override. Operations document their blocking behavior. Resource cleanup is invisible by default but accessible via try-with-resources for customers who want deterministic timing.

Customer interaction with the underlying engines and the legacy `x9Sdk` is through the escape hatch the charter defines. Day-to-day work does not require any reference to engine types or to legacy types; the path exists for the genuine power-user case.

## Worked simplest example

The acceptance test for whether the principles are honored is a single readable customer example. The legacy `X9VerifyX9` example program is approximately 760 lines and demonstrates open / load / parse / list / validate / modify / write across an x9.37 file. The same six verbs in the modern API:

```java
public static void main(String[] args) throws IOException {
    X9 x9 = X9.create();                                    // license, config, threads — all from convention
    X9File937 file = x9.read(args[0]);                      // dialect inferred from input
    X9ValidationReport report = file.validate();            // typed report, never throws on data errors
    file.items().first().setAmount(X9Money.usd("88.88"));   // typed money, JavaBean setter
    file.items().remove(1);                                 // attached addenda auto-cleaned, trailers auto-rebalance
    x9.write(file, args[1]);                                // format inferred from input
}
```

Five functional lines. A 150× compression versus the legacy example. Each line is doing customer-domain work; nothing visible is SDK ceremony. License, configuration, dialect binding, image folder, trailer rebalance, output format, EBCDIC handling, atomic write, resource cleanup, threading — all defaulted into invisibility. Each implicit decision has an override path, and every override path leads to the engines or to legacy when the customer needs full control.

This example is not the only API the modern surface supports. A long-running batch processor still gets per-file `try-with-resources` for prompt memory release; a Spring Boot service still gets `X9SdkApplication`-as-bean and per-request `X9SdkBase`; a multi-dialect router still gets explicit per-call dialect typing. The point of the example is not that it is the only shape — it is that the *simplest* shape, the one a prospect reads in five seconds and decides whether to keep evaluating, is honest about how compact good API design lets it become.

## What this document does NOT do

The vision deliberately does not commit to several specific things. Naming them protects the work against accidental scope expansion.

The vision **does not replace `x9Sdk`**. Existing X9.37 customers running the legacy SDK in production are unaffected. The dual-API model the charter establishes is the load-bearing strategic decision; this vision builds on it without revising it.

The vision **does not require a Java version uplift**. The Java 8 floor is preserved. Newer Java features (records, sealed types, pattern matching, virtual threads at the JVM level) become *available* to customers running on newer JVMs but are never required by the SDK's source compatibility level.

The vision **does not take a Spring dependency**. It does not take a Lombok dependency. It does not bundle Micrometer or OpenTelemetry. The SDK is friendly to those ecosystems because its public shape is compatible, not because it carries weight from any of them. This is a deliberate posture against introducing OSS surface that the team would have to track for currency and vulnerability.

The vision **does not reimplement the engines**. The modern API is a layered facade over the existing substrate. Engines do the work; the modern API exposes it under a customer-first surface. Where engine APIs need extension to support the modern surface (for example, injectable executors, observability hooks), the extensions are additive — engines are not rewritten.

The vision **does not break customer integrations**. Every existing code path against `x9Sdk` continues to compile and run as it does today. The modern API is parallel; the modernization happens through customer adoption, not through forced migration.

## Business case

This section answers the question "why do this work, not just for engineering aesthetics, but for the business?" Each argument is paired with the customer or revenue outcome it produces.

### Customer acquisition in growth segments

X9.37 carries the current commercial weight, and X9.37 customers are not asking for the modern API. The customers who *are* judging the SDK against contemporary alternatives are the ones evaluating it for ACH, ISO 20022, and CPA 005 — the segments where X9Ware has zero or near-zero current customers and where each acquisition is incremental revenue rather than a defense of existing revenue.

A prospect evaluating an SDK for ISO 20022 pain.001 generation in 2026 is comparing the SDK against alternatives that look like Spring Boot's `RestClient`, Apache Kafka's `Producer`, AWS SDK v2 — modern Java APIs. They are reading the quickstart example, building an evaluation project, deciding within an hour whether the SDK is a credible choice. The SDK's quickstart code is the first piece of evidence they evaluate, and "five lines that do real work" is a different sales conversation from "seventy-line preamble before the work begins."

The modernization is the entry-evaluation-friendly surface that makes ACH, ISO 20022, and CPA 005 prospects credible to win. The charter's parent design document already acknowledges this when it identifies these dialects as the segments where the modern API matters most. This vision sharpens the acquisition argument by extending the modernization to the dimensions a 2026 evaluator actually screens against.

### Sales positioning against competitors

Every modern Java SDK in adjacent markets — payments, messaging, data integration, financial services — has at least a contemporary fluent surface and increasingly has the surface dimensions this vision names. Walking into a sales conversation with a contemporary API to demonstrate alongside the substrate-stability story for existing customers is competitive maintenance, not surface ergonomics theater. Walking in without one is letting the competitor define the framing.

The CICS macro-level versus command-level analogy the charter uses is itself a sales asset because it positions the dual-API approach as a deliberate, well-known industry pattern rather than transitional churn. Existing customers hear "your investment is protected"; new customers hear "here is the modern API." Both messages are honest, both are first-class, neither is an apology.

### Future-proofing the SDK's competitive position

The pace of Java ecosystem evolution is steady. Virtual threads, structured concurrency, pattern matching, records, and sealed types have all become standard features within the past five years. The next five years will bring further evolution. An SDK whose architecture *precludes* virtual-thread compatibility, *precludes* DI integration, *precludes* observability seams — even on a Java 8 floor — has dug a hole that future ecosystem changes will deepen. An SDK whose architecture *composes* with each of those features, even when the JVM floor today does not require them, is positioned to inherit the ecosystem's gains for free.

This is the difference between modernization that pays dividends and modernization that solves a one-time problem. The principles in this document are chosen so that future Java ecosystem shifts compose with the SDK's design rather than fighting it.

### Risk reduction for the existing customer base

The dual-API model is itself the risk-reduction story. Existing X9.37 customers running `x9Sdk` are not asked to adopt anything new; their integrations continue running unchanged; their support relationship is unaffected. The modernization happens in parallel, not on top of them. This is the difference between a modernization that risks the current customer base and one that protects it.

Without the dual-API guarantee, customer-base risk is the dominant cost of any modernization. With it, the cost is the engineering effort of producing the second API — bounded, scoped, and not on the critical schedule for any existing customer.

### Why the cost is justified

The modernization cost is bounded because the substrate is sound and stays untouched. The engines do not get rewritten. The dialect specifications do not get rewritten. The trailer logic does not get rewritten. What gets written is a layered facade — interfaces, JavaBean shapes, observability seams, and a customer-first surface — that exposes the existing substrate through a contemporary API.

The investment is recoverable: every customer acquired in ACH, ISO 20022, or CPA 005 because the modern API made the SDK credible to evaluate is incremental revenue against zero. The competitive cost of *not* doing the work is the slow attrition of failed evaluations to competing SDKs whose contemporary surface looks more polished.

## How this composes with the charter

The vision and the existing fluent API charter sit at different levels and address different concerns. They compose; neither is above the other.

The **charter** locks the structural conventions every fluent Engine follows: terminal verb, factory shape, source/sink shape, dialect dispatch, escape hatch, lifecycle ownership, exception model. These conventions are necessary for the API's grammar to read consistently across the engines.

The **vision** establishes the principles that govern *which* dimensions the modernization addresses and *how* customers experience the modernized API at its public surface. The principles produce: interface-based public surface, JavaBean data shapes, observability seams, modern threading, ecosystem-friendliness, layered access, first-impression-as-budget.

The two together describe what the modernized SDK looks like in 2026.

| Concern | Where it lives |
|---|---|
| Fluent grammar conventions | Charter |
| Engine factory shape | Charter |
| Source and sink builder methods | Charter |
| Lifecycle ownership rules | Charter |
| Escape hatch shape | Charter |
| Application-managed logging default | Charter |
| Interface-based public surface | Vision |
| JavaBean data conventions | Vision |
| Observability seams (Micrometer, OpenTelemetry) | Vision |
| Modern threading and virtual-thread compatibility | Vision |
| Ecosystem-friendliness without dependency | Vision |
| First-impression code as a budget | Vision |
| Layered access pattern | Vision |
| Customer-facing posture matching the work | Vision |

A reader looking for "what shape does the validate Engine's run method return?" is reading the charter or its child specs. A reader looking for "what does the customer's first hour with the SDK look like, and why is it shaped that way?" is reading this vision. Both questions are necessary; both answers are part of what shipping the modern API means.

## What is not decided here, and why

The vision deliberately defers several specific commitments to documents that come after the substrate review. Each deferral is recorded so the absence is intentional and not an oversight.

**Specific class signatures, interface shapes, and method names** are deferred. The shapes that read well at the call site depend on the actual substrate types they wrap (`X9SdkBase`, `X9SdkIO`, the engine bases, the dialect-specific concretes), the actual lifecycle calls those substrate types make, and the actual data structures the engines produce and consume. A vision that committed to specific signatures without that grounding would be making promises it could not honor; the substrate would force revision the moment specifics met code.

The deferral is not indefinite. The next phase after this vision is a substrate review that reads the relevant classes (`X9SdkBase`, `X9SdkIO`, `X9ObjectManager`, `X9Binder`, `X9Rules`, `X9RulesManager`, `X9Writer` and the dialect readers, the engine bases) and the existing example programs and User Guide. After that review, concrete charters and specs ground the modern API in the actual substrate.

**Implementation sequencing and release timelines** are deferred to an implementation plan that follows the substrate review. The vision does not pin which Engine retrofits first, which dialects ship in which release, or which customers see which capabilities when. These decisions depend on substrate review findings and on team and customer signal that the vision cannot anticipate.

**Detailed Engine conventions** beyond what the charter already locks are deferred. If observability seams need a charter (they probably do), if threading patterns need a charter (they may), if interface naming conventions need a charter (they may), each one follows the same pattern Lowell's fluent charter follows — a focused document that locks the conventions for that dimension. The vision spawns charters; it does not absorb their responsibilities.

**Concrete code beyond the worked simplest example** is not in this document. The worked example exists to demonstrate the principles concretely; further code lives in the eventual specs and in the implementation itself, not in the vision.

## Path forward

The work proceeds in stages, each independently shippable and independently customer-visible.

**Stage 1: Vision (this document).** Captures the destination and the principles. No code. The artifact is this markdown file. After review and any revisions, the vision is published as the foundational design reference for the modern API.

**Stage 2: Substrate review.** Reads the relevant classes (`X9SdkBase`, `X9SdkIO`, `X9ObjectManager`, `X9Binder`, `X9Rules`, `X9RulesManager`, the engine bases, the dialect readers and writers) and the existing example programs and User Guide. Produces a substrate review document that summarizes what is there, what the public API will need to wrap, what is internal and stays internal, and what extensions the modern API will require of the existing engine bases. The substrate review is the bridge between vision and concrete design — without it, concrete commitments in the next stage would be speculation.

**Stage 3: Charters and specs.** Concrete API design grounded in the substrate review. Lowell's existing fluent charter is one such charter; it sits alongside any new charters the vision spawns (for observability, for threading, for interface conventions, etc.). Specs follow the charters and pin specific class signatures and method names per Engine.

**Stage 4: Pilot implementation.** One Engine or one capability shipped end-to-end through the modern API surface. Validates the charters and specs against working code. Customer-visible. The charter's existing pilot proposal (`X9ValidateEngine` retrofitted as the first fluent Engine) is the natural pilot for this work as well; the vision's principles apply to it without modification.

**Stage 5: Incremental rollout.** Additional Engines and capabilities migrate to the modern surface one at a time, each independently customer-visible and independently valuable. The dual-API model means there is no flag day; legacy `x9Sdk` continues unchanged throughout.

The cadence is recognizable from the charter's own staging — the vision does not propose a different rhythm, only an additional layer that informs the work the charter already plans.

## When to revisit

This vision captures the position as of May 8, 2026. Specific signals that warrant revising the vision rather than adapting around it:

The customer base composition shifts substantially — paying customers acquired in ACH, ISO 20022, or CPA 005 in non-trivial numbers may pull the principles in directions the current X9.37-centric framing does not anticipate.

A 2026-modern dimension named here proves to be a non-issue empirically — for example, observability seams turn out to be unimportant to the customer base we acquire, or virtual-thread compatibility turns out not to be on the evaluation screen. The dimension can be retired from the vision rather than carried indefinitely.

The substrate review surfaces a structural reality the vision did not anticipate — for example, an existing engine API that genuinely cannot be wrapped in an interface-based facade without rewriting the engine itself. The vision adapts to the substrate review's findings rather than the other way around.

A different dual-API outcome is forced — for example, a customer demand or a regulatory requirement that the legacy `x9Sdk` be retired by a date certain. The dual-API guarantee is the strategic load-bearing decision; if it changes, the vision's framing changes with it.

The vision is otherwise expected to be stable for the duration of the modern API rollout. Routine charter and spec decisions do not require vision revision — they live in the children documents the vision spawns.

## Related documents

- [`x9Sdk` fluent API charter](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md) — the structural conventions every fluent Engine follows
- [`x9Sdk` fluent API parent design](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-design.md) — the strategic context the charter sits in
- [`x9SdkApi` README](../../README.md) — the project's relationship to `x9Sdk` and current status
- *Substrate review document* — to be authored in Stage 2
- *Concrete API charters and specs* — to be authored in Stage 3
