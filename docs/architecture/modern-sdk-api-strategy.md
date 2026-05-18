# Modern X9Ware SDK API Strategy

X9Ware LLC • May 2026

## Status

**DRAFT — under active development.** This strategy describes a modern X9Ware SDK API surface, the structural decisions that shape the work, and the principles that guide subsequent design. Specific class signatures, method names, and implementation sequencing are out of scope and belong to the design documents that follow.

## Vision

The modern X9Ware SDK API aims to meet the expectations enterprise Java developers have of a modern library — operations matching how they think about their work, inputs accepted from any reasonable source, simplicity and conventions consistent with the rest of the Java ecosystem, and clean integration with their existing tools.

X9Ware's competitive advantage lives in what we ship, not how fast we ship it. AI tooling has flattened the speed at which any team can produce incremental features; differentiation now comes from the choices we make about what we build. This strategy describes the shape of a modern X9Ware SDK API and the structure for delivering it.

## Executive summary

X9Ware should add a modern Java API to the existing SDK. The legacy `x9Sdk` direct-construction surface continues unchanged for the eighteen customers built on it; the modern API surface lands in the same `x9Sdk` artifact and becomes what prospects evaluate when comparing X9Ware against commercial and open-source alternatives.

The strategic case is **new customer acquisition**. Attracting developers to our SDK means meeting modern Java expectations — and Spring ecosystem compatibility is table stakes, with roughly nine of ten Fortune 500 companies running Java on Spring Boot. AI-armed competitors can match incremental moves quickly, so differentiation has to live in *what we ship*, not how fast we ship it.

The modern API surface aims for that result. Engines (verb-shaped operations like `X9ValidateEngine`, `X9WriteEngine`) and the `X9SdkApplication` lifecycle root become the customer-facing surface, shaped by the design language modern Java APIs follow: Engine-centric, source-agnostic (file, stream, programmatic items uniformly), interface-based, JavaBean-conventional, observability-ready, Spring-friendly, container-friendly. The same operation runs in 84 lines of plain Java, or 77 lines as a Spring Boot `@Service` consumed through the starter — down from 760 lines in the legacy direct-construction style today. The verbose setup block every example opens with today — license registration, configuration loading, dialect binding, and SDK options — collapses to a small builder block in plain Java and to zero customer code in Spring Boot.

## Strategic position

Four operating principles shape the strategy.

**Think big, deliver big.** What we build has to be worth aiming for. Incremental polish on the existing SDK is not enough; the goal is an API that developers actively want to use.

**Differentiation lives in what we ship, not how fast.** AI lets X9Ware build anything we can dream of — and it lets competitors do the same. The advantage is no longer in shipping faster; it is in shipping something materially better than what an open-source integration framework or a commercial payment file-processing SDK ships. The bar is high because the competition's bar is rising too.

**The new customer acquisition lens.** The eighteen current x9Sdk customers came in over the past decade and use the legacy direct-construction surface. Their tech stacks reflect that era; they are not the primary demographic this strategy targets. The audience the modern API aims at is the new prospect a sales conversation surfaces tomorrow — potentially Cloud, Spring Boot, container-deployed, modernization-aware. That prospect evaluates the SDK by feel before they ever write a line of integration code; the API has to read as something they would have built themselves.

**The Spring ecosystem has changed developer expectations.** Spring Boot remains the dominant backend framework for enterprise Java development, powering approximately 60% of enterprise backends, and nearly 92% of Fortune 100 companies still rely on the Java ecosystem for mission-critical operations. Quarkus and Micronaut exist but are bounded — their differentiators (cold-start time, memory footprint) do not apply to file-processing workloads. Spring ecosystem compatibility is table stakes today. For X9Ware's market, supporting Spring through an opt-in companion artifact is a low-risk decision with high upside — and the SDK API itself should reflect the qualities developers value in Spring: clear conventions, sensible defaults, minimal ceremony, and clean substitution.

## The proposal — modern API delivered in x9Sdk

The customer-facing end state is a single SDK distribution:

- **`x9Sdk`** carries both the **legacy direct-construction surface** that the eighteen existing customers depend on and the **modern API surface** (Engines, `X9SdkApplication`, fluent grammar, interface-based types, observability seams). Java 1.8 floor. The two surfaces coexist within one SDK, in the spirit of the IBM-CICS dual-API model that kept macro-level and command-level APIs together in one product. Customers choose whichever surface fits their integration, and the legacy surface continues exactly as it is for the customers depending on it.
- An optional **Spring Boot starter** (`x9-sdk-spring-boot-starter`) ships separately as a companion library, providing zero-ceremony Spring integration for customers who want it. The starter depends on x9Sdk; it is not required for the modern API surface itself.

**Development happens in a temporary `x9SdkApi` repository.** During the development phase, the modern API surface is built in a separate `x9SdkApi` repository so the experimental work does not touch x9Sdk's production codebase. Before the first customer release of the modern API surface, `x9SdkApi`'s content is merged into `x9Sdk` with git history preserved. Customers never see `x9SdkApi` as a separate library — it is purely internal scaffolding. The merge procedure is documented in the runbook referenced under *Path forward*.

**Packaging within x9Sdk uses a flat package tree.** Modern API classes and existing classes coexist in functional packages; there is no top-level `com.x9ware.api.*` namespace separating modern from legacy. The modern API surface is identified by documentation, the User Guide, and example programs, not by package-naming convention. A `com.x9ware.internal.*` namespace was rejected because moving existing classes into it would break the eighteen existing customers' imports.

### JDK floor for the modern API surface

**Decision: Java 1.8.** x9Sdk's existing Java 1.8 floor is preserved; the modern API surface compiles and runs at the same level. This section answers whether the *modern* surface is reachable from Java 8. It is, intentionally — the absence of a dramatic-better case for upgrading outweighs the customer-segment cost of excluding Java 1.8 shops from the modern API.

Candidates considered:

| Floor | What it would enable in our own code | Excluded segment |
|---|---|---|
| **Java 1.8** (chosen) | Matches x9Sdk's existing floor; every current customer can adopt the modern API surface | — |
| **JDK 11** | `var`, modules, `java.net.http`, text blocks (JDK 13 backport varies) | Java 1.8 holdouts on long-term-support contracts |
| **JDK 17** | Records, sealed types, pattern-matching `instanceof`, text blocks, switch expressions; aligns with Spring Boot 3 starter floor | Java 1.8 + JDK 11 LTS shops |
| **JDK 21** | Virtual threads, record patterns, switch patterns; aligns with likely Spring Boot 4 floor | Narrower still; threading is not typically a concern at the SDK API boundary |
| **JDK 25** | Latest LTS; signals "modern" by being current; adds scoped values, stream gatherers, further pattern-matching refinements | Narrowest segment; meaningful adoption pace lags LTS by 12–24 months |

Customer call-site features — `var`, pattern-matching `instanceof`, switch expressions, text blocks in their own code, virtual threads in their own runtime — work at any JDK the customer chooses, regardless of our compiled floor. The features that require *us* to upgrade are the ones we ship in our public surface.

**Explicitly traded away.** Two API-design moves have no Java 1.8 equivalent and are worth naming as the cost of this choice:

- **Sealed result hierarchies with exhaustive `switch`** (JDK 17). An interface plus package-private constructors approximates the closed-subtype invariant; nothing approximates the compile-time exhaustiveness check at the customer call site. Validation outcomes (`Clean | Warnings | Errors`) stay JavaBean-shaped with an `if (result.getSeverity().isError())` idiom rather than an exhaustive switch.
- **Module-enforced public API boundary** (JDK 9+). The convention-based "internal" marker (package naming, `@apiNote`, Javadoc warnings) remains the only mechanism keeping customers out of implementation types. JPMS would enforce the boundary at compile time so a customer reaching for an internal class would get a compile error rather than a runtime surprise.

Records-as-return-types can be approximated with verbose final classes; the customer-side difference is destructuring syntax in `switch`, which the JavaBean idiom approximates with `instanceof` + getter chains. Virtual threads (JDK 21+) similarly stay out of our internal implementation; customer applications running on a JDK 21+ runtime get the benefit of virtual threads regardless of our compile target.

This decision is revisitable as the Java 1.8 customer segment shrinks. JDK 17 floor + JPMS modules + sealed result types is a coherent forward-looking package — recorded here so the option is preserved rather than rediscovered.

## Pillars of a modern X9Ware SDK API

Our north star is the "would you write this yourself" test. The pillars below name what passing that test actually requires of a modern SDK. Each is a foundation the surface rests on; together they produce the coherent experience a prospect expects. Each pillar is independently meaningful but does not carry the API on its own — the strength comes from their alignment.

**Ease of use.** The customer should reach the result quickly, with minimal ceremony and minimal boilerplate the SDK can manage internally. Three design moves carry this pillar:

- **Engine-centric fluent API.** The customer's first interaction is with an Engine — `X9ValidateEngine`, `X9WriteEngine`, `X9ReadEngine` — not with a file, a facade method, or a utility-shaped wrapper. Source (file, stream, programmatic items) is configuration on the Engine; the Engine's lifecycle is build → configure → run; the result is a typed summary. The fluent grammar makes the chain read as a single intentional act rather than as a sequence of imperative steps. The detailed design of the Engine factory pattern, terminal `.run()` operation, source/sink builder methods, and conventions every Engine follows lives in the fluent API design referenced under *Related documents*.
- **Spring Boot zero-ceremony onboarding.** A customer using the optional `x9-sdk-spring-boot-starter` gets `X9SdkApplication` autowired and lifecycle managed through `@PreDestroy` — no builder block, no try-with-resources, no explicit shutdown. The Engine pipeline that runs in 84 lines of plain Java collapses to roughly 77 lines as a Spring Boot `@Service`.
- **Verbose setup absorbed by the SDK.** Boilerplate that today opens every example — license registration, configuration loading, dialect binding, SDK options — is internalized in the builder (or the starter), out of customer code.

**Source-agnostic by construction.** The API accepts input from any source type uniformly: `Path`, `InputStream`, Spring `Resource`, programmatic items. No type in the public surface forces customers through `java.io.File`. The current `X9File` extends `java.io.File` and encodes that constraint into the type system; modern x9SdkApi cannot. Cloud storage, Resource abstractions, and stream-only inputs all become first-class without API duplication.

**Customer-facing type design.** Customer-facing types on the modern API surface follow modern Java conventions consistently:

- **JavaBean accessors throughout.** No public fields; predictable `get*` / `set*` / `is*` naming on result objects (`X9WriteSummary`, `X9ReadSummary`, `X9ValidationResult`), configuration types, item classes (`X9Item937`, `X9ItemAch`, etc.), and `X9SdkApplication`. Legacy x9Sdk classes carry historical inconsistencies — some expose public fields directly — and stay where they are for the eighteen existing customers, but no new type on the modern surface inherits the inconsistency.
- **Interfaces, not concrete classes, for major types.** Engines and the lifecycle root expose Java interfaces, not just concrete classes, for the public API. This decouples customer code from internal implementation evolution — we can change concrete implementations, add proxies, or swap strategies without breaking the customer. Spring Framework consumers get clean `@Bean` substitution; test code gets clean mocking and stubbing. Plain `new` construction loses nothing.
- **Typed exceptions for predictable error handling.** Each customer-facing failure mode that warrants programmatic dispatch is its own exception type, so customer code can catch specifically what it can recover from rather than catching a single sweeping type. Public exceptions are unchecked (`RuntimeException` subclasses) by default for caller simplicity; checked exceptions are used rarely, only when the calling application is expected to handle the specific situation. The modern surface introduces typed exceptions at the new Engine boundary; legacy paths in the underlying SDK continue to throw a single generic exception type as they do today.

The `X9Object` internal byte-array representation stays inside the implementation; customers work with the typed item classes and never reach for `X9Object` directly.

**Sensible defaults with opt-out for control.** Modern enterprise customers expect their SDK dependencies to "just work" without explicit lifecycle ceremony. Spring Boot autoconfiguration, HikariCP's internal lifecycle, Netty's graceful shutdown, the JDK's `HttpClient` — none of these require customers to write start or shutdown code. The X9Ware SDK takes the same position: `X9SdkApplication` is the lifecycle root, exposed as a Java interface implementing `AutoCloseable`, and the SDK handles cleanup automatically in the common case. Customers who need explicit control reach for it through the same `AutoCloseable` contract and a small set of opt-out APIs.

The customer-facing scenarios:

| Scenario | Customer ceremony | How cleanup happens |
|---|---|---|
| Spring Boot via the starter | None | The `x9-sdk-spring-boot-starter` contributes `X9SdkApplication` as an `@Bean` and wires `@PreDestroy` to `close()`. The starter internally disables the JVM shutdown hook because Spring Boot owns shutdown — the customer never has to think about it. |
| Plain Java standalone | None | A JVM shutdown hook (installed by default) calls `close()` at JVM exit. Customer writes no lifecycle code beyond the builder block. |
| Plain Java with scope-bounded determinism | `try-with-resources` (one line) | The try block's exit calls `close()` deterministically. The shutdown hook's call becomes a no-op. |
| Non-Spring framework integration (Quarkus, Micronaut, app server) | `.disableShutdownHook()` plus framework wiring | Framework calls `close()` via its own lifecycle mechanism (Quarkus `@Shutdown`, Micronaut `@PreDestroy`, etc.); the SDK's hook is disabled so it does not compete for ordering. |
| Custom shutdown sequencing (own JVM hook, test orchestration) | `.disableShutdownHook()` plus explicit `close()` | Customer fully controls cleanup timing. |

Three mechanisms underpin the default behavior:

- **JVM shutdown hook** registered at first build (off-able via `.disableShutdownHook()`). Fires on JVM exit if `close()` was not called earlier. Hook is bounded to SDK-owned cleanup, with each step internally tolerant and wrapped in try/catch so a hung worker cannot strand JVM exit.
- **Phantom-reference cleanup** registered at build. Fires when the customer's reference to the application becomes unreachable during JVM lifetime — covers the case where the customer drops the reference but the JVM stays alive (variable reassigned, scope exited without `try-with-resources`). At the Java 1.8 floor, the implementation uses `PhantomReference` + `ReferenceQueue` + a daemon cleanup thread; migrates to `java.lang.ref.Cleaner` if the JDK floor is raised.
- **Explicit `close()`** for deterministic lifecycle. Idempotent — any later auto-cleanup paths see "already closed" and no-op.

Per-engine lifecycle is implicit in `.run()` — Engine resources are acquired and released within the fluent chain's terminal operation. Long-lived readers that yield records lazily, or writers that hold file handles across multiple `addItem` calls, expose the Engine as `AutoCloseable` for explicit wrapping; the default case never sees this.

Use-after-close fails fast: any operation on a closed `X9SdkApplication` throws `IllegalStateException` with a clear message. This catches lifecycle bugs at the call site rather than as silently miscounted statistics or partially-written files downstream.

**Standards-based observability.** The modern API surface emits signals through standard Java observability interfaces — SLF4J for logs (the customer wires their own SLF4J binding; the modern API does not configure logging itself), Micrometer for metrics, and OpenTelemetry-compatible spans on long-running operations. The customer's existing observability stack — Prometheus, Datadog, New Relic, Grafana, anything else — receives those signals because both sides speak the same standards. The SDK does not pick the backend or require the customer to wire up custom hooks; it conforms to the interfaces the customer's infrastructure already consumes. When no observability stack is present (the customer has not configured a `MeterRegistry`, a `Tracer`, or a particular SLF4J binding), the SDK is silently inert on those channels — no logs lost, no metrics buffered, no spans created without a receiver. This is a step beyond legacy x9Sdk, which emits only logs; the modern surface adds metrics and traces as first-class signals when the customer's application provides a `MeterRegistry` and a `Tracer`. Spring Boot customers get this wiring through standard auto-configuration; other customers wire it explicitly, as they would for any modern Java library.

**Modern threading and container-friendliness.** The modern API surface is built to run cleanly in the environments customers actually deploy into. Engines hold no per-customer state in static caches; per-task state lives on instances the customer creates or that the customer's framework scopes. Spring Framework consumers get clean `@Bean` definition and `@Autowired` substitution because the interface-based public surface is DI-friendly by construction; Spring Boot consumers get near-zero ceremony — the customer supplies a license key via configuration, and the `x9-sdk-spring-boot-starter` companion artifact handles the rest, contributing an `@Bean X9SdkApplication` and wiring `@PreDestroy` to `close()` automatically. Customers running on JDK 21+ can call SDK code from virtual threads without pinning concerns — no synchronized blocks on long I/O, no thread-confined state. Container lifecycle (`@PreDestroy` from any container that honors it, Kubernetes pod shutdown via `AutoCloseable`) closes resources cleanly.

### A worked example

The same validation operation in plain Java:

```java
try (X9SdkApplication x9 = X9SdkApplication.builder()
        .applicationName("my-app")
        .licenseKey(LICENSE_KEY)
        .build()) {
    X9ValidationResult result = X9ValidateEngine.x937(x9)
            .fromStream(upload)
            .validateTiffImages(true)
            .run();
}
```

Eight lines including the builder block and `try-with-resources`. No license-registration call, no XML configuration loading, no dialect bind, no image-folder setting, no manual `SdkIO` plumbing. The customer types the application builder, the Engine, and the operation; everything else is handled by the builder's defaults and the Engine's defaults.

What this replaces: a substantial preamble that opens every example today, plus the imperative I/O loop and manual heap walks that follow it.

The same Engine accepts a `Path` source instead of a stream, or a programmatic list of items, with no different API shape — just a different builder method. Source is configuration on the Engine, not a different API for each source type.

### Looking ahead — Engines as operators

The pillars above are what the modern API surface delivers. The Engine design also opens a door this strategy does not require for the foundation to be complete but is worth naming because the Engine shape positions it naturally: **Engines have clean input/output contracts that let them play the role of operators in payment-file workflows.** A read operator emits an X9Item stream from a file or stream; transformation operators (modify, validate, scrub) consume and produce X9Item streams; a sink operator writes the stream into a file or stream. The shape is the pipe-and-filter pattern familiar from shell pipelines and stream-processing frameworks, applied to the natural unit of payment-file processing — the X9Item.

A future direction the modern API surface accommodates is chaining these operators into reusable workflows for complex payment-file operations: file consolidation across distributed capture points, multi-stage processing pipelines, dialect-conversion flows. X9Collector is a concrete candidate — its file-consolidation workflow is a natural composition of Engine operators.

The connective tissue between operators — the streaming protocol, backpressure handling, error semantics, resource lifecycle across stages, workflow runtime — is genuinely complex and is not specified by this strategy. The foundation today supports independent Engine reuse via dependency injection. When and whether to extend the Engine design with a workflow primitive is a subsequent design and implementation decision; the Engine shape does not rule out that direction.

## What is not decided here

Specific decisions deferred to subsequent design work:

- Per-Engine builder method names beyond the universal operations (those belong to the per-Engine designs).
- The exact source-abstraction interface for input and output (a related design follows from this strategy).

## Tradeoffs accepted

Decisions in this strategy carry deliberate tradeoffs. Naming them explicitly invites alignment conversations rather than leaving the costs implicit.

- **Full backward compatibility for x9Sdk.** Legacy public fields, naming inconsistencies, and direct-construction patterns in x9Sdk's existing classes are not modernized. The eighteen existing customers see no changes to the classes they depend on. The modern API surface follows new conventions; the legacy surface stays as it is. The dual-API position absorbs the asymmetry.
- **Java 1.8 floor.** The modern API surface ships at Java 1.8 to match x9Sdk's existing floor and avoid excluding customers on long-term-support Java 8 contracts. The cost: no records, sealed types, JPMS modules, virtual threads, or other post-Java-8 features in our own code.
- **Spring supported but not used.** The modern API surface carries no Spring dependency. Spring integration is available through the optional `x9-sdk-spring-boot-starter` companion artifact. Customers who do not use Spring carry no Spring code; Spring shops get near-zero-ceremony integration via the starter. The cost: the modern API surface cannot use Spring Framework features (IoC container, autoconfiguration, resource abstractions) internally — Spring's value flows to customers who consume it; we do not get to lean on Spring inside the SDK itself.
- **Convention-based API boundary.** Without JPMS modules (a consequence of the Java 1.8 floor), the public-vs-internal boundary relies on package-naming convention, `@apiNote` Javadoc tags, and documentation. A customer reaching into an internal class gets no compile-time block; the discipline is editorial rather than enforced.

## Path forward

1. **Strategy review.** This document, reviewed internally and revised with feedback.
2. **Foundation decisions for the pilot.** Source abstraction, lifecycle integration, observability hooks — written as separate designs where the choice has cross-Engine consequences, settled in-line with the pilot otherwise.
3. **Pilot Engine.** One Engine implemented in the x9SdkApi development repository following the design language. Validates the conventions against real code before they are locked across the surface.
4. **Incremental delivery.** Subsequent Engines, with the User Guide reframe shipping topics in lockstep.
5. **Merge x9SdkApi into x9Sdk** before the first customer release of the modern API surface, preserving x9SdkApi's git history. See [`modern-api-merge-runbook.md`](../projects/modern-api-merge/modern-api-merge-runbook.md) for the procedure.
6. **Spring Boot starter.** Companion artifact that contributes `X9SdkApplication` as an `@Bean` and wires `@PreDestroy` to `close()`, delivered alongside the merged core.

The pilot Engine is the empirical test. If the design language proves brittle in real code, the conventions revise before delivery proceeds. The strategy itself is stable; specific design choices below it (interface shapes, exact method names) are subject to change.

## When to revisit

The strategy is expected to be stable for the modernization timeframe. Specific signals that warrant revision:

- A paying customer surfaces with conflicting needs that x9Sdk cannot serve and x9SdkApi cannot serve in its proposed form.
- The Spring ecosystem shifts materially (Spring Framework 7, Spring Boot 4) in ways that change the foundation decision.
- A competitor ships a payment file-processing SDK that materially redefines what "looking better" means and the strategy's design language no longer matches.
- AI tooling changes the bar for "would you write this yourself" — what counts as "modern" has to keep moving.

## Appendix — Code Examples

This appendix shows the same operation — open an X9.37 file, validate it, modify two records, and write a new output — in two customer-facing examples that illustrate the integration shapes a customer might adopt:

1. **Plain Java** — the Engine pipeline in plain Java. Customer constructs `X9SdkApplication` explicitly via builder, uses Engines, releases via try-with-resources. No Spring dependency.
2. **Spring Boot via the starter** — the same Engine pipeline consumed through the `x9-sdk-spring-boot-starter` companion artifact. Starter provides `X9SdkApplication` as a Spring-managed `@Bean`; Spring Boot handles construction and `@PreDestroy`. Operation code is identical to Example 1.

### Example 1 — Plain Java

The canonical Engine pipeline in plain Java. `X9SdkApplication` is typed as the public interface; `var` hides dialect-concrete locals so the customer never types `X9ValidateEngine937`; `Path` is the source type (cloud-friendly, `Resource`-compatible); the Engine chain runs inline as a fluent expression. The customer constructs `X9SdkApplication` in a builder block and releases it via try-with-resources — the explicit per-application lifecycle, the project's default for standalone Java. JVM-global cleanup happens automatically via the defensive auto-cleanup mechanisms described in the lifecycle pillar; the customer never sees it.

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

/**
 * X9CashLetterValidation is an example of x9.37 file processing using the modern Engine API. It
 * opens an input x9.37 file, validates it with image checks enabled, modifies records (setting
 * the first item's amount and dropping the second), and writes the result to an output file.
 *
 * @author X9Ware LLC. Copyright(c) 2012-2026 X9Ware LLC. All Rights Reserved. This is proprietary
 *         software as developed and licensed by X9Ware LLC under the exclusive legal right of the
 *         copyright holder. All licensees are provided the right to use the software only under
 *         certain conditions, and are explicitly restricted from other specific uses including
 *         modification, sharing, reuse, redistribution, or reverse engineering.
 */
public final class X9CashLetterValidation {

    /**
     * Logger instance.
     */
    private static final Logger LOGGER = X9LoggerFactory.getLogger(X9CashLetterValidation.class);

    /**
     * X9CashLetterValidation Constructor (private; this is a static-main example).
     */
    private X9CashLetterValidation() {
    }

    /**
     * Main().
     *
     * @param args
     *            command line arguments: licenseKey, inputPath, outputPath
     */
    public static void main(final String[] args) {
        if (args.length != 3) {
            LOGGER.error("usage: X9CashLetterValidation <licenseKey> <input.x9> <output.x9>");
            System.exit(1);
        }
        final String licenseKey = args[0];
        final Path inputPath = Path.of(args[1]);
        final Path outputPath = Path.of(args[2]);

        try (var app = X9SdkApplication.builder()
                .applicationName("X9CashLetterValidation")
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

### Example 2 — Spring Boot via the starter

Example 1's Engine pipeline consumed through the `x9-sdk-spring-boot-starter` companion artifact. The starter contributes an `@Bean X9SdkApplication` built from `@ConfigurationProperties`, so the customer's Spring Boot application gets an autowired application root without constructing it. Spring Boot manages the per-application lifecycle — `@PreDestroy` runs `X9SdkApplication.close()` at shutdown, releasing per-application state. The starter disables the JVM shutdown hook that plain-Java callers rely on (Spring Boot owns shutdown), so there is no double-close concern. What changes from Example 1 is the scaffolding around the operation: `main()` is Spring Boot's, configuration is externalized to `application.yml`, the builder block is absorbed into the starter, and the operation becomes a `@Service` method callable from anywhere in the Spring Boot component graph. The Engine pipeline itself is byte-for-byte identical to Example 1's.

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

/**
 * X9CashLetterValidationService is the Spring Boot variant of the x9.37 validation example. The
 * X9SdkApplication is autowired from the x9-sdk-spring-boot-starter, lifecycle is managed by
 * Spring Boot ({@code @PreDestroy} calls {@code close()} at shutdown), and the operation lives
 * as an {@code @Service} method callable from anywhere in the Spring Boot component graph. The
 * Engine pipeline itself is identical to the plain-Java example.
 *
 * @author X9Ware LLC. Copyright(c) 2012-2026 X9Ware LLC. All Rights Reserved. This is proprietary
 *         software as developed and licensed by X9Ware LLC under the exclusive legal right of the
 *         copyright holder. All licensees are provided the right to use the software only under
 *         certain conditions, and are explicitly restricted from other specific uses including
 *         modification, sharing, reuse, redistribution, or reverse engineering.
 */
@Service
public final class X9CashLetterValidationService {

    /**
     * Logger instance.
     */
    private static final Logger LOGGER = LoggerFactory.getLogger(X9CashLetterValidationService.class);

    /**
     * Application root, autowired from the x9-sdk-spring-boot-starter.
     */
    @Autowired X9SdkApplication x9;

    /**
     * Verifies an x9.37 file by validating images and applying record modifications, writing the
     * result to the supplied output path.
     *
     * @param inputPath
     *            input x9.37 file
     * @param outputPath
     *            output x9.37 file
     */
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

## Related documents

- [`x9Sdk/docs/development/fluent-api/sdk-fluent-api-design.md`](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-design.md) — Design specification for the fluent API surface (Engine pattern, `X9SdkApplication`, Reader/Writer/Modify Engines, source-based I/O)
- [`x9Sdk/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md`](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md) — Style charter for fluent Engine conventions (terminal verb, factory pattern, source/sink naming, lifecycle ownership, exception model)
