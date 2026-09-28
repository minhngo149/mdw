# State Contract

Compare the business states a producer exposes across a boundary with the states a consumer
understands: the names, the meanings, which transitions can happen, and which states are final.
Only states that **cross the boundary** matter here: values in an event, an API response, or a
shared record. A service's full lifecycle belongs to `distributed-flow`.

## Contents

1. Extract the producer's exposed states
2. Extract the consumer's interpretation
3. Build the mapping table
4. Mismatch patterns
5. Structural vs semantic mismatch
6. Evidence to collect
7. Common false positives

## 1. Extract the producer's exposed states

- The value set: the enum or constants, and every value actually **assigned** on a path that
  reaches the boundary. A declared but never assigned value cannot cross.
- The transitions: the transition table, guards (`if o.Status != Pending`), or state-machine
  library configuration. Note which transitions **emit** across the boundary (an event per
  change, or only on some changes).
- Terminal states according to the producer: states with no outgoing transition.
- When each state is emitted: on entry into the state, on every save, on retries.

## 2. Extract the consumer's interpretation

- The values it handles: `switch`, `if`, mapping tables, lookup maps, enum decode.
- What each handled value makes it do: an action, a local state it moves to, or nothing.
- Its default branch: what happens to a value it does not list.
- Its assumptions about order and finality: a state it treats as terminal (it deletes, archives,
  or ignores later updates), transitions it refuses (`if local.Status == Shipped { return }`), a
  state it expects to have seen earlier.

## 3. Build the mapping table

One row per producer state that can cross the boundary, plus one row per consumer-handled value
that the producer never sends:

```text
| Producer state | Emitted when            | Consumer handling            | Consumer meaning    | Status     | Label     |
|----------------|-------------------------|------------------------------|---------------------|------------|-----------|
| ACTIVE         | Activate()              | toEntitlement -> grant       | full access         | ALIGNED    | CONFIRMED |
| CANCELED       | Cancel()                | toEntitlement -> revoke      | no access           | ALIGNED    | CONFIRMED |
| PAST_DUE       | MarkPastDue()           | default branch: logged, acked| none                | MISALIGNED | CONFIRMED |
| (none)         | -                       | case "SUSPENDED"             | read-only access    | dead branch| CONFIRMED |
```

The table above is an illustration of the format, not a template to fill with these values.

Then compare the transition assumptions:

```text
Producer transitions      CANCELED -> ACTIVE (Reactivate)
Consumer assumption       CANCELED is final (entitlement row deleted, later events ignored)
Result                    MISALIGNED: a reactivated subscription never regains access
```

## 4. Mismatch patterns

| Pattern | What it looks like |
|---|---|
| Different names, no mapping | producer `ACTIVE`, consumer `ENABLED`; there is no code that converts one into the other |
| Missing state | producer emits `PAST_DUE`; consumer has no branch for it |
| Dead branch | consumer handles `SUSPENDED`; no producer path assigns it (drift: the producer removed or renamed it) |
| Different terminal states | producer allows `CANCELED -> ACTIVE`; consumer treats `CANCELED` as final |
| Unsupported transition | consumer rejects `TRIALING -> CANCELED` that the producer can emit |
| Order assumption | consumer requires `CREATED` before `ACTIVE`; the producer can emit `ACTIVE` for a record whose `CREATED` event was never published (show the code path) |
| Same name, different meaning | `COMPLETED` = payment captured (producer) vs order delivered (consumer) |
| Collapsed states | producer distinguishes `DECLINED` and `ERROR`; consumer maps both to `FAILED` and then retries both, or neither |

## 5. Structural vs semantic mismatch

| Kind | Test |
|---|---|
| Structural | a value the producer emits has no matching branch or mapping entry, or a consumer branch matches no producer value |
| Semantic | the value matches, but the consumer's action assumes a different point in the business process (what has happened, what can still happen) than the producer's transition means |

For a semantic claim, cite what the producer does right before it assigns the state (what has
really happened), and what the consumer does right after it reads the state (what it assumes has
happened). Without both, the claim is `INFERRED` at best.

## 6. Evidence to collect

```text
Producer   enum/constants; transition table or guards; every assignment site on a path to the boundary; emit site
Consumer   the switch or mapping; default branch; guards on local state; terminal-state handling
Mapping    adapter or translation table between them, if any
Tests      state-transition tests on either side; fixtures with state values
```

## 7. Common false positives

- **Explicit mapping.** `TRIALING -> grant` in an adapter is alignment, not a mismatch.
  Check that the mapping covers every emitted value.
- **Explicit no-op.** A consumer branch that deliberately does nothing for a state
  (`case "INCOMPLETE": return nil`) handles it.
- **Internal-only states.** A state the producer never exposes across the boundary (stored only
  in its own table) is not part of this contract.
- **A same-named type in another service.** `reporting.SubscriptionStatus` is not the billing
  service's state unless a value crosses a boundary between them. Do not compare types that no
  boundary connects.
