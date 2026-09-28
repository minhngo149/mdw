# Branching

A branch is a decision that sends execution down one of several paths. Flattening branches into
one line ("charges the payment") hides the paths that behave differently. Enumerating every
combination buries the reader. This reference covers finding decisions, recording each arm,
checking which arms can be reached, and choosing the paths to document.

## Contents

1. Kinds of decisions
2. Recording a decision
3. Which arms to include
4. Reachability
5. Paths, not combinations
6. Representation

## 1. Kinds of decisions

| Kind | Code shapes | Watch for |
|---|---|---|
| Conditional | `if / else if / else`, ternary, guard | the `else` that is missing: when no arm matches, execution continues silently |
| Switch or match | `switch`, `case`, `when` (Kotlin), `match` (Rust, Python, Scala) | **fall-through.** C, C++, Java, JavaScript, and C# (for empty cases) fall into the next case without `break`. Go does not, unless the case says `fallthrough`. Also a missing `default`. |
| Type dispatch | Go type switch, `instanceof`, `isinstance`, sealed-class `when`, pattern matching | which concrete types can reach it on this path |
| Polymorphic dispatch | an interface call whose implementation depends on the input (a strategy chosen per request) | each implementation that input can select is an arm |
| Table dispatch | `handlers[kind]`, `strategies.get(method)`, a route table, a command bus | the key's source, and what happens for a missing key (nil function, `KeyError`, default handler) |
| Configuration and feature flags | `if cfg.FeatureX`, `flags.Enabled("x")`, profiles, environment checks | decided at deploy time or at runtime. The in-repo default is `CONFIRMED`; the deployed value is `UNKNOWN`. Document both arms. |
| Error checks | `if err != nil`, `try/catch`, `.catch`, `Result` matching | usually an error exit (`x-->`), not a full branch. It becomes a full branch when the error arm makes further calls (compensation, fallback, retry). |
| Result-based checks | rows affected `== 0`, cache hit or miss, `ok` from a map lookup, HTTP status switch | often the most important decision in the function, and easy to miss inside a helper |

## 2. Recording a decision

Give each decision an ID (`B1`, `B2`, ...) and record:

```text
B1  <file> <Symbol>: <condition as written in code>
    Arm           Guard            Executes                          Outcome
    6a            p.Method=CARD    PSPClient.Refund()                refund at PSP; continue to 7
    6b            STORE_CREDIT     Ledger.Credit()                   ledger row; continue to 7
    6x            default          -                                 ErrUnsupportedMethod -> 422
```

- **Condition**: quote the expression (short) or paraphrase it faithfully, and include the
  variable and where its value came from (`p.Method`, loaded at step 4).
- **Executes**: the calls in that arm, with step IDs.
- **Outcome**: where control goes next: continue to step N, return a value, exit with an
  error.
- **Label**: an arm is `CONFIRMED` when you read its code. Reachability is a separate claim
  (section 4).

## 3. Which arms to include

- Include every arm that leads to a **distinct outcome**: different calls, a different state
  written, a different response, or a different error.
- Group arms that behave identically: `[CARD, DEBIT_CARD] -> PSPClient.Refund()`.
- An arm that does nothing is still an outcome. Write it: `[MANUAL_REVIEW] no call; status stays
  REQUESTED`.
- Record the `default` or `else` arm, or its absence: "no default: an unknown method falls
  through and the refund completes with no money moved" is a critical finding.
- Error checks that only return the error stay as `x-->` exits on the step. They appear in Error
  Paths, not as branches.

## 4. Reachability

An arm exists in the code; whether this operation can reach it is a separate question.

1. Find where the condition's value comes from on this path: request input, validation, a
   database value, config, a constant.
2. Check whether earlier steps constrain it. A validator that accepts only `CARD` and
   `STORE_CREDIT` makes a `GIFT_CARD` case unreachable on this path.
3. Record the result per arm: **reachable** (with the input that reaches it), **not reachable
   on this path** (with the constraint's evidence), or `UNKNOWN` (the value comes from data or
   config you cannot see).

Never delete an arm silently because you think it is unreachable. List it with the reason.
Unreachable arms often show drift or dead code. Code that looks related but is never reached goes
in "Not on this path".

## 5. Paths, not combinations

Three decisions with three arms each make 27 combinations. Document decisions once each, then
name the few paths a reader needs:

```text
| Path | Decisions taken | Outcome | Label |
|---|---|---|---|
| P1 happy path (card) | B1=CARD, B2=full amount | 201 COMPLETED; PSP refund; T1 committed | CONFIRMED |
| P2 store credit | B1=STORE_CREDIT | 201 COMPLETED; ledger row; T1 committed | CONFIRMED |
| P3 PSP declined | B1=CARD, PSP 402 | 402; nothing written | CONFIRMED |
| P4 T1 fails after PSP refund | B1=CARD, T1 error | 500; PSP refund done, no refund row | CONFIRMED |
```

Choose: the happy path, each business alternative, each failure with a distinct outcome
(especially partial ones like P4), and any path the user asked about. When the operation named in
the request is a condition ("when payment succeeds"), that condition's arm is the main path. Show
the other arms briefly.

## 6. Representation

In the main sequence and call tree, show the decision at the step where it happens, with its
arms indented under it (notation in `sequence-format.md`). Draw a separate branch diagram (`B1`)
when an arm contains more than two steps or its own nested decision. Past three levels of
nesting, give the inner decision its own diagram and reference it (`see B3`).

```text
RefundService.ProcessRefund()
    |
    +-- refund method?   switch p.Method   (B1)
          |
          +-- [CARD]           +--> PSPClient.Refund()
          +-- [STORE_CREDIT]   +--> Ledger.Credit()
          +-- [default]        x--> ErrUnsupportedMethod   (not reachable: validate() rejects others)
```
