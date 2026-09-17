# Using the trading stream

The wire contract is defined in [trading_service.proto](../trading/trading_service.proto).
These notes describe the subscription contract and platform behavior. Deployment
addresses, credentials, quotas, and retention settings are configured outside
this repository. Schema updates alone do not deploy the corresponding behavior;
see the implementation references in [development](development.md).

## Connection and authentication

Obtain a stream endpoint and credentials for your target Anthid environment.
The platform authenticates requests before invoking either RPC and derives the
organization from the authenticated principal. The browser client sends
`authorization: Bearer <access-token>` and uses a gRPC-Web transport with
`SubscribeEvents`. A browser requires the environment's gRPC-Web gateway;
the protobuf definition alone does not provide that transport.

For native gRPC, the fully qualified methods are
`trading.TradingService/StreamEvents` and
`trading.TradingService/SubscribeEvents`.

## Subscription fields

| Field | Meaning |
| --- | --- |
| `trading_account_id` | Account UUID. In the current platform, empty means all accounts in the authenticated organization; a named account is checked for organization ownership. |
| `subscriptions` | Event kinds to receive. Send at least one recognized, non-unspecified kind. An empty list or a list containing only unknown/unspecified values is rejected. |
| `instrument_key` | Subject-safe instrument key for narrowing orders and positions on a named account. Empty means all instruments. |
| `instrument_class` | Lowercase subject class, such as `equity`, when a key is set. Empty matches any class, including unresolved instruments. A class alone does not narrow a subscription. |
| `order_delivery` | Unspecified or `ORDER_DELIVERY_LIVE` follows new order updates; `ORDER_DELIVERY_PRIMED` starts with retained current order state. Terminal snapshots expire under the platform's retention policy. |

In the current server, an organization-wide subscription ignores instrument
filters. Instrument filters on an account do not narrow trades or intent actions;
filter those payloads in the client if needed. A key is a stream subject coordinate,
not an `Instrument` message or a general symbol search. Obtain its subject-safe
spelling from the platform rather than inventing escaping rules.

## Requests

These examples use protobuf JSON field names. Replace the example account UUID
with a trading account owned by your organization.

For `SubscribeEvents`, send a `Subscribe` directly:

```json
{
  "tradingAccountId": "00000000-0000-4000-8000-000000000001",
  "subscriptions": [
    "EVENT_TYPE_BROKER_ORDER",
    "EVENT_TYPE_BROKER_POSITION",
    "EVENT_TYPE_BROKER_TRADE",
    "EVENT_TYPE_INTENT_ACTION"
  ],
  "orderDelivery": "ORDER_DELIVERY_PRIMED"
}
```

Keep reading until the call is cancelled or ends. To change this subscription,
cancel the call and start a new one with the desired fields.

For `StreamEvents`, keep the request stream open and wrap each command in
`StreamEventsRequest`. For example, subscribe to AAPL orders and positions:

```json
{
  "subscribe": {
    "tradingAccountId": "00000000-0000-4000-8000-000000000001",
    "subscriptions": ["EVENT_TYPE_BROKER_ORDER", "EVENT_TYPE_BROKER_POSITION"],
    "instrumentKey": "AAPL",
    "orderDelivery": "ORDER_DELIVERY_PRIMED"
  }
}
```

The current platform keeps one subscription per account scope per connection.
Another `Subscribe` for that scope replaces its event selection, instrument
filter, and delivery mode, and attaches again. Different account scopes can
coexist. Overlapping organization and account subscriptions can deliver the same
event more than once.

To remove the subscription for that account scope:

```json
{
  "unsubscribe": {
    "tradingAccountId": "00000000-0000-4000-8000-000000000001"
  }
}
```

`Unsubscribe` has only `trading_account_id`: it cannot selectively remove an
event kind or instrument. An empty ID addresses the organization-wide
subscription; it does not remove separate account subscriptions.

## Reading events and replay

Inspect `TradingEvent.event`, a `oneof`, for each response:

| Variant | Meaning |
| --- | --- |
| `heartbeat` | Stream liveness timestamp (`sent_at`) |
| `error` | Application error text (`message`); also handle gRPC call failures separately |
| `status` | Informational text (`message`), currently including `subscribed` and `unsubscribed` |
| `broker_order` | Order state and cumulative execution totals |
| `broker_position` | Position state |
| `broker_trade` | One execution, distinct from cumulative order state |
| `intent_action` | A committed platform command, not broker acceptance or execution |
| `primed` | Replay completion for its `event_type` |

Positions always replay retained state when requested. Orders replay retained
state only with primed delivery. Expect a separate `Primed` for each primed
kind, including when there is no retained state. For example:

```json
{"primed":{"eventType":"EVENT_TYPE_BROKER_POSITION"}}
```

A `status` message or a quiet stream is not evidence that replay is complete.
Trades and intent actions have no `Primed` completion contract. Retained order
state is one message per retained order, not every historical order transition.

The retention policy expires terminal-order and zero-position snapshots after
an hour. Working orders and nonzero positions remain until updated. Within a
successfully primed scope, an order absent from the order replay is not working,
and a position absent from the position replay is flat under that policy. This
says nothing about instruments or accounts excluded by the subscription.

`TradingEvent` has no deletion variant. Use terminal order statuses and zero
position quantities when received; a later replay may omit expired records.
Clients attaching after expiry will not receive those closing snapshots.

`Primed` contains no account ID, request ID, or instrument coordinates. Clients
that need an unambiguous readiness boundary per scope should use separate calls
for those scopes. Do not infer an atomic snapshot across event kinds.

On disconnect, reconnect and send subscriptions again. The requests have no
resume cursor or acknowledgement fields, so do not assume gap-free history or
exactly-once delivery. Handle repeated state and committed commands; see the
[data model](data-model.md) for correlation keys. Treat unknown enum values and
unrecognized/unset event variants deliberately when consuming newer schemas.
