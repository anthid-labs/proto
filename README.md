# Anthid Trading Proto

Protobuf message definitions and the gRPC event subscription contract for the
Anthid trading platform. All five schemas use proto3 and the `trading` package.

This repository defines how clients observe broker orders, positions, executions,
and committed intent commands. It contains schemas, not a running server, an SDK,
or an order submission API. The platform's streamer implements the service;
order creation, replacement, and cancellation happen through the platform's
separate intents API.

## Schema layout

| File | Contents |
| --- | --- |
| [trading/trading_service.proto](trading/trading_service.proto) | RPCs, subscription requests, event envelope, delivery mode, and control events |
| [trading/trading_models.proto](trading/trading_models.proto) | `BrokerOrder`, `BrokerPosition`, `BrokerTrade`, and `IntentAction` |
| [trading/trading_instruments.proto](trading/trading_instruments.proto) | Instrument variants and asset-specific metadata |
| [trading/trading_types.proto](trading/trading_types.proto) | Broker, order, venue, and intent enums |
| [trading/trading.proto](trading/trading.proto) | Import-only entry point; declares no messages or services |

Imports are relative to the repository root. There is no root-level
`trading.proto`. Generate bindings from **all** `trading/*.proto` files: importing
schemas does not necessarily generate their bindings, and the entry point uses
ordinary imports rather than public re-exports.

## Streaming API

`trading.TradingService` exposes two RPCs:

| RPC | Request → response | Use |
| --- | --- | --- |
| `StreamEvents` | stream of `StreamEventsRequest` → stream of `TradingEvent` | Native/backend clients managing subscriptions on one connection |
| `SubscribeEvents` | one `Subscribe` → stream of `TradingEvent` | Browser clients over gRPC-Web, or native clients with a fixed subscription |

Clients select one or more event types: `EVENT_TYPE_BROKER_ORDER`,
`EVENT_TYPE_BROKER_POSITION`, `EVENT_TYPE_BROKER_TRADE`, or
`EVENT_TYPE_INTENT_ACTION`. The response also carries heartbeat, error, status,
and replay-completion (`Primed`) events.

Subscriptions use `trading_account_id`; payloads use `account_id`. Positions
are primed with retained state on attach. Orders default to live delivery;
request `ORDER_DELIVERY_PRIMED` to receive retained order state before live
updates. A `Primed` event identifies the event kind whose replay has completed.
This is retained current state, not an order-history query.

## Get started

```sh
git clone git@github.com:anthid-labs/proto.git
cd proto

# Requires protoc with proto3 optional support and the standard protobuf includes.
protoc -I . --include_imports \
  --descriptor_set_out=/tmp/trading-descriptor.pb \
  trading/*.proto
```

Use the descriptor to inspect the contract, or configure your language's message
and gRPC generators with the same inputs and include root. No generated bindings,
package manifest, or generation pipeline are checked into this repository.

- [Usage and subscription lifecycle](docs/usage.md): requests, scope, replay, authentication, and client behavior.
- [Data model](docs/data-model.md): decimals, identifiers, timestamps, instruments, and intent correlation.
- [Development and compatibility](docs/development.md): generation, validation, and schema changes.

## License

No license file is currently included in this repository. The previous README
stated MIT, but the repository does not contain the corresponding license terms.
