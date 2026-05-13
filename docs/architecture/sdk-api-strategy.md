# X9Ware SDK API — Strategy

X9Ware LLC • May 8, 2026

## Status

**DRAFT — under active design.** This document is a strategy, not a charter. It articulates the destination and the principles that lead the design there, and explicitly does not pin specific class signatures, method names, or implementation sequencing.

## Working Notes — Synthesis as of May 8, 2026 (NOT part of the final vision; captures discussion state for the upcoming rewrite)

These notes record the position reached through discussion of Topic 1 (Spring positioning) and Topic 2 (charter overview) before Topic 3 (substrate review) begins. They will be folded into the rewrite of this document and removed from the published version. Authoring constraints follow at the end.

### Audience and document length

- The primary reader is internal. Calibrate vocabulary to a reader; concrete code carries more weight than abstract terminology. The 4/28 internal email named the modernization themes (interface-based design, JavaBeans, observability hooks, modern threading, Spring-ecosystem compatibility, containerization), so the proposal is not landing cold.
- The 4/28 email invited "a better idea entirely" and a later message invited critique of the Engine-based design. Direct critique is welcome; deference is not.
- Target length: 4–8 pages, ~6 pages preferred. Executive summary up front serves as the under-a-page elevator pitch. Body earns the rest of the length by adding substance the summary cannot carry.

### Spring positioning (Topic 1 outcome)

- **In 2026, Spring ecosystem compatibility is table stakes for enterprise Java integrations.** Market data: 55–60% of enterprise Java workloads on Spring Boot, ~92% of enterprise backend job postings, dominant in regulated sectors (banking, healthcare). Quarkus / Micronaut are real but bounded; their differentiators (cold start, memory footprint) do not translate to file-processing workloads. Losses from a Spring choice are small in X9Ware's specific market.
- Working framing for the vision: "Market analysis tells us Spring ecosystem compatibility is table stakes in 2026, but we should also decide whether x9SdkApi itself is built on top of Spring Framework. x9Sdk will remain a framework-neutral option and carry no additional open source dependencies. X9Ware will soon be embarking on Spring Boot application development with X9Collector and will therefore need capabilities to efficiently build and release quality software to quickly remediate open source vulnerabilities."
- **Spring Framework vs. Spring Boot is a real fork.** Spring Framework is the library substrate (DI, Resource, ConversionService, transactions). Spring Boot is the application framework above it (autoconfiguration, starters, actuator). For x9SdkApi as a library, Spring Framework is the right depth.
- **Spring Boot AutoConfiguration is a meaningful customer-facing win** — it can collapse the X9SdkApplication setup from ~5 lines to zero. That alone may justify a thin Spring Boot starter as a companion artifact to x9SdkApi. To be developed further; do not name "starter" terminology in the vision (Boot starters are not familiar to the primary reader).
- **Java floor split:** x9Sdk stays Java 8 (existing 18 customers depend on this). x9SdkApi targets Java 17 (or 21) — Spring Framework 6.x and Boot 3.x require Java 17, and the dual-API model already preserves the Java 8 path for customers who need it.
- **Decision is not yet made.** The vision presents this as a decision to make, not a decision made, but the reasoning leans toward Spring Framework as the floor for x9SdkApi.

### Customer base context

- Existing 18 SDK customers came in over 10+ years and use the legacy x9Sdk surface (X9SdkBase / X9Writer / X9SdkIO). Their tech stacks reflect that era; they are not the audience for x9SdkApi.
- New prospects in 2026 are predominantly Cloud + Spring Boot. They are the audience for x9SdkApi.
- The dual-API model is not "old API + new API" abstractly; it is "old customers stay on x9Sdk + new customers use x9SdkApi" with the era cleanly separating them. **The escape hatch from x9SdkApi already exists — it is x9Sdk.**

### Engine reframe (Topic 2 outcome)

- **Engines belong in x9SdkApi, not in x9Sdk.** Engines are weeks old (38 classes renamed in commit `4bc29141`); they are not legacy. The 18 existing customers use the legacy direct-construction surface, not Engines. Putting Engines in x9Sdk and saying "x9SdkApi recommends you skip past them" is the wrong message — it makes Engines sound retrofitted or broken.
- **Engines should conform to x9SdkApi's design language.** Spring-friendly (interface-based public surface), JavaBean conventions on result objects, customer ergonomics (minimal cognitive load), pipeline composability surfaced explicitly.
- The work is not "build a facade above Engines." The work is "make Engines themselves feel native to x9SdkApi." Improvements to Engines are charter revisions to react to, not workarounds.
- **Engine-centric, not facade-centric, not file-centric.** Earlier feedback on a sketch example was that it looked "essentially like x9utilities" — a file-shaped batch verb rather than embeddable application code. The mental model from that feedback: the Engine is the starting point; source (file, stream, programmatic items) is configuration on the Engine; lifecycle is startup → populate options → run → shutdown. The customer's first 5 lines lead with the Engine, not with a file or facade.
- **Source flexibility is a design property, not an enumeration.** The API supports any source by construction. Worked examples should show file and stream (the most common cases); the design language carries the rest implicitly.

### Charter critique points to surface honestly

These are points where the charter and the x9SdkApi design language diverge. Each deserves direct engagement in the vision doc, not papered over:

- **Dialect-concrete return types from factories.** Charter chose `X9WriteEngine.x937(app)` returning `X9WriteEngine937` for compile-time builder visibility. Earlier feedback reacted against `X9ValidateEngine937 validateEngine = ...` as "dialect specific assignment, which seems to go against standard generic usage." Possible paths: return abstract base; document `var` as the canonical idiom; auto-detect dialect on read; design so the dialect concrete is rarely captured into a local variable. This is the cleanest place where author instincts and the charter itself point in different directions.
- **X9SdkApplication as customer-visible lifecycle root.** Charter makes this the customer's mental-model anchor; modern Java API expectations push it to be implementation detail handled by the container or by the facade. With Spring Boot AutoConfiguration: zero customer code. Without Spring: ~5 lines of try-with-resources. This is a meaningful divergence from the charter's framing.
- **`engine.legacy()` accessor.** Charter calls it "the bridge to legacy state." In the artifact split where Engines live in x9SdkApi, there is no "legacy state" to bridge to within x9SdkApi — legacy lives in the parent x9Sdk artifact. The accessor either disappears or is repurposed (e.g., `unsafe()` for power-user operations the surface does not cover).
- **SLF4J as default — smaller change than the charter implies, but with a caveat.** SDK *runtime* code already uses SLF4J everywhere (verified — no `java.util.logging` imports anywhere in x9Sdk). The JUL coupling lives in `x9SysTools` logging package — `X9JdkLogger`, `X9LogConsoleFormatter`, `X9LogFileFormatter`, `X9LogConsoleHandler` — which examples and X9Assist call as `X9JdkLogger.initializeLoggingEnvironment()` to wire JUL underneath SLF4J. The actual change is whether the modern API calls `X9JdkLogger.initializeLoggingEnvironment()` (current example behavior) or not (well-behaved-library default). **Caveat:** existing code may depend on JUL-specific format features (line numbers etc.) — needs to be verified before claiming "logback or any other binding works fine."
- **Javadoc template per Engine.** Policy is right; template content was written before x9SdkApi design-language decisions and is worth revisiting.
- **`X9File extends java.io.File` is a structural mismatch with source-flexible design.** Hard tie to `java.io.File` encodes the file assumption into the type system — cloud storage (S3, Azure Blob), Spring `Resource`, `java.nio.file.Path`, and stream-only inputs all become second-class. The modern API's source/sink abstraction cannot be `X9File`-rooted. X9File becomes implementation detail customers do not see.
- **X9Object exposure to customer code.** X9Object is internally a memory-optimized record representation (raw byte arrays, linked-list via X9ObjectManager) — designed for memory efficiency on large files, not for customer ergonomics. Examples confirm customers walk X9Object directly (`getFirst`/`getNext`/`getPrev`, type wrappers, byte-array mutation) — that is a layer customers should not be writing code in. The customer-facing type is `X9Item937` (and equivalents); X9Object stays internal.

### Pipeline composability (positive sales point)

- Read → validate → modify → write composes as a fluent chain. Demos cleanly to prospects in a way the legacy imperative style cannot. Worth highlighting in the vision as a place where the fluent grammar and the customer-friendly surface reinforce each other rather than compete. Avoid the word "Camel" (the reference may not be familiar); name the value as "pipeline composition" or "chained operations."

### Substrate review (Topic 3 outcome)

- **The substrate is healthier than expected.** Most "Spring resistance" lives in addressable patterns, not structural ones. The SDK is reasonable Java; it is the *examples* that look mainframe-flavored (hardcoded paths, `System.exit` in main, `X9Options` global mutation, linked-list traversal). Vision must distinguish these honestly: most of the customer-facing perception gap comes from examples + missing abstractions, not from a broken SDK.
- **Engines confirmed pre-fluent.** Conventional Java classes after the rename, constructor-DI shaped (`new X9ValidateEngine937(sdkBase, fileAttributes, repairEngine, progressEvent)`). Caveat: a pending large commit may include some of these changes. The vision describes what the API should be, not the state of the code this week.
- **`X9SdkApplication` does not exist yet.** It is a proposal in the charter, not actual code. Whatever it should look like in x9SdkApi, we shape it from the start (no migration burden).
- **Customer-facing Spring-resistance points (refined to bleed-through-only).** Do not surface internal-only patterns; only those that bleed into customer code:
  1. The seven-step preamble (~50 lines of invariant boilerplate per example, every customer hits this)
  2. `java.io.File` hard ties via X9File — customers in cloud-storage scenarios cannot use the SDK ergonomically
  3. X9Object linked-list as a customer pattern — examples show customers walking it directly; modern API expectations are typed-item streaming or list iteration
  4. Direct `new` of Engines, no factory interface — DI-resistant
  5. Concrete-class public types — fine for constructor injection but limit substitution and testing patterns
  6. License key embedded as a hex compile-time constant in `X9RuntimeLicenseKey` — resists multi-tenant, evaluation, and rotation scenarios
- **What is *not* customer-facing pain (do not over-claim):** synchronized methods on X9SdkBase, static initialization in X9MessageManager, internal `X9Options` references inside SDK code. These are real but stay internal; the vision should not list them as customer concerns.
- **`X9AchWriter` missing `addItem(X9ItemAch)`.** Confirmed by `X9ConstructAch.java` (447 lines) which builds CSV arrays manually as a workaround. The charter's plan to add this is well-motivated by real customer pain.
- **x9SysTools is already Spring-safe.** No static initializers touching global state, no JVM property writes at class load, SLF4J is the substrate. Opaque infrastructure dependency for x9SdkApi; classes do not surface in the public API.

### Strategic framing

- **Adopt "think big / deliver big" as the operating principle, named in the doc.** This is a phrase already in use internally; using it grounds the vision in an existing strategic posture rather than imposing one.
- **The AI-disruption lens makes incrementalism a losing strategy.** AI lets X9Ware build anything we can dream of, but it also enables disruption — competitors with the same AI tools can match incremental moves. Differentiation has to be in the *destination*, not the velocity. The vision earns its scope by being a destination worth aiming for.
- **The "would you write this yourself" test as the design north star.** If a developer evaluating us thinks "yes, this is what I would build," we win. If they think "this is what an enterprise SDK looks like," we lose.
- **Build on the existing competitive framing in the charter.** Strong passages already exist: "The core APIs need to look rational and, more importantly, better than anything else the technical teams from potential new customers are considering" and "A contemporary API surface that compares cleanly against modern SDKs in the financial messaging space." The vision adds the 2026-specific lens — what "looking better" actually means in 2026 (interface-based, Spring-friendly, JavaBean conventional, stream-first, observable, container-ready, virtual-thread-compatible, AutoConfiguration-eligible).
- **Earlier feedback paraphrased (not direct quote).** A sketch example "looked essentially like x9utilities" — a file-shaped batch verb, not embeddable application code. Stream support and database-built x9 file scenarios were raised as illustrations of source flexibility (not a checklist of cases the vision must demonstrate). The mental model: Engines have their own API; source is configuration; lifecycle is startup → options → run → shutdown.
- **The competition is open-source and commercial.** Open-source integration libraries (Apache Camel and similar) and commercial check-processing SDKs both compete with X9Ware. The bar is "developers would write this themselves" — that is the differentiation play.

### X9Collector framing (do not overstate)

- X9Collector is a Spring Boot server-based product that consolidates files from distributed capture points at financial institutions. It is one customer of x9SdkApi, not a generic protocol facade over a service catalog. Frame it as evidence X9Ware will have a Spring footprint regardless, which means the engineering capability to manage Spring dependencies and CVE remediation is needed regardless of the x9SdkApi Spring decision.
- **Do not weight X9Collector above paying customers.** Their needs come first; X9Ware's internal product needs are an organizational consequence, not a tipping factor.

### Artifact split (working synthesis)

- **x9Sdk** = legacy direct-construction API the existing 18 customers depend on (X9SdkBase, X9Writer, X9SdkIO, etc.). Framework-neutral. No new OSS dependencies. Java 8. Continues unchanged.
- **x9SdkApi** = the modern API. Engines live here, conforming to one design language. Spring Framework as the substrate (decision pending). Java 17 (or 21) floor. Possibly a thin Spring Boot AutoConfiguration companion artifact.

### Authoring constraints for the rewrite

- Read `C:\Users\X9ware7\Downloads\sdk-fluent-api-charter-review.md` before rewriting. That is an earlier review of the charter, written earlier in this session and shared internally on 4/28. Build on it where the prior thinking holds; explicitly mark where thinking has evolved (and why) so the progression is visible rather than feeling contradictory.
- Include direct before/after code comparisons. Concrete code is the universal language; prose framing serves the code, not the other way around.
- Do not introduce Spring Boot starter terminology. Boot starters are not familiar to the primary reader; explain *what AutoConfiguration removes* (customer setup code) rather than *how it works*.
- Be explicit that the document is a vision/draft proposal, not a final design.
- Hold target length at 4–8 pages, ~6 preferred. Executive summary up front. The current draft is over budget and needs a real cut, not just a polish pass.
- **Worked examples must be Engine-centric, not file-centric.** Lead with the Engine; show source as configuration on the Engine. No `x9.validate(file)` shorthand. No utility-shaped `process(inputFile, outputFile)` patterns. Show file and stream as sources; do not enumerate every source type one by one.
- **Examples must not name `X9File` or `X9Object` in customer-facing code.** Those are implementation detail. Use `Path` or `InputStream` as input types; use `X9Item937` (or equivalents) as the typed-item customer surface.
- **Lead the vision with strategic framing ("think big / deliver big," AI-disruption, "would you write this yourself" test).** The competitive case earns the scope; the technical proposals follow from it.
- **No personal names anywhere in the document, including working notes.** Use roles, the team, the artifact itself, or neutral framing.

## Executive summary

X9Ware should publish a second, modern Java API alongside the existing SDK. The legacy `x9Sdk` continues unchanged for the eighteen customers built on it. A new artifact, `x9SdkApi`, becomes the surface 2026 enterprise prospects evaluate when comparing X9Ware against commercial and open-source alternatives.

The strategic case is **customer acquisition**. In 2026, Spring ecosystem compatibility is table stakes for enterprise Java; roughly nine of ten Fortune 500 companies use Java with Spring Boot as the primary backend choice. AI-armed competitors can match incremental moves quickly, which means our differentiation has to live in the *destination*, not the velocity. The right test for the destination is: "would a developer evaluating this say *yes, this is what I would build myself*."

`x9SdkApi` delivers that destination. The renamed Engines and the proposed `X9SdkApplication` become the customer-facing surface, refined to the design language modern Java APIs follow: Engine-centric (the Engine is the noun, source is configuration), source-agnostic (file, stream, programmatic items uniformly), interface-based, JavaBean-conventional, observability-ready, container-friendly, virtual-thread-compatible. The seventy-line preamble that opens every example today collapses to zero customer code in Spring Boot, and to a small builder block elsewhere.

The work builds on the existing fluent API charter; it does not replace it. The charter's universal disciplines (single terminal verb `run`, dual-overload Engine factory, application-managed SLF4J), its IBM-CICS dual-API framing, its escape-hatch design, and its testing discipline all carry forward. The vision adds the dimensions a 2026 enterprise prospect expects beyond fluent grammar — and refines a small number of charter points where the customer-facing surface needs to evolve.

This is "think big, deliver big" applied to the SDK API.

## Strategic position

Two operating principles drive this vision.

**Think big, deliver big.** The destination has to be worth aiming for. Incremental polish on the existing SDK is not the destination; an API that 2026 enterprise developers want to use is.

**Differentiation lives in the destination, not the velocity.** AI lets X9Ware build anything we can dream of — but it lets competitors do the same. The advantage is no longer in shipping faster; it is in shipping a destination materially better than what an open-source integration framework or a commercial check-processing SDK ships. The bar is high because the competition's bar is rising too.

**The acquisition lens.** The eighteen current x9Sdk customers came in over the past decade and use the legacy direct-construction surface. Their tech stacks reflect that era. They are not the audience for `x9SdkApi`; they continue with x9Sdk unchanged. The audience for `x9SdkApi` is the new prospect a sales conversation surfaces tomorrow — predominantly Cloud, Spring Boot, container-deployed, modernization-aware. That prospect evaluates the SDK by feel before they ever ship a line of integration code; the API has to read as something they would have built themselves if they had a year to do it right.

**Spring ecosystem compatibility is table stakes in 2026.** Approximately 55–60% of enterprise Java workloads run on Spring Boot, and ~92% of enterprise backend Java job postings reference Spring. Quarkus and Micronaut exist but are bounded — their differentiators (cold-start time, memory footprint) do not apply to file-processing workloads. For X9Ware's market, choosing Spring as the substrate is a low-risk decision with high upside.

The existing charter already states the competitive case well: "The core APIs need to look rational and, more importantly, better than anything else the technical teams from potential new customers are considering." This vision agrees and operationalizes it — in 2026, "looking better" means a specific set of conventions modern Java APIs share.

## The proposal — two artifacts, two roles

The dual-API model becomes two physical Maven artifacts.

- **`x9Sdk`** — the legacy direct-construction surface that eighteen existing customers depend on. Java 8 floor. Framework-neutral, no new open-source dependencies. Continues exactly as it is. The existing design conventions stay intact for the customers who depend on them.
- **`x9SdkApi`** — the modern API. Java 17 (or 21) floor. Built on Spring Framework as the substrate (decision pending — see *What is not decided here*). New customers and 2026 prospects use this artifact. One coherent design language across every type it exposes.

`x9SdkApi` depends on `x9Sdk` as a Maven artifact. Customers wanting full direct-construction control just import x9Sdk; customers wanting the modern surface import x9SdkApi (which transitively brings x9Sdk). The escape hatch from x9SdkApi already exists: it is the parent x9Sdk artifact.

**The renamed Engines (`X9ValidateEngine`, `X9ScrubEngine`, `X9CloneEngine`, etc.) live in `x9SdkApi`, not in `x9Sdk`.** They are weeks old, not legacy; the eighteen existing customers use the legacy direct-construction classes (`X9SdkBase`, `X9Writer`, `X9SdkIO`), not Engines. Putting Engines in x9Sdk and recommending customers skip past them sends the wrong message — it makes Engines sound retrofitted or broken. The artifact split lets each surface own its design language coherently. The existing charter for fluent Engines becomes the design baseline for x9SdkApi; the conventions x9Sdk customers depend on stay in x9Sdk.

**Java floor split.** Spring Framework 6.x and Spring Boot 3.x require Java 17. Pinning x9SdkApi to a Java 17 floor (or 21) lets x9SdkApi use modern Spring; existing customers on Java 8 stay on x9Sdk, which keeps its Java 8 floor. The dual-API model already preserves the Java 8 path for customers who need it; x9SdkApi does not have to compromise the modern surface to accommodate them.

## What "modern" means in 2026

The "would you write this yourself" test is the design north star. The dimensions that follow are the answer to "what does that test actually require?"

**Engine-centric API.** The Engine is the noun. The customer's first interaction is with an Engine — `X9ValidateEngine`, `X9WriteEngine`, `X9ReadEngine` — not with a file, a facade method, or a utility-shaped wrapper. Source (file, stream, programmatic items) is configuration on the Engine, not part of the verb's identity. The Engine's lifecycle is build → configure → run, and the result is a typed summary the customer reads.

**Source-agnostic by construction.** The API supports any source uniformly: `Path`, `InputStream`, Spring `Resource`, programmatic items. No type in the public surface forces customers through `java.io.File`. The current `X9File` extends `java.io.File` and encodes that constraint into the type system; modern x9SdkApi cannot. Cloud storage, Resource abstractions, and stream-only inputs all become first-class without API duplication.

**JavaBean conventions on customer-facing types.** Result objects (`X9WriteSummary`, `X9ReadSummary`, `X9ValidationResult`) follow JavaBean naming — `getRecordCount()`, `getErrorCount()`, etc. Configuration types (the equivalent of `X9HeaderXml937`) accept setter-based or builder-based population. The existing `X9Object` byte-array record representation stays inside the implementation; customers work with typed item classes (`X9Item937`, `X9ItemAch`, etc.) and never reach for X9Object directly.

**Interface-based public surface.** Engines and the major lifecycle types expose Java interfaces, not just concrete classes, for the public API. Customers using Spring Framework's dependency injection get clean substitution (test doubles, partial mocks); customers using plain `new` construction lose nothing.

**Lifecycle invisible to the customer.** `X9SdkApplication` is the lifecycle root, exposed as a Java interface. In Spring Boot, AutoConfiguration constructs the implementation and wires it as a bean; the customer `@Autowired`s `X9SdkApplication` and uses it as a parameter to Engine factories. In plain Java, a small builder block constructs an implementation once and keeps it for the application's lifetime; it does not appear in per-operation code paths. The customer never *constructs or manages* `X9SdkApplication`; the container or the builder handles both. Customers who need to substitute their own implementation can do so against the interface.

**Observability seams.** SLF4J for logs (already in place across x9Sdk runtime). Micrometer interfaces for metrics (the customer's existing meter registry is honored). OpenTelemetry-compatible spans on long-running operations. The customer's observability stack — Prometheus, Datadog, New Relic, anything else — works without us picking the backend.

**Pipeline composability.** Read → validate → modify → write composes as a chained operation. This is a place where the fluent grammar and the modern API design language reinforce each other rather than compete; the chain reads cleanly and demos cleanly to prospects evaluating the SDK.

**Modern threading and container-friendliness.** Engines do not hold thread-locals or static caches. Per-task state lives on instances the customer creates (or that Spring scopes). Virtual threads (Java 21+) work without additional configuration. Container lifecycle (Spring `@PreDestroy`, Kubernetes pod shutdown) closes resources cleanly via `AutoCloseable`.

## A worked example

Modern Spring Boot service, validating an uploaded check file:

```java
@Service
public class CheckProcessor {

    @Autowired X9SdkApplication x9;

    public X9ValidationResult verify(InputStream upload) {
        return X9ValidateEngine.x937(x9)
            .fromStream(upload)
            .validateTiffImages(true)
            .run();
    }
}
```

Nine lines including the class declaration and method signature. No setup code. No license registration. No XML configuration loading. No dialect bind. No image folder setting. No `try-with-resources` around an SdkIO. The customer types the Engine and the operation; everything else is handled by Spring AutoConfiguration plus the Engine's defaults.

What this replaces: a roughly seventy-line preamble that opens every example today, plus the imperative I/O loop and manual heap walks that follow it. The Appendix shows the full contrast end-to-end against the current 760-line `X9VerifyX9.java`.

The same Engine accepts a `Path` source instead of a stream, or a programmatic list of items, with no different API shape — just a different builder method. Source is configuration on the Engine, not a different API for each source type.

## How this composes with the existing charter

The charter is the foundation. The vision builds on it directly.

**Carries forward without change:**

- The IBM-CICS dual-API framing — the charter's most strategically important commitment, and a sales-presentation asset on its own
- The three universal disciplines: single terminal verb `run`, dual-overload Engine factory accepting either an `X9SdkApplication` or an `X9SdkBase`, application-managed SLF4J logging by default
- The escape-hatch design (a single named door from fluent code to legacy state) — repurposed in the artifact split (see *Refines* below)
- The testing discipline: JUnit 5, charter-conformant test matrix per Engine, legacy-vs-fluent byte-equivalent convergence tests
- The ACH `addItem(X9ItemAch)` capability work — real customer pain, confirmed by `X9ConstructAch.java` building CSV arrays manually as a 447-line workaround
- The User Guide reframe (ODM master document, sdkFiles topic structure, dual-purpose website feature cards)
- The X9ModifyEngine addition

**Refines (areas where the customer-facing surface needs to evolve):**

- *Lifecycle visibility.* The charter makes `X9SdkApplication` the customer's mental-model anchor. The vision moves it to implementation detail handled by Spring AutoConfiguration (in Spring Boot apps) or by a small builder block (otherwise). The customer's code does not name `X9SdkApplication` directly; it `@Autowired`s an interface bean instead.
- *Source/sink abstraction.* The charter's `fromFile(File)` and `fromStream(InputStream)` shape is correct. The vision adds: `fromPath(Path)` and Spring `Resource`-based inputs become first-class peers. `X9File` (which extends `java.io.File`) becomes implementation detail customers do not see.
- *Typed item layer is the customer surface, not X9Object.* Examples today show customers walking the X9Object linked list (`getFirst`/`getNext`) and mutating byte arrays via type wrappers. That is a layer customers should not be writing code in. The customer surface is `X9Item937` and equivalents; X9Object stays internal.
- *Dialect-concrete return types from factories.* The charter chose `X9WriteEngine.x937(app)` returning `X9WriteEngine937` for compile-time builder visibility; earlier feedback reacted against `X9ValidateEngine937 validateEngine = ...` as "dialect specific assignment, which seems to go against standard generic usage." The vision: documenting `var` as the canonical idiom, or returning an abstract base where the dialect concrete is rarely captured into a local variable, resolves the tension.
- *`engine.legacy()` accessor.* The charter calls it the bridge to legacy state. In the artifact split, x9SdkApi has no "legacy state" within itself — legacy is the parent x9Sdk artifact. The accessor either disappears or repurposes (e.g., as a power-user escape for operations the surface does not cover).
- *SLF4J as default — smaller change than the charter implies.* SDK runtime code already uses SLF4J everywhere. The actual change is whether the modern API calls `X9JdkLogger.initializeLoggingEnvironment()` (current example behavior) or not (the well-behaved-library default). Caveat: existing logging-format dependencies (line numbers, class names) need to be verified before claiming any SLF4J binding works equivalently.

**Adds (dimensions the charter does not address):**

- Spring Framework as the substrate of x9SdkApi — the dependency that earns the rest of the design language
- Spring Boot AutoConfiguration as the highest-friction-removing additive layer — collapses the customer's setup code from a small builder block to zero (decision deferred; treats the customer's setup code as the wrong place for ceremony)
- Source-agnostic abstraction (Path, Resource, InputStream as first-class inputs)
- Observability seams (Micrometer interfaces, OpenTelemetry-compatible spans, alongside SLF4J)
- JavaBean conventions on customer-facing types
- Interface-based public surface for DI substitution and testing patterns

### On the prior review

A technical review of the charter was produced on April 28 and shared internally. That review found the charter's structural choices defensible and recommended a leaner first cycle; it did not engage deeply with the Spring decision, the source-flexibility question, or the artifact-split implications. Three positions in this vision evolved beyond that review:

- *X9SdkApplication.* The prior review called it "the right central abstraction." Current view: it is the right *lifecycle* abstraction, but customers should not see it in their code. AutoConfiguration hides it in Spring Boot; a builder hides it otherwise.
- *engine.legacy() accessor.* The prior review called it "unusually thoughtful." Current view: in the artifact split, x9SdkApi has no internal "legacy" to bridge to, and the accessor either disappears or repurposes as a power-user escape.
- *Engine location.* The prior review treated Engines as part of the existing SDK substrate. Current view: Engines belong in x9SdkApi (they are weeks old, not legacy, and the eighteen existing customers use the legacy direct-construction surface, not Engines).

These evolutions are the result of the artifact-split realization and a substrate review of x9Sdk and x9SysTools done after the prior review. They build on the prior review's substance rather than contradicting it.

## What is not decided here

This is a vision, not a charter. Specific decisions deferred to sibling charters:

- Whether x9SdkApi commits to Spring Framework as the substrate (recommendation: yes; final decision deferred)
- Whether x9SdkApi ships an AutoConfiguration companion artifact alongside the core (recommendation: yes after the core stabilizes; deferred for now)
- Java 17 vs Java 21 floor for x9SdkApi (Java 17 covers Spring 6.x; Java 21 unlocks virtual threads natively without back-port)
- Per-Engine builder method names beyond the charter's universal verbs (those remain the per-Engine charters' responsibility)
- The exact source-abstraction interface (a sibling charter for source/sink handling follows from this vision)
- The exact lifecycle abstraction for non-Spring deployments (a sibling charter for lifecycle follows from this vision)
- The ordering of Engine fluent-retrofit work between x9Sdk's existing renamed Engines and x9SdkApi's exposed Engines

## Path forward

1. **Vision review.** This document, reviewed internally and revised with feedback.
2. **Substrate decisions.** Sibling charters for: Spring Framework substrate, source/sink abstraction, lifecycle abstraction, observability seams.
3. **Pilot Engine.** One Engine implemented in x9SdkApi following the design language. Validates the conventions against real code before they are locked across the surface.
4. **Incremental rollout.** Subsequent Engines per charter, with the User Guide reframe shipping topics in lockstep.
5. **Spring Boot AutoConfiguration** as a follow-on artifact once the core stabilizes and customer demand is observed.

The pilot Engine is the empirical test. If the design language proves brittle in real code, the conventions revise before the rollout proceeds. The vision itself is stable; specific design choices below it (interface shapes, exact method names) are pilot outcomes.

## When to revisit

The vision is expected to be stable for the modernization rollout. Specific signals that warrant revision:

- A paying customer surfaces with conflicting needs that x9Sdk cannot serve and x9SdkApi cannot serve in its proposed form.
- The Spring ecosystem shifts materially (Spring Framework 7, Spring Boot 4) in ways that change the substrate decision.
- A competitor ships a check-processing SDK that materially redefines what "looking better in 2026" means and the vision's design language no longer matches.
- AI tooling changes the bar for "would you write this yourself" — the destination has to keep moving.

## Appendix — Five forms of X9VerifyX9

This appendix shows the same operation — open an X9.37 file, validate it, modify two records, and write a new output — in five forms. The contrast is the point.

The five forms in order:

1. **Legacy / existing** — current `X9VerifyX9.java` from x9SdkExamples
2. **Fluent (exactly per charter)** — what the charter alone delivers, no Spring
3. **Fluent (charter + x9SdkApi design language)** — same Engine grammar, refined to the design language this vision adds
4. **Modern Spring** — design-language fluent surface plus Spring AutoConfiguration
5. **Facade-shaped** — an earlier modernization sketch where Engines were fully abstracted; included for completeness as the opposite extreme of form 1

### 1. Legacy / existing

The current `X9VerifyX9.java` from `x9SdkExamples`. Seven-step preamble, manual heap walks via `X9ObjectManager.getFirst()/getNext()`, manual byte-array record mutation via type wrappers, manual trailer recomputation, manual stream-to-file copy. 760 lines.

```java
package com.x9ware.examples;

import java.io.ByteArrayOutputStream;
import java.io.File;
import java.math.BigDecimal;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;

import org.slf4j.Logger;

import com.x9ware.actions.X9Exception;
import com.x9ware.base.X9Item937;
import com.x9ware.base.X9Object;
import com.x9ware.base.X9ObjectManager;
import com.x9ware.base.X9Sdk;
import com.x9ware.base.X9SdkBase;
import com.x9ware.base.X9SdkFactory;
import com.x9ware.base.X9SdkIO;
import com.x9ware.base.X9SdkObject;
import com.x9ware.base.X9SdkObjectFactory;
import com.x9ware.base.X9SdkRoot;
import com.x9ware.core.X9;
import com.x9ware.core.X9FileAttributes;
import com.x9ware.core.X9Reader;
import com.x9ware.elements.X9C;
import com.x9ware.error.X9Error;
import com.x9ware.error.X9ErrorManager;
import com.x9ware.logging.X9JdkLogger;
import com.x9ware.logging.X9LoggerFactory;
import com.x9ware.options.X9Options;
import com.x9ware.tools.X9D;
import com.x9ware.tools.X9Decimal;
import com.x9ware.tools.X9FileIO;
import com.x9ware.tools.X9FileUtils;
import com.x9ware.tools.X9Numeric;
import com.x9ware.tools.X9TempFile;
import com.x9ware.types.X9Type01;
import com.x9ware.types.X9Type20;
import com.x9ware.types.X9Type25;
import com.x9ware.types.X9Type70;
import com.x9ware.types.X9Type99;
import com.x9ware.validate.X9TrailerManager937;
import com.x9ware.validate.X9ValidateEngine;
import com.x9ware.validate.X9ValidateEngine937;
import com.x9ware.validate.X9ValidateTiff;

/**
 * X9VerifyX9 is a fairly comprehensive example of x9.37 file processing. It includes the ability to
 * open files, load them to the heap, parse and list various record types, modify records, run the
 * same rule based validations that are used by our X9Assist desktop application, and write a
 * modified file back to the file system. This is a good overview of what the SDK can do.
 *
 * @author X9Ware LLC. Copyright(c) 2012-2024 X9Ware LLC. All Rights Reserved. This is proprietary
 *         software as developed and licensed by X9Ware LLC under the exclusive legal right of the
 *         copyright holder. All licensees are provided the right to use the software only under
 *         certain conditions, and are explicitly restricted from other specific uses including
 *         modification, sharing, reuse, redistribution, or reverse engineering.
 */
public final class X9VerifyX9 {

	/*
	 * Private.
	 */
	private final X9SdkBase sdkBase = new X9SdkBase();
	private final X9ObjectManager x9objectManager;
	private final X9Sdk sdk;
	private X9Reader x9reader;
	private X9Type01 x9Type01;
	private X9Type99 x9Type99;

	/*
	 * Computed totals (these are based on the actual items and not the trailer totals).
	 */
	private int fileBundleCount = 0;
	private int fileItemCount = 0;
	private int fileDebitCount = 0;
	private int fileCreditCount = 0;
	private BigDecimal fileDebitAmount = BigDecimal.ZERO;
	private BigDecimal fileCreditAmount = BigDecimal.ZERO;

	/*
	 * Trailer totals (taken from the file control trailer record).
	 */
	private int trailerCashLetterCount;
	private int trailerTotalRecordCount;
	private int trailerTotalItemCount;
	private BigDecimal trailerFileTotalAmount;

	/*
	 * Constants.
	 */
	private static final String X9VERIFYX9 = "X9VerifyX9";

	/**
	 * Logger instance.
	 */
	private static final Logger LOGGER = X9LoggerFactory.getLogger(X9VerifyX9.class);

	/*
	 * X9VerifyX9 Constructor.
	 */
	public X9VerifyX9() {
		/*
		 * Set the runtime license key (this class is part of sdk-examples). X9RuntimeLicenseKey
		 * must be updated with the license key text provided for your evaluation.
		 */
		X9RuntimeLicenseKey.setLicenseKey();

		/*
		 * Initialize the environment and bind to an x9.37 configuration.
		 */
		X9SdkRoot.logStartupEnvironment(X9VERIFYX9);
		X9SdkRoot.loadXmlConfigurationFiles();
		sdk = X9SdkFactory.getSdk(sdkBase);
		if (!sdkBase.bindConfiguration(X9.X9_37_CONFIG)) {
			throw new X9Exception("bind unsuccessful");
		}
		x9objectManager = sdkBase.getObjectManager();
	}

	/**
	 * Load the x9.37 file, run the validator, and list all errors.
	 */
	private void process() {
		/*
		 * Define our test file and ensure it exists.
		 */
		final File folder = new File("C:/Users/X9Ware5/Documents/x9_assist/files");
		final File inputX9File = new File(folder,
				"Test file with 10 checks and type 68 records.x9");
		if (X9FileUtils.existsWithPathTracing(inputX9File)) {
			LOGGER.info("processing x9file({})", inputX9File);
		} else {
			throw new X9Exception("x9file not found({})", inputX9File);
		}
		
		/*
		 * Read the x9.37 file, populate x9objects, and validate tiff images. This example reads an
		 * input file, but you can also use sdkIO.openInputReader() to read from an input stream.
		 */
		try {
			/*
			 * Load and validate from an input file.
			 */
			final boolean hasRecords = loadAndValidateFromInputFile(inputX9File);

			/*
			 * Provide summary information when the file is not structurally flawed.
			 */
			if (hasRecords) {
				/*
				 * Gather file content.
				 */
				gatherFileContent();

				LOGGER.info(
						"file data totals: bundleCount({}) itemCount({}) debitCount({}) "
								+ "creditCount({}) debitAmount({}) creditAmount({}) "
								+ "debitAmount({}) creditAmount",
						fileBundleCount, fileItemCount, fileDebitCount, fileCreditCount,
						fileDebitAmount, fileItemCount, fileDebitCount, fileCreditCount,
						fileDebitAmount, fileCreditAmount);
				LOGGER.info(
						"trailerCashLetterCount({}) trailerTotalRecordCount({}) "
								+ "trailerTotalItemCount({}) trailerFileTotalAmount({})",
						trailerCashLetterCount, trailerTotalRecordCount, trailerTotalItemCount,
						trailerFileTotalAmount);

				/*
				 * List some records in their native 80+ byte format.
				 */
				int loggingCount = 0;
				X9Object x9o = x9objectManager.getFirst();
				while (x9o != null && loggingCount < 10) {
					loggingCount++;
					LOGGER.info("recordType({}) recordNumber({}) content({})", x9o.x9ObjType,
							x9o.x9ObjIdx, new String(x9o.x9ObjData));
					x9o = x9o.getNext();
				}

				/*
				 * List validation errors.
				 */
				listErrorsToLog();

				/*
				 * Modify some records and write to another output file.
				 */
				try {
					modifysRecordAndWrite(new File(folder, "test.x9"));
					LOGGER.info("finished");
				} catch (final Exception ex) {
					throw new X9Exception(ex);
				} finally {
					sdkBase.systemReset(); // release all sdkBase storage
				}
			}
		} catch (final Exception ex) {
			throw new X9Exception(ex);
		}
	}

	/*
	 * Load and validate an x9.37 file.
	 *
	 * @param x9InputFile input file to be loaded and validated
	 *
	 * @return true if the file contains records otherwise false
	 */
	private boolean loadAndValidateFromInputFile(final File x9InputFile) {
		/*
		 * Read the input file, but you can also use sdkIO.openInputReader() to read from a stream.
		 */
		try (final X9SdkIO sdkIO = sdk.getSdkIO();
				final X9Reader allocated_Reader = sdkIO.openInputFile(x9InputFile)) {
			/*
			 * Allocate a tiff validator instance (we validate tiff images as the file is read).
			 */
			x9reader = allocated_Reader;
			final X9ValidateTiff x9validateTiff = new X9ValidateTiff(sdkBase);

			/*
			 * Read and store records until end of file.
			 */
			X9SdkObject sdkObject = sdkIO.readNext();
			final X9FileAttributes x9fileAttributes = sdkIO.getInputFileAttributes();
			while (sdkObject != null) {
				/*
				 * Create and store a new x9object for this x9.37 record.
				 */
				final X9Object x9o = sdkIO.createAndStoreX9Object();
				x9reader.finalizeReaderErrors(x9o);

				/*
				 * Attach the image byte array directly to this x9object.
				 */
				if (x9o.isRecordType(X9.IMAGE_VIEW_DATA)) {
					final byte[] imageArray = x9reader.getImageBuffer();
					if (imageArray != null) {
						x9o.setDirectlyAttachedImage(imageArray);
					}
				}

				/*
				 * Validate tiff images while the file is being read. Any error messages are queued
				 * and will be subsequently picked up by the validator.
				 */
				if (sdkObject.getRecordType() == X9.IMAGE_VIEW_DATA) {
					x9validateTiff.validateIncomingImage(x9o, x9reader.getImageBuffer());
				}

				/*
				 * Get next record.
				 */
				sdkObject = sdkIO.readNext();
			}

			/*
			 * Validate the file when the reader was successful.
			 */
			if (x9objectManager.getNumberOfRecords() > 0) {
				/*
				 * Assign x9header indexes.
				 */
				x9objectManager.assignHeaderObjectIndexReferences();

				/*
				 * Create a validator instance and verify the file from the x9objects.
				 */
				X9Options.applyAbaFrbDistrictValidation = true;
				final X9ValidateEngine x9validateEngine = new X9ValidateEngine937(sdkBase,
						x9fileAttributes);
				x9validateEngine.verifyFile();

				/*
				 * Log validation statistics.
				 */
				final X9ErrorManager x9errorManager = sdkBase.getErrorManager();
				LOGGER.info(
						"validation completed recordCount({}) errorCount({}) highestSeverity({})",
						x9validateEngine.getX9RecordCount(), x9errorManager.getTotalRunErrors(),
						x9errorManager.getRunSeverity());
			}
		} catch (final Exception ex) {
			LOGGER.error("validation exception", ex);
		}

		/*
		 * Return true when the file has one or more records.
		 */
		return x9objectManager.getNumberOfRecords() > 0;
	}

	/**
	 * Gather file content.
	 */
	private void gatherFileContent() {
		/*
		 * Get the first and last records within the file, which obtains the first and last physical
		 * records regardless of actual record type. If there is only one record on the file, then
		 * test two references would actually point to the same record.
		 */
		final X9Object fileHeader = x9objectManager.getFirst(); // file header is expected
		final X9Object x9last = x9objectManager.getLast(); // file trailer is expected

		/*
		 * Abort if record instances were not found. These can only be null when the file did not
		 * contain at least one record. If the caller does not want this abort, then they should
		 * check the record count before they invoke us.
		 */
		if (fileHeader == null) {
			throw new X9Exception("first record not found");
		}

		if (x9last == null) {
			throw new X9Exception("last record not found");
		}

		/*
		 * The file header is absolutely needed; we cannot continue without it.
		 */
		if (fileHeader.x9ObjType != X9.FILE_HEADER) {
			throw new X9Exception("first record not fileHeader({})", fileHeader.x9ObjType);
		}

		/*
		 * List file header.
		 */
		x9Type01 = new X9Type01(fileHeader);
		LOGGER.info(
				"file header: standardLevel({}) testFileIndicator({}) "
						+ "immediateDestinationRoutingNumber({}) immediateDestinationName({}) "
						+ "immediateOriginRoutingNumber({}) immediateOriginName({}) "
						+ "fileCreationDate({}) fileCreationTime{}) resendIndicator({}) "
						+ "fileIdModifier({}) countryCode({}) ucdIndicator({})",
				x9Type01.standardLevel, x9Type01.testFileIndicator,
				x9Type01.immediateDestinationRoutingNumber, x9Type01.immediateDestinationName,
				x9Type01.immediateOriginRoutingNumber, x9Type01.immediateOriginName,
				x9Type01.fileCreationDate, x9Type01.fileCreationTime, x9Type01.resendIndicator,
				x9Type01.fileIdModifier, x9Type01.countryCode, x9Type01.ucdIndicator);

		/*
		 * Create the file trailer type 9 instance when the last record is of proper type.
		 */
		final X9Object fileTrailer;
		if (x9last.x9ObjType == X9.FILE_CONTROL_TRAILER) {
			fileTrailer = x9last;
			x9Type99 = new X9Type99(fileTrailer);
		} else {
			fileTrailer = null;
			x9Type99 = null;
			LOGGER.warn("last record not fileControlTrailer({})", x9last.x9ObjType);
		}

		/*
		 * List file trailer.
		 */
		if (x9Type99 != null) {
			LOGGER.info(
					"file trailer: cashLetterCount({}) totalRecordCount({}) totalItemCount({}) "
							+ "fileTotalAmount({}) immediateOriginContactName({}) "
							+ "immediateOriginContactPhoneNumber({})",
					x9Type99.cashLetterCount, x9Type99.totalRecordCount, x9Type99.totalItemCount,
					x9Type99.fileTotalAmount, x9Type99.immediateOriginContactName,
					x9Type99.immediateOriginContactPhoneNumber);
		}

		/*
		 * List bundles.
		 */
		X9Object x9o = x9objectManager.getFirst();
		while (x9o != null) {
			if (x9o.isBundleHeader()) {
				final X9Type20 x9Type20 = new X9Type20(x9o);
				LOGGER.info(
						"bundle header:  collectionTypeIndicator({}) destinationRoutingNumber({}) "
								+ "eceInstitutionRoutingNumber({}) bundleBusinessDate({}) "
								+ "bundleCreationDate({}) bundleIdentifier({}) "
								+ "bundleSequenceNumber({}) cycleNumber({}) "
								+ "returnLocationRoutingNumber({}) bundleCreationTime({})",
						x9Type20.collectionTypeIndicator, x9Type20.destinationRoutingNumber,
						x9Type20.eceInstitutionRoutingNumber, x9Type20.bundleBusinessDate,
						x9Type20.bundleCreationDate, x9Type20.bundleIdentifier,
						x9Type20.bundleSequenceNumber, x9Type20.cycleNumber,
						x9Type20.returnLocationRoutingNumber, x9Type20.bundleCreationTime);
				final List<X9Item937> itemList = listBundleContent(x9o);
				LOGGER.info("bundle item Count({})", itemList.size());
			}
			x9o = x9o.getNext();
		}

		/*
		 * Set trailer record counts and amounts when present.
		 */
		if (x9Type99 == null) {
			trailerCashLetterCount = 0;
			trailerTotalRecordCount = 0;
			trailerTotalItemCount = 0;
			trailerFileTotalAmount = BigDecimal.ZERO;
		} else {
			trailerCashLetterCount = X9Numeric.toInt(x9Type99.cashLetterCount);
			trailerTotalRecordCount = X9Numeric.toInt(x9Type99.totalRecordCount);
			trailerTotalItemCount = X9Numeric.toInt(x9Type99.totalItemCount);
			trailerFileTotalAmount = X9Decimal.getAsAmount(x9Type99.fileTotalAmount);
		}
	}

	/**
	 * List bundle content.
	 * 
	 * @param bundleHeader
	 *            current bundle header
	 * @return item list for this bundle
	 */
	private List<X9Item937> listBundleContent(final X9Object bundleHeader) {
		/*
		 * Abort if the bundle header is not as expected.
		 */
		if (bundleHeader == null) {
			throw new X9Exception("currentBundle unexpectedly null");
		}

		if (!bundleHeader.isBundleHeader()) {
			throw new X9Exception("currentBundle is incorrect recordType({})",
					bundleHeader.x9ObjType);
		}

		/*
		 * Accumulate all items within the provided bundle.
		 */
		final List<X9Item937> itemList = new ArrayList<>();
		int itemCount = 0;
		int debitCount = 0;
		int creditCount = 0;
		BigDecimal debitAmount = BigDecimal.ZERO;
		BigDecimal creditAmount = BigDecimal.ZERO;
		X9Object x9o = bundleHeader.getNext();

		/*
		 * Count by transaction type.
		 */
		while (x9o != null && (x9o.isItem() || x9o.isAddendum())) {
			if (x9o.isItem()) {
				itemCount++;
				if (x9o.isDebit()) {
					debitCount++;
					debitAmount = debitAmount.add(x9o.getItemAmount());
				} else if (x9o.isCredit()) {
					creditCount++;
					creditAmount = creditAmount.add(x9o.getItemAmount());
				}

				/*
				 * Add to item list for this bundle.
				 */
				itemList.add(new X9Item937(x9o));
			}

			/*
			 * Get next record.
			 */
			x9o = x9o.getNext();
		}

		/*
		 * List actual totals for items within this bundle.
		 */
		LOGGER.info(
				"bundle content: itemCount({}) debitCount({}) creditCount({}) debitAmount({}) "
						+ "creditAmount({})",
				itemCount, debitCount, creditCount, debitAmount, creditAmount);

		/*
		 * Accumulate actual data totals, which can be contrasted to the trailer records.
		 */
		fileBundleCount++;
		fileItemCount += itemCount;
		fileDebitCount += debitCount;
		fileCreditCount += creditCount;
		fileDebitAmount = fileDebitAmount.add(debitAmount);
		fileCreditAmount = fileCreditAmount.add(creditAmount);

		/*
		 * List the bundle trailer when it exists.
		 */
		final X9Object bundleTrailer = x9o != null && x9o.isBundleTrailer() ? x9o : null;
		final X9Type70 x9Type70 = bundleTrailer != null ? new X9Type70(x9o) : null;
		if (x9Type70 != null) {
			LOGGER.info(
					"bundlebtrailer: itemCount({}) itemAmount({}) micrValidAmount({}) "
							+ "imageCount({}) creditCount({}) creditAmount({})",
					x9Type70.itemCount, x9Type70.itemAmount, x9Type70.micrValidAmount,
					x9Type70.imageCount, x9Type70.creditCount, x9Type70.creditAmount);
		}

		/*
		 * Return the list of items for this bundle.
		 */
		return itemList;
	}

	/**
	 * List all validation errors to the system log.
	 */
	private void listErrorsToLog() {
		final X9ErrorManager x9errorManager = sdkBase.getErrorManager();
		final Map<String, ArrayList<X9Error>> errorMap = x9errorManager.getErrorMap();
		if (errorMap.size() == 0) {
			LOGGER.info("no validation errors");
		} else {
			final StringBuilder sb = new StringBuilder();
			for (final Entry<String, ArrayList<X9Error>> entry : errorMap.entrySet()) {
				final ArrayList<X9Error> x9errors = entry.getValue();
				for (final X9Error x9error : x9errors) {
					sb.setLength(0);
					sb.append("errorKey(").append(entry.getKey());
					sb.append(") recordNumber(").append(x9error.getRecordNumber());
					sb.append(") recordType(").append(x9error.getRecordType());
					sb.append(") errorName(").append(x9error.getErrorName());
					sb.append(") severity(").append(x9error.getSeverity());
					sb.append(") errorFieldName(").append(x9error.getFieldName());
					sb.append(") errorMessage(").append(x9error.getCollectiveText());
					sb.append(") errorComments(").append(x9error.getComments());
					sb.append(")");
					LOGGER.debug(sb.toString());
				}
			}
		}
	}

	/**
	 * Modify a record and write to an output file. New hash totals will be computed automatically.
	 * 
	 * @param outputFile
	 *            output file to be written
	 */
	private void modifysRecordAndWrite(final File outputFile) {
		/*
		 * Change the amount on the first item and completely remove the second item.
		 */
		int modificationCount = 0;
		X9Object x9o = sdkBase.getFirstObject();
		while (x9o != null && modificationCount < 2) {
			if (x9o.isRecordType(X9.CHECK_DETAIL)) {
				/*
				 * Modify the amount on this item. If this file contains an offset, then that total
				 * will not be changed by this logic, and the file will now become out of balance.
				 */
				if (modificationCount == 0) {
					modificationCount++;
					LOGGER.info("modify recordNumber({}) recordType({}) dataBfore({})",
							x9o.x9ObjIdx, x9o.x9ObjType, new String(x9o.x9ObjData));
					final X9Type25 t25 = new X9Type25(x9o);
					t25.amount = "8888";
					t25.modify();
					LOGGER.info("modify recordNumber({}) recordType({}) dataAfter({})",
							x9o.x9ObjIdx, x9o.x9ObjType, new String(x9o.x9ObjData));
				} else if (modificationCount == 1) {
					/*
					 * Remove this type 25 record, which similarly puts the file out of balance when
					 * it has an offsetting debit or credit.
					 */
					modificationCount++;
					LOGGER.info("delete recordNumber({}) recordType({}) dataBfore({})",
							x9o.x9ObjIdx, x9o.x9ObjType, new String(x9o.x9ObjData));
					x9o.setDeleteStatus(true);

					/*
					 * Now remove the type 26-52 addenda records that are attached to this type 25.
					 */
					x9o = x9o.getNext();
					while (x9o != null && x9o.isAddendum()) {
						LOGGER.info("delete recordNumber({}) recordType({}) addenda({})",
								x9o.x9ObjIdx, x9o.x9ObjType, new String(x9o.x9ObjData));
						x9o.setDeleteStatus(true);
						x9o = x9o.getNext();
					}
				}
			}
			x9o = x9o.getNext();
		}

		/*
		 * Update hash counts in the batch trailer and file trailer records.
		 */
		final X9TrailerManager937 x9TrailerManager = new X9TrailerManager937(sdkBase);
		x9o = sdkBase.getFirstObject();
		while (x9o != null) {
			x9TrailerManager.accumulateAndPopulate(x9o);
			x9o = x9o.getNext();
		}

		/*
		 * Ensure we modified records as expected.
		 */
		if (modificationCount == 2) {
			/*
			 * Create a temporary file which will be renamed on completion.
			 */
			final X9TempFile x9tempFile = new X9TempFile(outputFile);

			/*
			 * Write to the temporary output file.
			 */
			writeToFile(x9tempFile.getTemp());

			/*
			 * Rename the output file to final.
			 */
			x9tempFile.renameTemp();
		} else {
			/*
			 * Records not modified as expected.
			 */
			throw new X9Exception("unexpected modificationCount({})", modificationCount);
		}
	}

	/**
	 * Write records to an output file.
	 * 
	 * @param outputFile
	 *            output file to be written
	 */
	private void writeToFile(final File outputFile) {
		/*
		 * Initialize counters.
		 */
		int debitCount = 0;
		int creditCount = 0;
		int recordsWritten = 0;
		BigDecimal debitTotal = BigDecimal.ZERO;
		BigDecimal creditTotal = BigDecimal.ZERO;

		/*
		 * Set sdk writer options.
		 */
		final boolean isEbcdic = true;
		final X9SdkObjectFactory x9sdkObjectFactory = sdkBase.getSdkObjectFactory();
		x9sdkObjectFactory.setIsOutputEbcdic(isEbcdic);
		x9sdkObjectFactory.setFieldZeroInserted(true);

		/*
		 * Write to an output x9.37 file.
		 */
		try (final X9SdkIO sdkIO = sdk.getSdkIO()) {
			/*
			 * Open output as a file. Note that sdkIO also has methods "openOutputStream()" and
			 * "openOutputFile()" that can be used based on your requirements.
			 */
			final ByteArrayOutputStream outputStream = new ByteArrayOutputStream();
			sdkIO.openOutputStream(outputStream, "outputStream"); // open as an output stream
			// sdkIO.openOutputFile(outputFile); // open as an output file

			/*
			 * Walk the list and write each x9.
			 */
			X9Object x9o = sdkBase.getFirstObject();
			while (x9o != null) {
				/*
				 * Create the sdkObject.
				 */
				final X9SdkObject sdkObject = sdkIO.makeOutputRecord(x9o);

				/*
				 * Increment counts and amounts.
				 */
				if (x9o.isDebit()) {
					debitCount++;
					debitTotal = debitTotal.add(x9o.getRecordAmount());
				} else if (x9o.isCredit()) {
					creditCount++;
					creditTotal = creditTotal.add(x9o.getRecordAmount());
				}

				/*
				 * Write this x9.37 record from the sdkObject.
				 */
				recordsWritten++;
				sdkIO.writeOutputFile(sdkObject);

				/*
				 * Get the next x9.37 record.
				 */
				x9o = x9o.getNext();
			}

			/*
			 * Log our statistics.
			 */
			LOGGER.info(sdkIO.getSdkStatisticsMessage(outputFile));

			/*
			 * Copy the output stream to our output file.
			 */
			writeOutputStreamToFile(outputStream, outputFile);

			/*
			 * Write summary message to the log.
			 */
			final String characterSet = isEbcdic ? X9C.EBCDIC : X9C.ASCII;
			LOGGER.info(
					"outputFile({}) characterSet({}) debitCount({}) debitTotal({}) "
							+ "creditCount({}) creditTotal({}) recordsWritten({})",
					outputFile.toString(), characterSet, X9D.formatLong(debitCount),
					X9D.formatBigDecimal(debitTotal), X9D.formatLong(creditCount),
					X9D.formatBigDecimal(creditTotal), X9D.formatLong(recordsWritten));
		} catch (final Exception ex) {
			throw new X9Exception(ex);
		}
	}

	/**
	 * Write the byte array output stream to an output file.
	 * 
	 * @param outputStream
	 *            constructed byte array output stream
	 * @param outputFile
	 *            output file to be written
	 * @param isAppendPadRecords
	 *            true if pad records are to be appended
	 */
	private void writeOutputStreamToFile(final ByteArrayOutputStream outputStream,
			final File outputFile) {
		try {
			outputStream.flush();
			outputStream.close();
			X9FileIO.writeFile(outputStream.toByteArray(), outputFile);
			LOGGER.info("output written({})", outputFile);
		} catch (final Exception ex) {
			throw new X9Exception(ex);
		}
	}

	/**
	 * Main().
	 *
	 * @param args
	 *            command line arguments
	 */
	public static void main(final String[] args) {
		int status = 0;
		X9JdkLogger.initializeLoggingEnvironment();
		LOGGER.info(X9VERIFYX9 + " started");
		try {
			final X9VerifyX9 x9verify = new X9VerifyX9();
			x9verify.process();
		} catch (final Throwable t) { // catch both errors and exceptions
			status = 1;
			LOGGER.error("main exception", t);
		} finally {
			X9SdkRoot.shutdown();
			X9JdkLogger.closeLog();
			System.exit(status);
		}
	}

}
```

### 2. Fluent (exactly per charter)

Charter conventions strictly applied: dialect-concrete return types from static factories, `fromFile(File)` source, single terminal verb `run`, builder block constructing `X9SdkApplication` as `AutoCloseable`. No Spring. Validate and modify expressed as separate Engine pipelines composed in customer code.

```java
package com.x9ware.examples;

import java.io.File;
import java.math.BigDecimal;
import java.util.concurrent.atomic.AtomicInteger;

import org.slf4j.Logger;

import com.x9ware.application.X9SdkApplication;
import com.x9ware.engines.X9ModifyEngine;
import com.x9ware.engines.X9ModifyEngine937;
import com.x9ware.engines.X9ValidateEngine;
import com.x9ware.engines.X9ValidateEngine937;
import com.x9ware.logging.X9LoggerFactory;
import com.x9ware.summaries.X9ModifySummary;
import com.x9ware.summaries.X9ValidationResult;

public final class X9VerifyX9 {

    private static final Logger LOGGER = X9LoggerFactory.getLogger(X9VerifyX9.class);

    private X9VerifyX9() {
    }

    public static void main(final String[] args) {
        if (args.length != 3) {
            LOGGER.error("usage: X9VerifyX9 <licenseKey> <input.x9> <output.x9>");
            System.exit(1);
        }
        final String licenseKey = args[0];
        final File inputFile = new File(args[1]);
        final File outputFile = new File(args[2]);

        try (X9SdkApplication app = X9SdkApplication.builder()
                .applicationName("X9VerifyX9")
                .licenseKey(licenseKey)
                .build()) {

            final X9ValidateEngine937 validator = X9ValidateEngine.x937(app);
            final X9ValidationResult result = validator
                    .fromFile(inputFile)
                    .validateTiffImages(true)
                    .run();
            LOGGER.info("validation: errorCount({}) severity({})",
                    result.getErrorCount(), result.getRunSeverity());

            final AtomicInteger itemIndex = new AtomicInteger(0);
            final X9ModifyEngine937 modifier = X9ModifyEngine.x937(app);
            final X9ModifySummary modifySummary = modifier
                    .fromFile(inputFile)
                    .toFile(outputFile)
                    .transform(item -> {
                        int index = itemIndex.getAndIncrement();
                        if (index == 0) {
                            item.setAmount(new BigDecimal("88.88"));
                            return item;
                        }
                        if (index == 1) {
                            return null;
                        }
                        return item;
                    })
                    .run();
            LOGGER.info("modify: modifiedCount({})", modifySummary.getModifiedCount());
        }
    }
}
```

### 3. Fluent (charter + x9SdkApi design language)

Same Engine grammar and same business logic as form 2, refined by the design moves this vision adds: `X9SdkApplication` typed as the public interface; `var` hides dialect-concrete locals (the customer never types `X9ValidateEngine937`); `Path` source instead of `File` (cloud-friendly, `Resource`-compatible); the Engine chain runs inline (no separate `validator` / `modifier` capture, since the dialect concrete never needs to appear in a customer-typed local). Still no Spring — the customer constructs the application in a small builder block.

```java
package com.x9ware.examples;

import java.math.BigDecimal;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicInteger;

import org.slf4j.Logger;

import com.x9ware.application.X9SdkApplication;
import com.x9ware.engines.X9ModifyEngine;
import com.x9ware.engines.X9ValidateEngine;
import com.x9ware.logging.X9LoggerFactory;

public final class X9VerifyX9 {

    private static final Logger LOGGER = X9LoggerFactory.getLogger(X9VerifyX9.class);

    private X9VerifyX9() {
    }

    public static void main(final String[] args) {
        if (args.length != 3) {
            LOGGER.error("usage: X9VerifyX9 <licenseKey> <input.x9> <output.x9>");
            System.exit(1);
        }
        final String licenseKey = args[0];
        final Path inputPath = Path.of(args[1]);
        final Path outputPath = Path.of(args[2]);

        try (var app = X9SdkApplication.builder()
                .applicationName("X9VerifyX9")
                .licenseKey(licenseKey)
                .build()) {

            final var result = X9ValidateEngine.x937(app)
                    .fromPath(inputPath)
                    .validateTiffImages(true)
                    .run();
            LOGGER.info("validation: errorCount={} severity={}",
                    result.getErrorCount(), result.getSeverity());

            final var itemIndex = new AtomicInteger(0);
            final var modifySummary = X9ModifyEngine.x937(app)
                    .fromPath(inputPath)
                    .toPath(outputPath)
                    .transform(item -> {
                        int index = itemIndex.getAndIncrement();
                        if (index == 0) {
                            item.setAmount(new BigDecimal("88.88"));
                            return item;
                        }
                        if (index == 1) {
                            return null;
                        }
                        return item;
                    })
                    .run();
            LOGGER.info("modify: modifiedCount={}", modifySummary.getModifiedCount());
        }
    }
}
```

### 4. Modern Spring

Form 3 with the same business logic, but Spring AutoConfiguration takes over the parts of the program that are no longer the customer's concern: the `main()` driver (Spring Boot starts the application), the argument-parsing block (configuration is externalized to `application.yml` properties or method parameters), the `X9SdkApplication.builder()` setup (AutoConfiguration constructs and wires the implementation), and the `try-with-resources` lifecycle (the container's `@PreDestroy` and shutdown hooks close the application). What remains is the actual operation: a `@Service` method callable from anywhere in the Spring Boot component graph.

```java
package com.x9ware.examples;

import java.math.BigDecimal;
import java.nio.file.Path;
import java.util.concurrent.atomic.AtomicInteger;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import com.x9ware.application.X9SdkApplication;
import com.x9ware.engines.X9ModifyEngine;
import com.x9ware.engines.X9ValidateEngine;

@Service
public final class X9VerifyX9 {

    private static final Logger LOGGER = LoggerFactory.getLogger(X9VerifyX9.class);

    @Autowired X9SdkApplication x9;

    public void verify(final Path inputPath, final Path outputPath) {
        final var result = X9ValidateEngine.x937(x9)
                .fromPath(inputPath)
                .validateTiffImages(true)
                .run();
        LOGGER.info("validation: errorCount={} severity={}",
                result.getErrorCount(), result.getSeverity());

        final var itemIndex = new AtomicInteger(0);
        final var modifySummary = X9ModifyEngine.x937(x9)
                .fromPath(inputPath)
                .toPath(outputPath)
                .transform(item -> {
                    int index = itemIndex.getAndIncrement();
                    if (index == 0) {
                        item.setAmount(new BigDecimal("88.88"));
                        return item;
                    }
                    if (index == 1) {
                        return null;
                    }
                    return item;
                })
                .run();
        LOGGER.info("modify: modifiedCount={}", modifySummary.getModifiedCount());
    }
}
```

### 5. Facade-shaped (for completeness)

An earlier modernization sketch where Engines are fully abstracted behind the facade. The customer reads a `X9File937` from the `X9Context` and operates on it directly via methods like `.validate(...)` and `.write(...)`. Compact in trivial cases, but loses Engine discoverability and pipeline composability — the design opposite extreme of form 1, included so the journey is visible.

```java
package com.x9ware.examples;

import java.io.IOException;

import org.slf4j.Logger;

import com.x9ware.X9Context;
import com.x9ware.X9LogManager;
import com.x9ware.exceptions.X9LicenseException;
import com.x9ware.io.X9Format;
import com.x9ware.x937.X9File937;
import com.x9ware.x937.X9Money;
import com.x9ware.x937.X9ValidationOptions;
import com.x9ware.x937.X9ValidationReport;
import com.x9ware.x937.X9WriteReport;

/**
 * Open an x9.37 file, validate it, modify two records, and write a new file.
 *
 * @author X9Ware LLC. Copyright(c) 2012-2026 X9Ware LLC. All Rights Reserved.
 */
public final class X9VerifyX9 {

    private static final Logger LOGGER = X9LogManager.loggerFor(X9VerifyX9.class);

    private X9VerifyX9() {
    }

    public static void main(final String[] args) throws IOException, X9LicenseException {
        if (args.length != 3) {
            LOGGER.error("usage: X9VerifyX9 <license.xml> <input.x9> <output.x9>");
            System.exit(1);
        }
        final String licensePath = args[0];
        final String inputPath = args[1];
        final String outputPath = args[2];

        final X9ValidationOptions validationOptions = X9ValidationOptions.builder()
                .applyAbaFrbDistrictValidation(true)
                .build();

        try (final X9Context x9Context = X9Context.fromLicenseFile(licensePath)) {
            final X9File937 x9File = x9Context.read937(inputPath);

            final X9ValidationReport validationReport = x9File.validate(validationOptions);
            LOGGER.info("validated: {}", validationReport.summary());

            x9File.items().first().setAmount(X9Money.usd("88.88"));
            x9File.items().remove(1);

            final X9WriteReport writeReport = x9File.write(outputPath, X9Format.EBCDIC);
            LOGGER.info("wrote: {}", writeReport.summary());
        }
    }
}
```

## Related documents

- `sdk-fluent-api-style-charter-design.md` (April 28, 2026) — the fluent API charter this vision builds on
- `sdk-fluent-api-design.md` (April 27, 2026) — the parent design document the charter implements
- `sdk-fluent-api-charter-review.md` (April 28, 2026) — independent technical review of the charter, foundational input to this vision
