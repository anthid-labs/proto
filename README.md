# Trading Events Proto

Public Protobuf + gRPC contract for streaming real-time trading events.

This repository defines a **server-streaming API** for consuming live trading data such as intents, broker orders,
positions, and execution results. It is intended for external clients integrating with a trading platform via long-lived
gRPC connections.

## Overview

The API exposes a single streaming RPC:

rpc StreamEvents(StreamEventsRequest) returns (stream TradingEvent);

Clients establish one connection per account and receive a continuous stream of typed events.

## Features

- Single unified event stream
- Strongly typed events via oneof
- Account-scoped subscriptions
- Selective event filtering
- Designed for low-latency, real-time systems
- Forward-compatible schema evolution

## Installation

Clone the repository:

git clone git@github.com:anthid-labs/proto.git

## API

### StreamEvents

rpc StreamEvents(StreamEventsRequest) returns (stream TradingEvent);

### Request

- account_id: string
- subscriptions: list of EventType

### Event Types

- EVENT_TYPE_INTENT
- EVENT_TYPE_BROKER_ORDER
- EVENT_TYPE_BROKER_POSITION
- EVENT_TYPE_INTENT_RESULT

## Event Envelope

All messages are delivered through a single wrapper:

TradingEvent {
oneof event {
Intent
BrokerOrder
Connected
Heartbeat
BrokerPosition
IntentResult
}
}

## Connection Lifecycle

1. Client connects via StreamEvents
2. Server sends a Connected event confirming subscriptions
3. Events stream continuously
4. Periodic Heartbeat messages maintain liveness

## Event Types

### Connected

Confirms active subscriptions.

### Heartbeat

Used for connection health.

### Intent

Represents a user-submitted action such as create, replace, or cancel order.

### IntentResult

Represents state transitions for an intent (submitted, accepted, filled, etc).

### BrokerOrder

Represents broker-side order state and execution details.

### BrokerPosition

Represents current position state per symbol.

## Notes

- All timestamps use google.protobuf.Timestamp (UTC)
- Price is represented as fixed precision (integer + precision)
- Schema is designed to be append-only for backward compatibility

## License

MIT
