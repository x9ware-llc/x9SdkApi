# x9SdkApi

Modern Java API surface for the X9Ware SDK — a layered facade over [`x9Sdk`](https://github.com/x9ware-llc/x9Sdk) with customer-first ergonomic design for 2026.

## Status

**Under design — pre-1.0.** This repository is the design-and-discovery home for the modern API. Concrete implementation has not yet begun. The companion design documents will be referenced below as they are produced.

## Relationship to x9Sdk

`x9SdkApi` is a *parallel surface* over the existing `x9Sdk` legacy library, not a replacement.

- `x9Sdk` continues to be developed and released without change. Existing customers using `x9Sdk` directly are not affected.
- `x9SdkApi` depends on `x9Sdk` as a published Maven artifact and exposes its functionality through a modern API designed for the patterns developers and organizations expect of Java SDKs in 2026.
- Customers adopt `x9SdkApi` for new integrations while existing `x9Sdk` integrations continue running unchanged. Both APIs coexist, intentionally and indefinitely.

The two-API model follows the IBM CICS macro-level / command-level coexistence pattern — both are first-class, both are documented, neither replaces the other. The choice between them is a customer decision based on coding style, integration context, and existing investment.

## Design references

- *Broader 2026-modern API design document* — to be authored as the design-and-discovery work progresses; link added here once published.
- [`x9Sdk` fluent API charter](https://github.com/x9ware-llc/x9Sdk/blob/main/docs/development/fluent-api/sdk-fluent-api-style-charter-design.md) — the engine-fluent design produced inside `x9Sdk`. `x9SdkApi`'s broader design positions itself as a peer artifact.

## Java compatibility

`x9SdkApi` targets **Java 8** for source and target compatibility, matching the floor that customer enterprise environments require.

## Build

Standard Gradle project using the X9Ware conventions plugin. From the repository root:

```
./gradlew build
```

The build resolves `x9Sdk` from `mavenLocal` or GitHub Packages by default. See `settings.gradle` for the composite-build pattern that lets Eclipse navigate into `x9Sdk` source while command-line builds use published artifacts.
