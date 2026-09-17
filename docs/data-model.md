# Trading data model

The field definitions and wire numbers in
[trading_models.proto](../trading/trading_models.proto),
[trading_instruments.proto](../trading/trading_instruments.proto), and
[trading_types.proto](../trading/trading_types.proto) are authoritative.

## Values and presence

Prices, quantities, costs, multipliers, increments, strikes, and payouts are
**decimal strings**, for example `"123.4500"` or `"0.00000001"`. There is no
`Price` message or integer/precision pair. Preserve decimal precision in client
arithmetic; protobuf string fields themselves enforce neither numeric syntax nor
a scale limit. Sequence numbers, calendar years, funding intervals, and enums
use integer fields rather than decimal strings.

Timestamps use `google.protobuf.Timestamp`. Optional fields distinguish absence
from a supplied value; do not turn an absent price, execution total, or broker
timestamp into zero. Proto3 non-optional scalar defaults do not establish
application validity. Sequence fields are `int64`; preserve their full range
(in protobuf JSON they are represented as strings).

## Events and identifiers

| Message | Interpretation and correlation |
| --- | --- |
| `BrokerOrder` | Platform `order_id`, `intent_id`, and `command_seq`; optional broker `external_order_id` and caller `client_reference_id`. Contains state/status, execution parameters, quantity, and filled quantity. |
| `BrokerPosition` | Account and symbol position level with quantity and average cost; optional instrument metadata, broker P&L, and bought/sold cash totals. |
| `BrokerTrade` | Individual execution. `order_id` is the broker order ID; `client_order_id` is the platform order ID. Optional `exec_id` identifies the execution. Optional cumulative, order, and leaves quantities are totals at that execution. |
| `IntentAction` | Durable create, replace, or cancel command, identified by `(intent_id, seq)`. Includes the committed parameter snapshot and optional caller `client_reference_id`. |

All four payloads carry `account_id` and `organization_id`. Only `BrokerOrder`
and `BrokerPosition` have a `broker` enum field; do not expect it on a trade or
intent action. The current `Broker` enum names Lightspeed Connect and Alpaca
plus unspecified. An enum or instrument variant does not guarantee a deployed
broker supports every represented asset or order type.

Correlate `IntentAction.seq` with `BrokerOrder.command_seq` under the same
`intent_id`. An intent action says the platform durably stored an instruction;
it can arrive without a broker connection and is not proof the broker acted.
Use `(intent_id, seq)` to deduplicate republished commands. `client_reference_id`
is caller correlation across an intent's create/replace/cancel chain, not a
unique identifier for each command or fill.

Use an execution's `exec_id` where supplied, with its account/broker context.
Without it the schema offers no universally collision-free execution ID; an
order ID alone cannot distinguish several fills. Do not count both a trade's
quantity and an order's cumulative filled quantity as new executions.

`IntentAction.principal_type` identifies the credential class (API key, user,
or service). Credential/person IDs, IP addresses, request IDs, and idempotency
keys from the internal command record are not published in this message.

## Broker position figures

`BrokerPosition` also carries four optional decimal strings supplied by the
broker:

| Field | Meaning |
| --- | --- |
| `realized_pnl` | Realized profit and loss using the broker's cost basis. |
| `daily_pnl` | Profit and loss for the broker's current session. |
| `value_bought` | Running cash total bought in this symbol during the broker's accounting period. |
| `value_sold` | Running cash total sold in this symbol during that period. |

Absence means the broker did not supply the figure; a supplied `"0"` is a
measured zero. These are broker accounting figures, not platform-calculated
returns. Cost bases and session boundaries can differ between brokers, and a
carried position may be re-marked overnight.

Bought/sold values are cash totals, not share quantities or individual fills.
They can change for trading outside the platform and can reset at the broker's
accounting boundary. They provide reconciliation evidence, not execution IDs.

## Clocks

Broker payload `received_at` is the platform ingest timestamp used to compare
received state. Optional `broker_sent_at` is the broker's reported time and is
for reporting; it is a different clock and should not replace `received_at` in
state comparisons. Neither field is a global stream sequence number.

For intent actions, `received_at` is instruction arrival and `created_at` is
database commit time, stable across republishing the command. Order commands
within an intent by `seq`; do not treat arrival order across broker and intent
events as one global timeline.

## Instruments

`Instrument.instrument` is a oneof with seven variants:

| Variant | Metadata beyond the venue's symbol |
| --- | --- |
| `equity` | Optional listing exchange |
| `future` | Root, exchange, last-trade expiration, tick size, multiplier, currency |
| `option_instrument` | Underlying, expiration, strike, call/put, exercise style, multiplier, optional exchange/tick size |
| `currency` | Base and quote currencies |
| `crypto` | Base/quote, venue, price and quantity increments |
| `perpetual` | Base/quote, venue, underlying kind, settlement, multiplier, funding schedule, tick and quantity increments |
| `prediction` | Market, outcome, venue, payout, currency, tick size, optional expiration |

The oneof identifies the asset class; `Instrument` has no separate `asset_class`
field. `AssetClass` is a standalone enum. `ContractMonth` represents a parsed
futures root, calendar month, and full year, not a resolved futures contract.

`BrokerOrder.instrument` is required by the documented application contract,
although protobuf permits an absent message. Producers must populate it and
keep `symbol` consistent with it. Position and trade instruments are optional;
absence means unknown, not equity. `IntentAction` currently carries a symbol
but has no instrument field.

Instrument listing/venue metadata is distinct from the order's `OrderVenue`
routing instruction. Preserve contract multipliers, settlement type, payout,
and increments when interpreting values. A perpetual has no expiration field;
a prediction has an optional expiration. Unspecified settlement or outcome is
unknown, not implicitly linear or yes. `FundingSchedule` supplies an interval,
not the current funding rate.
