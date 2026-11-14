# Case Study 2: Order Lifecycle

### Q1. Illegal states in the naive Order (6 marks)

| Reachable bad state | Invariant violated |
| --- | --- |
| `shipped = true` and `cancelled = true` | Shipped orders cannot be cancelled / a cancelled order can never ship (transition rules) |
| `status = "DRAFT"` but `shippedAt` is set | shippedAt exists if and only if the order has been shipped |
| `status = "SHIPPED"` with an empty `lines` list | A placed (and later) order has at least one line |
| `total` does not match the lines after `lines.add(...)` on the public list | Total always equals the sum of line totals |
| `status = "SHIPPED"` while `paid = false` | Transitions must follow Draft, Placed, Paid, Shipped, Delivered (no skipping) |
| Lines added or removed on a PAID or SHIPPED order | Lines can change only while the order is a draft |
| Two `OrderLine` entries for the same product | Lines are unique by product, adding again increases quantity |

### Q2. Transition table and `ship` (8 marks)

| From \ Action | place | pay | ship | deliver | cancel |
| --- | --- | --- | --- | --- | --- |
| DRAFT | PLACED | not allowed | not allowed | not allowed | CANCELLED |
| PLACED | not allowed | PAID | not allowed | not allowed | CANCELLED |
| PAID | not allowed | not allowed | SHIPPED | not allowed | CANCELLED |
| SHIPPED | not allowed | not allowed | not allowed | DELIVERED | not allowed |
| DELIVERED | not allowed | not allowed | not allowed | not allowed | not allowed |
| CANCELLED | not allowed | not allowed | not allowed | not allowed | not allowed |

```java
void ship(TrackingNumber t, Instant at) {
    requireStatus(Status.PAID);                       // guard
    shipment = Optional.of(new Shipment(t, at));
    status = Status.SHIPPED;
}

private void requireStatus(Status expected) {
    if (status != expected) {
        throw new DomainException(
            "Expected status " + expected + " but order is " + status);
    }
}
```

**How it stops a cancelled order shipping:** `ship` only runs when the status is exactly `PAID`. Once `cancel()` has run, the status is `CANCELLED`, so `requireStatus(PAID)` fails and throws before anything is changed. The shipment is never set. Because `status` and `shipment` are private and only changed inside these methods, there is no other path to set a shipped state.

### Q3. Model A vs Model B (8 marks)

- **Type safety:** B is stronger. In B, `ShippedOrder` simply has no `cancel` method, so calling it will not compile. In A the same mistake is only caught at runtime, by an exception.
- **Code volume:** A is smaller, one class and a guard. B needs a class per state plus a transition method for each allowed move, and the code grows with every state.
- **Persistence:** A maps to one table or entity with a status column. B needs a discriminator and a mapping for each state type, and loading means building the right class from a row.

**Recommendation for eight states that change often: Model A.** With that many states, B means eight classes, and every new state or new transition touches the sealed interface, many classes and every `switch` over them. The type safety is nice, but the maintenance cost is high when the lifecycle keeps changing. A keeps all the rules in one place, can use a transition table, and is simple to persist. To make up for weaker compile-time checks, we cover every transition with unit tests, including the illegal ones. B is a better fit when there are few, stable states with really different data and behaviour.

### Q4. Why compute total instead of storing it (4 marks)

If total is a stored field, it is a second copy of information already in the lines. Any change to the lines that forgets to update it makes the two disagree, which is the exact bug in production. Computing it from the lines means there is only one source of truth, so it cannot drift.

Storing it can be justified when (a) the order is placed and we want a frozen snapshot of what the customer was charged, even if prices or rules change later, or (b) we need to query or sort by total in the database without loading every line.

Then we must protect an extra invariant: **stored total always equals the sum of the line totals.** We keep it by only updating the total inside the Order's own methods (recalculate whenever a line changes), never exposing a setter or a modifiable list, and for snapshots, freezing it at `place()` so lines can no longer change afterwards.

### Q5. Credit limit rule (6 marks)

This is **not an Order invariant.** An invariant must be checkable from the data inside the one aggregate. The credit limit depends on the customer's limit and on the amount of all their other open orders, and none of that lives inside a single Order.

**Where it should live:** in the Customer (or a `CreditAccount`) aggregate, which owns the limit and the credit already used. A method like `customer.reserveCredit(amount)` checks `used + amount <= limit` and records the reservation. An application service or domain service coordinates: reserve credit on the customer, then place the order, and if placing fails, release the reservation (compensation). Between the two aggregates the system is eventually consistent, not a single atomic step.

**Avoiding the race between two simultaneous orders:** both requests may read the same `used` value and both pass the check. To stop that, the Customer aggregate carries a **version number (optimistic locking)**. When both try to save, the second save fails because the version has changed. It then reloads the customer, re-runs the check with the new `used` value, and either succeeds or is rejected for exceeding the limit. A pessimistic lock on the customer row would also work, at the cost of throughput.
