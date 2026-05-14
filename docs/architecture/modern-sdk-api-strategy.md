# Modern X9Ware SDK API Strategy

X9Ware LLC • May 8, 2026

## Status

**DRAFT — under active development.** This strategy describes the destination for a modern X9Ware SDK API surface, the structural decisions that shape the work, and the principles that guide subsequent design. Specific class signatures, method names, and implementation sequencing are out of scope and belong to the design documents that follow.

## Vision

A modern Java API in 2026 is more than a fluent grammar. It is a coherent surface that reads as something an enterprise developer would have built themselves if given the year and the resources — interface-based, source-flexible, observability-ready, lifecycle-invisible, framework-friendly, container-deployable, and well-organized around customer mental models. Several pillars combine to produce that experience; no single one is sufficient on its own.

X9Ware's competitive advantage in 2026 lives in the destination, not in shipping velocity. AI tooling has flattened how quickly any team can produce incremental features; differentiation now comes from where you choose to head. This strategy names the destination and the structure for delivering it.

## Executive summary

X9Ware should publish a second, modern Java API alongside the existing SDK. The legacy `x9Sdk` continues unchanged for the eighteen customers built on it. A new artifact, `x9SdkApi`, becomes the surface 2026 enterprise prospects evaluate when comparing X9Ware against commercial and open-source alternatives.

The strategic case is **customer acquisition**. In 2026, Spring ecosystem compatibility is table stakes for enterprise Java; roughly nine of ten Fortune 500 companies use Java with Spring Boot as the primary backend choice. AI-armed competitors can match incremental moves quickly, which means differentiation must live in the *destination*, not the velocity. The test for the destination: "would a developer evaluating this say *yes, this is what I would build myself*."

`x9SdkApi` delivers that destination. Engines and the `X9SdkApplication` lifecycle root become the customer-facing surface, shaped by the design language modern Java APIs follow: Engine-centric (the Engine is the noun, source is configuration), source-agnostic (file, stream, programmatic items uniformly), interface-based, JavaBean-conventional, observability-ready, container-friendly, virtual-thread-compatible. The same operation runs in 61 lines of plain Java, or 49 lines as a Spring Boot `@Service` consumed through the starter — down from 760 lines in the legacy direct-construction style today. The verbose setup block every example opens with today — license registration, configuration loading, dialect binding, and SDK options — collapses to a small builder block in plain Java and to zero customer code in Spring Boot.

This is "think big, deliver big" applied to the SDK API.

## Strategic position

Four operating principles shape the strategy.

**Think big, deliver big.** The destination has to be worth aiming for. Incremental polish on the existing SDK is not the destination; an API that 2026 enterprise developers want to use is.

**Differentiation lives in the destination, not the velocity.** AI lets X9Ware build anything we can dream of — and it lets competitors do the same. The advantage is no longer in shipping faster; it is in shipping a destination materially better than what an open-source integration framework or a commercial check-processing SDK ships. The bar is high because the competition's bar is rising too.

**The acquisition lens.** The eighteen current x9Sdk customers came in over the past decade and use the legacy direct-construction surface. Their tech stacks reflect that era. They are not the audience for `x9SdkApi`; they continue with x9Sdk unchanged. The audience for `x9SdkApi` is the new prospect a sales conversation surfaces tomorrow — predominantly Cloud, Spring Boot, container-deployed, modernization-aware. That prospect evaluates the SDK by feel before they ever ship a line of integration code; the API has to read as something they would have built themselves if they had a year to do it right.

**Spring ecosystem compatibility is table stakes in 2026.** Approximately 55–60% of enterprise Java workloads run on Spring Boot, and ~92% of enterprise backend Java job postings reference Spring. Quarkus and Micronaut exist but are bounded — their differentiators (cold-start time, memory footprint) do not apply to file-processing workloads. For X9Ware's market, choosing Spring as the substrate is a low-risk decision with high upside.

## The proposal — modern API delivered in x9Sdk

The customer-facing end state is a single Maven artifact:

- **`x9Sdk`** carries both the **legacy direct-construction surface** that the eighteen existing customers depend on and the **modern API surface** (Engines, `X9SdkApplication`, fluent grammar, interface-based types, observability seams). Java 1.8 floor. The two surfaces coexist within one artifact, in the spirit of the IBM-CICS dual-API model that kept macro-level and command-level APIs together in one product. Customers choose whichever surface fits their integration, and the legacy surface continues exactly as it is for the customers depending on it.
- An optional **Spring Boot starter** (`x9-sdk-spring-boot-starter`) ships separately as the only other artifact, providing zero-ceremony Spring integration for customers who want it. The starter depends on x9Sdk; it is not required for the modern API surface itself.

**Development happens in `x9SdkApi` as a temporary isolation artifact.** During the development phase, the modern API surface is built in a separate `x9SdkApi` repository so the experimental work does not touch x9Sdk's production codebase. Before the first customer release of the modern API surface, `x9SdkApi`'s content is merged into `x9Sdk` with git history preserved. Customers never see `x9SdkApi` as a Maven artifact — it is purely internal scaffolding. The merge procedure is documented in the runbook referenced under *Path forward*.

**The Engines (`X9ValidateEngine`, `X9ScrubEngine`, `X9CloneEngine`, etc.) are part of the modern API surface** that lands in `x9Sdk` via the merge. The eighteen existing x9Sdk customers can use Engines without subscribing fully to the modern style, and X9Ware's own apps (x9Assist, x9Utilities) get the same access. The `engine.legacy()` accessor concept is unnecessary in the merged single-artifact end state: a customer reaching below the modern surface uses the legacy direct-construction classes (`X9SdkBase`, `X9Writer`, `X9SdkIO`) that already live in the same artifact, so there is no within-artifact "legacy" boundary to bridge.

**Packaging within the merged x9Sdk uses a flat package tree.** Modern API classes and existing classes coexist in functional packages; there is no top-level `com.x9ware.api.*` namespace separating modern from legacy. The modern API surface is identified by documentation, the User Guide, and example programs, not by package-naming convention. A `com.x9ware.internal.*` namespace was rejected because moving existing classes into it would break the eighteen existing customers' imports.

### JDK floor for the modern API surface

**Decision: Java 1.8.** x9Sdk's existing Java 1.8 floor is preserved; the modern API surface compiles and runs at the same level. This section answers whether the *modern* surface is reachable from Java 8. It is, intentionally — the absence of a dramatic-better case for upgrading outweighs the customer-segment cost of excluding Java 1.8 shops from the modern API.

Candidates considered:

| Floor | What it would enable in our own code | Excluded segment |
|---|---|---|
| **Java 1.8** (chosen) | Matches x9Sdk's existing floor; every current customer can adopt the modern API surface | — |
| **JDK 11** | `var`, modules, `java.net.http`, text blocks (JDK 13 backport varies) | Java 1.8 holdouts on long-term-support contracts |
| **JDK 17** | Records, sealed types, pattern-matching `instanceof`, text blocks, switch expressions; aligns with Spring Boot 3 starter floor | Java 1.8 + JDK 11 LTS shops |
| **JDK 21** | Virtual threads, record patterns, switch patterns; aligns with likely Spring Boot 4 floor | Narrower still; threading is not typically a concern at the SDK API boundary |

Customer call-site features — `var`, pattern-matching `instanceof`, switch expressions, text blocks in their own code, virtual threads in their own runtime — work at any JDK the customer chooses, regardless of our compiled floor. The features that require *us* to upgrade are the ones we ship in our public surface.

**Explicitly traded away.** Two API-design moves have no Java 1.8 equivalent and are worth naming as the cost of this choice:

- **Sealed result hierarchies with exhaustive `switch`** (JDK 17). An interface plus package-private constructors approximates the closed-subtype invariant; nothing approximates the compile-time exhaustiveness check at the customer call site. Validation outcomes (`Clean | Warnings | Errors`) stay JavaBean-shaped with an `if (result.getSeverity().isError())` idiom rather than an exhaustive switch.
- **Module-enforced public API boundary** (JDK 9+). The convention-based "internal" marker (package naming, `@apiNote`, Javadoc warnings) remains the only mechanism keeping customers out of implementation types. JPMS would enforce the boundary at compile time so a customer reaching for an internal class would get a compile error rather than a runtime surprise.

Records-as-return-types can be approximated with verbose final classes (or Lombok); the customer-side difference is destructuring syntax in `switch`, which the JavaBean idiom approximates with `instanceof` + getter chains.

This decision is revisitable as the Java 1.8 customer segment shrinks. JDK 17 floor + JPMS modules + sealed result types is a coherent forward-looking package — recorded here so the option is preserved rather than rediscovered.

## Pillars of a modern SDK API

The "would you write this yourself" test is the design north star. The pillars below name what that test actually requires of an SDK in 2026. Each is a foundation the surface rests on; together they produce the coherent experience a 2026 enterprise prospect expects. Each pillar is independently meaningful but does not deliver the destination on its own — the strength of the API comes from their alignment.

**Engine-centric Fluent API.** The Engine is the noun. The customer's first interaction is with an Engine — `X9ValidateEngine`, `X9WriteEngine`, `X9ReadEngine` — not with a file, a facade method, or a utility-shaped wrapper. Source (file, stream, programmatic items) is configuration on the Engine; the Engine's lifecycle is build → configure → run; the result is a typed summary the customer reads. The fluent grammar makes the chain read as a single intentional act rather than as a sequence of imperative steps. The detailed design of the Engine factory pattern, terminal verb, source/sink builder methods, and conventions every Engine follows lives in the fluent API design referenced under *Related documents*.

**Source-agnostic by construction.** The API supports any source uniformly: `Path`, `InputStream`, Spring `Resource`, programmatic items. No type in the public surface forces customers through `java.io.File`. The current `X9File` extends `java.io.File` and encodes that constraint into the type system; modern x9SdkApi cannot. Cloud storage, Resource abstractions, and stream-only inputs all become first-class without API duplication.

**Customer-facing type design.** Customer-facing types on the modern API surface follow modern Java conventions consistently:

- **JavaBean accessors throughout.** No public fields; predictable `get*` / `set*` / `is*` naming on result objects (`X9WriteSummary`, `X9ReadSummary`, `X9ValidationResult`), configuration types, item classes (`X9Item937`, `X9ItemAch`, etc.), and `X9SdkApplication`. Legacy x9Sdk classes carry historical inconsistencies — some expose public fields directly — and stay where they are for the eighteen existing customers, but no new type on the modern surface inherits the inconsistency.
- **Interfaces, not concrete classes, for major types.** Engines and the lifecycle root expose Java interfaces, not just concrete classes, for the public API. Customers using Spring Framework's dependency injection get clean substitution (test doubles, partial mocks); customers using plain `new` construction lose nothing. Interface-typed return values keep customer code decoupled from internal implementation evolution.

The `X9Object` internal byte-array representation stays inside the implementation; customers work with the typed item classes and never reach for `X9Object` directly.

**Customer-managed lifecycle with defensive auto-cleanup.** `X9SdkApplication` is the lifecycle root, exposed as a Java interface implementing `AutoCloseable`. Customers may manage it explicitly via try-with-resources or Spring Boot's `@PreDestroy` (through the starter), but cleanup is also automatic when they do not — the SDK installs unattended cleanup paths so legacy state is never leaked. The seven-step legacy preamble is absorbed into the builder.

Three mechanisms combine so cleanup is invisible to the customer in every common scenario:

- **Phantom-reference cleanup** registered at `build()` time. GC fires it when the customer's reference to the application becomes unreachable during the JVM lifetime (variable falls out of scope, instance replaced). At the Java 1.8 floor, the implementation uses `PhantomReference` + `ReferenceQueue` + a daemon cleanup thread; this would migrate to `java.lang.ref.Cleaner` if the JDK floor is ever raised.
- **JVM shutdown hook** registered at first construction, installed by default. Fires on JVM exit if the customer never released the application earlier. Bounded to SDK-owned cleanup; each step is wrapped in try/catch with a timeout on thread-pool shutdown so a hung worker cannot strand JVM exit.
- **Explicit `close()`** for deterministic lifecycle. Idempotent — the auto-cleanup paths see "already closed" and no-op.

| Scenario | Ceremony required | Cleanup path |
|---|---|---|
| Plain Java (CLI, JavaFX, daemon, no application framework) | None | JVM shutdown hook on exit, or phantom-reference cleanup if the reference is dropped earlier |
| Spring Boot via the starter | None | Starter wires `@PreDestroy`; starter disables the JVM hook (Spring Boot owns shutdown) |
| Anything else (explicit lifecycle preference, non-Spring framework integration, community extension) | Optional `try-with-resources` and/or `.disableShutdownHook()` | Customer manages lifecycle directly or via whatever integration their framework provides |

Per-engine lifecycle is implicit in `.run()` — Engine resources are acquired and released within the fluent chain's terminal verb. Long-lived readers that yield records lazily, or writers that hold file handles across multiple `addItem` calls, expose the Engine as `AutoCloseable` for explicit wrapping; the default case never sees this.

The JVM shutdown hook is bounded to SDK-owned cleanup. Each step is internally tolerant (the underlying shutdown sequence wraps its work in try/catch/finally; the logger close carries its own idempotency guard). The starter disables the hook in Spring Boot scenarios where Spring Boot owns shutdown. Customers using other frameworks opt out explicitly via `.disableShutdownHook()`. Phantom-reference cleanup covers the long-running case where try-with-resources does not fit and the hook has not fired yet.

**Standards-based observability.** The modern API surface emits signals through the standard Java observability interfaces — SLF4J for logs (already in place across x9Sdk runtime), Micrometer for metrics, OpenTelemetry-compatible spans on long-running operations. The customer's existing observability stack — Prometheus, Datadog, New Relic, Grafana, anything else — receives those signals because both sides speak the same standards. The SDK does not pick the backend or require the customer to wire up custom hooks; it conforms to the interfaces the customer's infrastructure already consumes. When no observability stack is present (the customer has not configured a `MeterRegistry`, a `Tracer`, or a particular SLF4J binding), the SDK is silently inert on those channels — no logs lost, no metrics buffered, no spans created without a receiver. This is a step beyond legacy x9Sdk, which emits only logs; the modern surface adds metrics and traces as first-class signals when the customer wants them.

**Modern threading and container-friendliness.** The modern API surface is built to run cleanly in the environments 2026 enterprise prospects actually deploy into. Engines do not hold thread-locals or static caches; per-task state lives on instances the customer creates or that the customer's framework scopes. Spring Framework consumers get clean `@Bean` definition and `@Autowired` substitution because the interface-based public surface is DI-friendly by construction; Spring Boot consumers get zero-ceremony onboarding via the `x9-sdk-spring-boot-starter` companion artifact, which contributes an `@Bean X9SdkApplication` and wires `@PreDestroy` to `close()` automatically. Virtual threads (Java 21+) work without additional configuration on the customer side because no thread-confined state hides in our implementation. Container lifecycle (`@PreDestroy` from any container that honors it, Kubernetes pod shutdown via `AutoCloseable`) closes resources cleanly.

### Looking ahead — Engines as operators

The pillars above are what the modern API surface delivers. The Engine design also opens a door this strategy does not require for the foundation to be complete but is worth naming because the Engine shape positions it naturally: **Engines have clean input/output contracts that let them play the role of operators in payment-file workflows.** A read operator emits an X9Item stream from a file or stream; transformation operators (modify, validate, scrub) consume and produce X9Item streams; a sink operator writes the stream into a file or stream. The shape is the pipe-and-filter pattern familiar from shell pipelines and stream-processing frameworks, applied to the natural unit of payment-file processing — the X9Item.

A future direction the modern API surface accommodates is chaining these operators into reusable workflows for complex payment-file operations: file consolidation across distributed capture points, multi-stage processing pipelines, dialect-conversion flows. X9Collector is a concrete candidate — its file-consolidation workflow is a natural composition of Engine operators.

The connective tissue between operators — the streaming protocol, backpressure handling, error semantics, resource lifecycle across stages, workflow runtime — is genuinely complex and is not specified by this strategy. The foundation today supports independent Engine reuse via dependency injection, as Example 3 in the Appendix illustrates. When and whether to extend the Engine design with a workflow primitive is a subsequent design and implementation decision; the Engine shape does not foreclose that direction.

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

What this replaces: a roughly seventy-line preamble that opens every example today, plus the imperative I/O loop and manual heap walks that follow it.

The same Engine accepts a `Path` source instead of a stream, or a programmatic list of items, with no different API shape — just a different builder method. Source is configuration on the Engine, not a different API for each source type.

## What is not decided here

Specific decisions deferred to subsequent design work:

- Whether the modern API surface commits to Spring Framework as a runtime substrate (recommendation: no — Spring lives in the opt-in starter, not in x9Sdk; final decision deferred)
- Whether the Spring Boot starter ships in the initial release cycle or follows the core's stabilization (recommendation: follow stabilization; deferred for now)
- Per-Engine builder method names beyond the universal verbs (those belong to the per-Engine designs)
- The exact source-abstraction interface (a sibling design for source/sink handling follows from this strategy)
- The ordering of Engine fluent-retrofit work between the existing renamed Engines and the new Reader/Writer/Modify Engines
- When (or whether) to implement Engine-as-operator composition with a workflow primitive. The Engine design accommodates this direction; the streaming protocol, backpressure handling, error semantics, resource lifecycle, and workflow runtime are deferred to subsequent design and implementation work. The decision can wait until a concrete consumer (X9Collector or another product) needs it

## Tradeoffs accepted

Decisions in this strategy carry deliberate tradeoffs. Naming them explicitly invites alignment conversations rather than leaving the costs implicit.

- **Full backward compatibility for x9Sdk.** Legacy public fields, naming inconsistencies, and direct-construction patterns in x9Sdk's existing classes are not modernized. The eighteen existing customers see no changes to the classes they depend on. The modern API surface follows new conventions; the legacy surface stays as it is. The dual-API position absorbs the asymmetry.
- **Java 1.8 floor.** The modern API surface ships at Java 1.8 to match x9Sdk's existing floor and avoid excluding customers on long-term-support Java 8 contracts. The cost: no records, sealed types, JPMS modules, or other post-Java-8 language features in our own code.
- **Spring supported but not used.** The modern API surface carries no Spring dependency. Spring integration is available through the optional `x9-sdk-spring-boot-starter` companion artifact. Customers who do not use Spring carry no Spring code; Spring shops get zero-ceremony integration via the starter.
- **Convention-based API boundary.** Without JPMS modules (a consequence of the Java 1.8 floor), the public-vs-internal boundary relies on package-naming convention, `@apiNote` Javadoc tags, and documentation. A customer reaching into an internal class gets no compile-time block; the discipline is editorial rather than enforced.

## Path forward

1. **Strategy review.** This document, reviewed internally and revised with feedback.
2. **Substrate decisions.** Sibling designs for: source/sink abstraction, lifecycle abstraction, observability seams.
3. **Pilot Engine.** One Engine implemented in the x9SdkApi development repository following the design language. Validates the conventions against real code before they are locked across the surface.
4. **Incremental rollout.** Subsequent Engines, with the User Guide reframe shipping topics in lockstep.
5. **Merge x9SdkApi into x9Sdk** before the first customer release of the modern API surface, preserving x9SdkApi's git history. See [`modern-api-merge-runbook.md`](../projects/modern-api-merge/modern-api-merge-runbook.md) for the procedure.
6. **Spring Boot starter** as a follow-on artifact once the merged core stabilizes and customer demand is observed.

The pilot Engine is the empirical test. If the design language proves brittle in real code, the conventions revise before the rollout proceeds. The strategy itself is stable; specific design choices below it (interface shapes, exact method names) are pilot outcomes.

## When to revisit

The strategy is expected to be stable for the modernization rollout. Specific signals that warrant revision:

- A paying customer surfaces with conflicting needs that x9Sdk cannot serve and x9SdkApi cannot serve in its proposed form.
- The Spring ecosystem shifts materially (Spring Framework 7, Spring Boot 4) in ways that change the substrate decision.
- A competitor ships a check-processing SDK that materially redefines what "looking better in 2026" means and the strategy's design language no longer matches.
- AI tooling changes the bar for "would you write this yourself" — the destination has to keep moving.

## Appendix — Code Examples

This appendix shows the same operation — open an X9.37 file, validate it, modify two records, and write a new output — in three customer-facing examples that illustrate progressive integration shapes a customer might adopt:

1. **Plain Java** — the Engine pipeline in plain Java. Customer constructs `X9SdkApplication` explicitly via builder, uses Engines, releases via try-with-resources. No Spring dependency.
2. **Spring Boot via the starter** — the same Engine pipeline consumed through the `x9-sdk-spring-boot-starter` companion artifact. Starter provides `X9SdkApplication` as a Spring-managed `@Bean`; Spring Boot handles construction and `@PreDestroy`. Operation code is identical to Example 1.
3. **Spring-based Engine reuse** — Engines defined as reusable `@Bean` definitions, composed via dependency injection in a workflow service. Illustrates the foundation the *Looking ahead* note describes: today, Spring DI gives customers reusable Engine instances; in the future, a workflow primitive could chain those Engines as operators.

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

### Example 3 — Spring-based Engine reuse

Engine definitions become reusable Spring beans, configured once and injected into multiple workflow services. The same `imageValidator` bean can serve daily settlement, exception processing, audit workflows — wherever image validation is needed. The service composes Engines explicitly in plain Java: it calls each Engine, handles its result, and decides what to do next.

This example demonstrates what Spring DI delivers today: **reusable Engine instances, not pipeline composability.** There is no common operator interface, no streaming protocol between Engines, no built-in backpressure or shared exception model. The service writes the orchestration logic explicitly. The Engine design supports a future workflow primitive that would chain these as operators (described under *Looking ahead*), but that primitive does not exist on the foundation this strategy commits to.

```java
package com.x9ware.examples;

import java.nio.file.Path;

import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.stereotype.Service;

import com.x9ware.application.X9SdkApplication;
import com.x9ware.engines.X9ModifyEngine;
import com.x9ware.engines.X9ValidateEngine;
import com.x9ware.engines.X9WriteEngine;
import com.x9ware.summaries.X9ValidationResult;
import com.x9ware.summaries.X9WriteSummary;

@Configuration
public class CheckProcessingEngines {

    @Bean
    public X9ValidateEngine imageValidator(final X9SdkApplication app) {
        return X9ValidateEngine.x937(app)
                .validateTiffImages(true)
                .build();
    }

    @Bean
    public X9ModifyEngine bofdNormalizer(final X9SdkApplication app) {
        return X9ModifyEngine.x937(app)
                .transform(item -> { /* normalize BOFD routing */ return item; })
                .build();
    }

    @Bean
    public X9WriteEngine writer(final X9SdkApplication app) {
        return X9WriteEngine.x937(app).build();
    }
}

@Service
public class DailySettlementService {

    private static final Logger LOGGER = LoggerFactory.getLogger(DailySettlementService.class);

    private final X9ValidateEngine imageValidator;
    private final X9ModifyEngine bofdNormalizer;
    private final X9WriteEngine writer;

    public DailySettlementService(final X9ValidateEngine imageValidator,
                                   final X9ModifyEngine bofdNormalizer,
                                   final X9WriteEngine writer) {
        this.imageValidator = imageValidator;
        this.bofdNormalizer = bofdNormalizer;
        this.writer = writer;
    }

    public X9WriteSummary process(final Path input, final Path output) {
        // Each Engine is invoked independently; the service orchestrates them.
        // A future workflow primitive could chain these into a single composition
        // expression — see "Looking ahead — Engines as operators" in the strategy body.
        final X9ValidationResult validation = imageValidator.fromPath(input).run();
        if (validation.getSeverity().isError()) {
            LOGGER.warn("validation reported errorCount={}", validation.getErrorCount());
        }
        return bofdNormalizer.fromPath(input).toPath(output).run();
    }
}
```

## Related documents

- [`x9Sdk/docs/development/fluent-api/sdk-fluent-api-design.md`](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-design.md) — Design specification for the fluent API surface (Engine pattern, `X9SdkApplication`, Reader/Writer/Modify Engines, source-based I/O)
- [`x9Sdk/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md`](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md) — Style charter for fluent Engine conventions (terminal verb, factory pattern, source/sink naming, lifecycle ownership, exception model)
