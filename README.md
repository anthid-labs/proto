# Trading Proto

Public Protobuf and gRPC contract for trading event subscriptions.

This repository currently keeps the proto surface in a single root-level file:

```text
.
├── README.md
└── trading.proto
```

`trading.proto` defines the `trading` package, the `TradingService` gRPC service, request envelopes, event envelopes, and broker data messages.

## Service

The API exposes one bidirectional streaming RPC:

```proto
service TradingService {
  rpc StreamEvents(stream StreamEventsRequest) returns (stream TradingEvent);
}
```

Clients open a long-lived stream, send an initial account message, and then manage broker data subscriptions over the same connection.

## Client Messages

Client-to-server messages are wrapped in `StreamEventsRequest`:

```proto
message StreamEventsRequest {
  oneof msg {
    Init init = 1;
    Subscribe subscribe = 2;
    Unsubscribe unsubscribe = 3;
    Heartbeat ping = 4;
  }
}
```

- `Init`: identifies the account with `account_id`.
- `Subscribe`: adds event type subscriptions, optionally scoped by `symbol`.
- `Unsubscribe`: removes event type subscriptions, optionally scoped by `symbol`.
- `Heartbeat`: keeps the stream alive.

## Server Events

Server-to-client events are wrapped in `TradingEvent`:

```proto
message TradingEvent {
  oneof event {
    Connected connected = 1;
    Received received = 2;
    Heartbeat heartbeat = 3;
    BrokerOrder broker_order = 4;
    BrokerPosition broker_position = 5;
  }
}
```

- `Connected`: confirms active subscriptions after connection setup.
- `Received`: acknowledges subscription updates and includes subscribed symbols.
- `Heartbeat`: reports stream liveness with `sent_at`.
- `BrokerOrder`: broker-side order state and execution details.
- `BrokerPosition`: broker-side position state by symbol.

## Event Types

The current subscription enum includes:

```proto
enum EventType {
  EVENT_TYPE_UNSPECIFIED = 0;
  EVENT_TYPE_BROKER_ORDER = 1;
  EVENT_TYPE_BROKER_POSITION = 2;
}
```

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
