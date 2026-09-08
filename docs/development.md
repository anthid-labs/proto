# Developing the contract

## Validate locally

Run commands from the repository root with `protoc` installed, including its
standard `google/protobuf` includes. The schemas use proto3 optional fields;
validation was checked with `libprotoc 3.21.12`.

```sh
protoc -I . --include_imports \
  --descriptor_set_out=/tmp/trading-descriptor.pb \
  trading/*.proto

git diff --check
```

If your compiler does not find `google/protobuf/timestamp.proto`, add an `-I`
pointing to the installed protobuf include directory. Keep `-I .`: imports such
as `trading/trading_models.proto` resolve from the repository root, not from the
`trading/` directory. The import-only `trading/trading.proto` and the service schema currently produce
unused-import warnings; these are not schema compilation failures.

## Generate bindings

Choose message and service generators for the consuming language. Pass all five
schemas and the repository include root to the generator. Plain protobuf
message generation does not by itself create a usable gRPC client/server.

For example, to generate Python **messages only** into a temporary directory:

```sh
mkdir -p /tmp/trading-python
protoc -I . --python_out=/tmp/trading-python trading/*.proto
```

This produces modules under `/tmp/trading-python/trading/`. Put
`/tmp/trading-python` on the Python import path and install a compatible protobuf
runtime to use them. A Python gRPC plugin is additionally needed for RPC stubs.
Generated output belongs in the consuming project or build directory.

The surrounding Anthid platform uses Rust bindings from `rust_types` through
`tonic-prost-build`, and TypeScript bindings generated with the Buf ES plugin.
Those build configurations and runtime dependencies belong to their respective
repositories, not this one. The current `rust_types` build script fetches this
repository into `.proto-cache` when it runs and may fall back to cached schemas
if fetching fails. Editing a separate local proto checkout does not automatically
update those bindings. Verify the actual schema revision used by each consumer.

## Compatibility and review

Before changing the wire contract:

1. Preserve existing field numbers and enum numeric values. Reserve removed
   fields' numbers and names, and removed enum values, instead of reusing them.
2. Review package, service, method, message, field-name, type, and oneof changes
   for binary, generated-source, and protobuf JSON compatibility. A field rename
   can preserve binary tags while breaking JSON clients.
3. Check defaults and presence semantics. An additive message field can still
   break applications if readers start requiring it, as with order instruments.
4. Update service and model documentation with changes in filtering, replay,
   event correlation, or application-level validation. Defining a field does
   not deploy its producers or consumers.
5. Compile the complete schema set, regenerate affected consumers, and run their
   relevant checks. Record the schema revision used; a successful descriptor
   build does not prove runtime compatibility with a deployed server.

This repository currently has no checked-in CI, Buf lint/breaking-change
configuration, generated SDKs, or automated compatibility tests. Validate changes
locally and coordinate producer/consumer rollout in the platform repositories.

## Documentation sources

The README and guides describe the checked-in schemas. Platform-specific usage
was also audited on 2026-09-08 against the `anthid` repository paths
`apps/streamer/src/api/validate_req.rs`,
`apps/streamer/src/api/stream/stream_events.rs`,
`apps/streamer/src/services/subscription_manager/{scope,manager,consumer}.rs`,
and `libs/packages/core/src/clients/tradingClient.ts`, plus `rust_types/src/build.rs`
for Rust generation. These are implementation references, not files shipped here
or guarantees that every deployment has the same revision.
