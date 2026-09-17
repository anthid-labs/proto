# Anthid Trading Proto

Protobuf message definitions and the gRPC event subscription contract for the
Anthid trading platform. All five schemas use proto3 and the `trading` package.

This repository defines how clients observe broker orders, positions, executions,
and committed intent commands. It contains schemas, not a running server, an SDK,
or an order submission API. The platform's streamer implements the service;
order creation, replacement, and cancellation happen through the platform's
separate intents API.

## Payloads

- `BrokerOrder`: order state, cumulative fills, and platform, broker, and caller
  correlation IDs.
- `BrokerPosition`: quantity and average cost, with optional broker-reported
  `realized_pnl`, `daily_pnl`, `value_bought`, and `value_sold`. Missing values
  mean unknown. The broker defines their cost basis and session boundaries.
- `BrokerTrade`: one execution, with an optional execution ID, order totals and the command it executed under
  as of that fill.
- `IntentAction`: a committed create, replace, or cancel instruction, identified
  by `(intent_id, seq)`. Broker acceptance arrives separately.

Prices, quantities, and monetary values are decimal strings. Instruments carry
asset-specific metadata; optional position and trade instruments can be absent.
See the [data model](docs/data-model.md) for presence and correlation rules.

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
The retention policy keeps working orders and open positions, and expires
terminal-order and zero-position snapshots after an hour. Replaying retained
state does not provide a history of transitions. `Primed` applies to its event
kind within the requested scope; it is not an atomic snapshot across kinds or
accounts. Trades and intent actions have no replay-completion marker.

Retention and delivery are implemented by the platform. Updating these schemas
does not update a deployed streamer or its producers; verify the versions used
by the target environment.

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

Apache License 2.0.
