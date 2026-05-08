# X9Ware SDK API — Design Vision

X9Ware LLC • May 8, 2026

## Status

**DRAFT — under active design.** This document is a vision, not a charter. It articulates the destination and the principles that lead the design there, and explicitly does not pin specific class signatures, method names, or implementation sequencing.

## Working Notes — Synthesis as of May 8, 2026 (NOT part of the final vision; captures discussion state for the upcoming rewrite)

These notes record the position reached through discussion of Topic 1 (Spring positioning) and Topic 2 (charter overview) before Topic 3 (substrate review) begins. They will be folded into the rewrite of this document and removed from the published version. Authoring constraints follow at the end.

### Audience and document length

- The primary reader is internal. Calibrate vocabulary to a reader with limited exposure to broader Java community jargon; concrete code carries more weight than abstract terminology. The 4/28 internal email named the modernization themes (interface-based design, JavaBeans, observability hooks, modern threading, Spring-ecosystem compatibility, containerization), so the proposal is not landing cold.
- The 4/28 email invited "a better idea entirely" and a later message invited critique of the Engine-based design. Direct critique is welcome; deference is not.
- Target length: 4–8 pages, ~6 pages preferred. Executive summary up front serves as the under-a-page elevator pitch. Body earns the rest of the length by adding substance the summary cannot carry.
- Calibrate vocabulary to the reader; explain through before/after code rather than through framework conventions that may not be familiar. Keep "ceremony" — the term has been requested.

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

**Lifecycle invisible to the customer.** `X9SdkApplication` exists as the lifecycle root, but customers do not see it in their code. In Spring Boot, AutoConfiguration constructs and wires it as a bean; the customer just `@Autowired`s a bean (typically by interface) and uses it. In plain Java, a small builder block constructs it once and keeps it for the application's lifetime; it does not appear in per-operation code paths. The `X9SdkApplication` concept is real and necessary — but it is the implementation, not the customer's mental model.

**Observability seams.** SLF4J for logs (already in place across x9Sdk runtime). Micrometer interfaces for metrics (the customer's existing meter registry is honored). OpenTelemetry-compatible spans on long-running operations. The customer's observability stack — Prometheus, Datadog, New Relic, anything else — works without us picking the backend.

**Pipeline composability.** Read → validate → modify → write composes as a chained operation. This is a place where the fluent grammar and the modern API design language reinforce each other rather than compete; the chain reads cleanly and demos cleanly to prospects evaluating the SDK.

**Modern threading and container-friendliness.** Engines do not hold thread-locals or static caches. Per-task state lives on instances the customer creates (or that Spring scopes). Virtual threads (Java 21+) work without additional configuration. Container lifecycle (Spring `@PreDestroy`, Kubernetes pod shutdown) closes resources cleanly via `AutoCloseable`.

## A worked example

Modern Spring Boot service, processing an uploaded check file:

```java
@Service
public class CheckProcessor {
    @Autowired X9SdkApplication x9;  // wired by Spring AutoConfiguration

    public X9WriteSummary process(InputStream upload, X9HeaderXml937 header) {
        return X9WriteEngine.x937(x9)
            .fromStream(upload)
            .header(header)
            .toBytes()
            .run();
    }
}
```

Five lines including the method signature. No setup code. No license registration. No XML configuration loading. No dialect bind. No image folder setting. No trailer-repair flag. No `try-with-resources` around an SdkIO. The customer types the Engine and the operation; everything else is handled by Spring AutoConfiguration plus the Engine's defaults.

What this replaces: a roughly seventy-line preamble that opens every example today, plus a fifteen-to-twenty-line imperative I/O loop with manual trailer recomputation. The compression is real because the modern API absorbs invariant boilerplate that customers literally never vary.

The same Engine accepts a `Path` source instead of a stream, or a programmatic list of items, with no different API shape — just a different builder method (`fromPath`, `fromItems`). Source is configuration on the Engine, not a different API for each source type.

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

## Related documents

- `sdk-fluent-api-style-charter-design.md` (April 28, 2026) — the fluent API charter this vision builds on
- `sdk-fluent-api-design.md` (April 27, 2026) — the parent design document the charter implements
- `sdk-fluent-api-charter-review.md` (April 28, 2026) — independent technical review of the charter, foundational input to this vision
