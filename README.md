# Trading Proto

Public Protobuf and gRPC contract for trading event subscriptions.

This repository currently keeps the proto surface in a single root-level file:

```text
.
├── README.md
└── trading.proto
```

`trading.proto` defines the `trading` package, the `TradingService` gRPC service, subscription request messages, event envelopes, and broker data messages.

## Service

The API exposes one bidirectional streaming RPC:

```proto
service TradingService {
  rpc StreamEvents(stream StreamEventsRequest) returns (stream TradingEvent);
}
```

Clients open a long-lived stream and manage broker data subscriptions over the same connection.

## Client Messages

Client-to-server messages are wrapped in `StreamEventsRequest`:

```proto
message StreamEventsRequest {
  oneof msg {
    Subscribe subscribe = 1;
    Unsubscribe unsubscribe = 2;
  }
}
```

- `Subscribe`: adds event type subscriptions for an `account_id`, optionally scoped by `symbol`.
- `Unsubscribe`: removes event type subscriptions for an `account_id`, optionally scoped by `symbol`.

```proto
message Subscribe {
  string account_id = 1;
  repeated EventType subscriptions = 2;
  optional string symbol = 3;
}

message Unsubscribe {
  string account_id = 1;
  repeated EventType subscriptions = 2;
  optional string symbol = 3;
}
```

## Server Events

Server-to-client events are wrapped in `TradingEvent`:

```proto
message TradingEvent {
  oneof event {
    Heartbeat heartbeat = 1;
    SubscriptionUpdate subscription_update = 2;
    BrokerOrder broker_order = 3;
    BrokerPosition broker_position = 4;
  }
}
```

- `Heartbeat`: reports stream liveness with `sent_at`.
- `SubscriptionUpdate`: confirms subscribe and unsubscribe requests, including the message, subscribed symbols, and event types.
- `BrokerOrder`: broker-side order state and execution details.
- `BrokerPosition`: broker-side position state by symbol.
- `BrokerTrade`: a single execution against an order.
- `IntentAction`: a command the platform has committed, before the broker has answered it.

## Event Types

The current subscription enum includes:

```proto
enum EventType {
  EVENT_TYPE_UNSPECIFIED = 0;
  EVENT_TYPE_BROKER_ORDER = 1;
  EVENT_TYPE_BROKER_POSITION = 2;
  EVENT_TYPE_BROKER_TRADE = 3;
  EVENT_TYPE_INTENT_ACTION = 4;
}
```

The first three are the broker's account of what happened on an account. The
last is the platform's own: what it was asked to do and has durably stored,
delivered whether or not a broker connection exists.

## Shared Types

- Timestamps use `google.protobuf.Timestamp`.
- `Price` is represented as a fixed-precision value with `integer` and `precision` fields.
- Broker data messages include account, organization, broker, symbol, and received timestamp fields.

## Usage

Clone the repository:

```sh
git clone git@github.com:anthid-labs/proto.git
```

Generate client or server bindings from `trading.proto` with the Protobuf and gRPC tooling for your target language.

## License

MIT
