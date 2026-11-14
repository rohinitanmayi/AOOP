# Case Study 3: Bank Account and Money Transfers

## Problem Analysis

The current design uses public fields, primitive types and separate service methods. Anyone can change a balance directly, `double` amounts allow zero or negative values, and a transfer is two unrelated updates, so a crash between them loses money. The rules live in services, not in the objects, so they are easy to skip. The fix is to make `Account` and `Money` enforce their own rules, and to make a transfer one atomic unit of work.

---

### Q1. Domain invariants (6 marks)

| # | Invariant | Why it matters | If violated |
| --- | --- | --- | --- |
| 1 | Every transaction amount must be greater than zero. | A zero or negative deposit or withdrawal makes no business sense, and a negative withdrawal is really a deposit. | Customers could add money by "withdrawing" a negative amount, and the ledger fills with meaningless entries. |
| 2 | Balance after a withdrawal must not go below minus the overdraft limit. | The limit is the bank's approved credit risk for that customer. | The bank lends money it never approved and takes losses. |
| 3 | A frozen or closed account cannot be debited. | Freezing is used for fraud, legal holds or disputes. | Fraudsters or legal-hold accounts can still move money, creating legal and regulatory problems. |
| 4 | A closed account cannot be reopened or changed, and can be closed only when its balance is zero. | Closing is final and money must not be left stranded. | Money disappears into dead accounts or customers lose funds. |
| 5 | Balance always equals the sum of the ledger entries (opening balance plus credits minus debits). | The ledger is the audit record. | Statements disagree with the balance, audits fail and disputes cannot be settled. |
| 6 | A transfer must debit one account and credit another by the same amount, or do neither. | Money must be conserved. | Money is lost or created, as in the crash scenario. |
| 7 | Both sides of a transfer must be in the same currency, and the source and destination accounts must be different. | Adding rupees to dollars, or sending to yourself, is not a valid operation. | Wrong balances, or pointless self-transfers that confuse the history. |
| 8 | Ledger entries are never changed or deleted once written. | History must be trustworthy. | Fraud can be hidden and the bank cannot prove what happened. |

---

### Q2. Entities and value objects (6 marks)

Main domain concepts: Customer, BankAccount, AccountNumber, Money, Overdraft limit, Ledger entry, Fund transfer, Account status.

| Concept | Kind | Justification |
| --- | --- | --- |
| Bank Account | Entity | Has an identity (account number) that stays the same for its whole life while the balance and status change. Two accounts with the same balance are still different accounts. It is mutable and has a lifecycle (open, frozen, closed). |
| Account Number | Value object | Defined only by its value, immutable, and equal if the numbers are equal. It has no life of its own, it just identifies the account. |
| Money | Value object | Defined by amount and currency. Rs 500 is interchangeable with any other Rs 500. Immutable, so operations return a new `Money`. |
| Ledger Entry | Entity (immutable once created) | It has its own identity (entry ID or sequence number) because each transaction must be individually traceable and referred to. It can be modelled as a value inside the account aggregate if it is never referenced from outside, but it is never modified. |
| Overdraft Limit | Value object | Just an amount of money (a non-negative `Money`). Two limits of Rs 10,000 are the same. A change replaces the old limit with a new value. |
| Fund Transfer | Entity | It has an identity (transfer ID), a lifecycle (initiated, debited, completed, failed, reversed) and must be tracked and recovered after a failure. |

---

### Q3. Self-enforcing objects and Java design (8 marks)

```java
public enum AccountStatus { ACTIVE, FROZEN, CLOSED }

public final class AccountNumber {
    private final String value;
    private AccountNumber(String value) { this.value = value; }
    public static AccountNumber of(String v) {
        if (v == null || !v.matches("\\d{10,16}"))
            throw new IllegalArgumentException("Invalid account number");
        return new AccountNumber(v);
    }
    @Override public boolean equals(Object o) {
        return o instanceof AccountNumber a && a.value.equals(value);
    }
    @Override public int hashCode() { return value.hashCode(); }
    @Override public String toString() { return value; }
}

public final class Money implements Comparable<Money> {
    private final long paise;                 // exact, no floating point
    private final String currency;            // simplified; could be an enum
    private Money(long paise, String currency) { this.paise = paise; this.currency = currency; }

    public static Money ofPaise(long paise, String currency) {
        if (paise < 0) throw new IllegalArgumentException("Money cannot be negative");
        return new Money(paise, currency);
    }
    public static Money zero(String currency) { return new Money(0, currency); }

    public Money plus(Money o)  { sameCurrency(o); return new Money(Math.addExact(paise, o.paise), currency); }
    public Money minus(Money o) {
        sameCurrency(o);
        if (o.paise > paise) throw new IllegalArgumentException("Result would be negative");
        return new Money(paise - o.paise, currency);
    }
    public boolean isPositive() { return paise > 0; }
    public long paise() { return paise; }
    public String currency() { return currency; }
    private void sameCurrency(Money o) {
        if (!currency.equals(o.currency)) throw new IllegalArgumentException("Currency mismatch");
    }
    @Override public int compareTo(Money o) { sameCurrency(o); return Long.compare(paise, o.paise); }
    @Override public boolean equals(Object o) {
        return o instanceof Money m && m.paise == paise && m.currency.equals(currency);
    }
    @Override public int hashCode() { return Long.hashCode(paise) * 31 + currency.hashCode(); }
}

public final class LedgerEntry {                  // immutable
    public enum Type { DEPOSIT, WITHDRAWAL, TRANSFER_IN, TRANSFER_OUT }
    private final long sequence;
    private final Type type;
    private final Money amount;
    private final long balanceAfter;             // signed balance in paise after this entry
    private final java.time.Instant at;
    private final String reference;               // e.g. transfer ID
    // constructor package-private; getters only, no setters
    LedgerEntry(long sequence, Type type, Money amount, long balanceAfterSigned,
                java.time.Instant at, String reference) {
        this.sequence = sequence; this.type = type; this.amount = amount;
        this.balanceAfter = balanceAfterSigned;
        this.at = at; this.reference = reference;
    }
    public long sequence() { return sequence; }
    public Type type() { return type; }
    public Money amount() { return amount; }
    public long balanceAfter() { return balanceAfter; }
    public java.time.Instant at() { return at; }
    public String reference() { return reference; }
}

public final class Account {
    private final AccountNumber number;
    private final String currency;
    private final Money overdraftLimit;
    private long balance;                         // signed, in paise; may go negative within the limit
    private AccountStatus status = AccountStatus.ACTIVE;
    private final java.util.List<LedgerEntry> ledger = new java.util.ArrayList<>();

    public Account(AccountNumber number, String currency, Money overdraftLimit) {
        if (!overdraftLimit.currency().equals(currency))
            throw new IllegalArgumentException("Overdraft currency mismatch");
        this.number = number; this.currency = currency; this.overdraftLimit = overdraftLimit;
    }

    public void deposit(Money amount, String ref) {
        requireActiveOrFrozenForCredit();
        requirePositive(amount);
        balance = Math.addExact(balance, amount.paise());
        record(LedgerEntry.Type.DEPOSIT, amount, ref);
    }

    public void withdraw(Money amount, String ref) {
        requireStatus(AccountStatus.ACTIVE);      // frozen and closed are rejected
        requirePositive(amount);
        if (balance - amount.paise() < -overdraftLimit.paise())
            throw new DomainException("Overdraft limit exceeded");
        balance -= amount.paise();
        record(LedgerEntry.Type.WITHDRAWAL, amount, ref);
    }

    public void freeze()   { requireStatus(AccountStatus.ACTIVE);  status = AccountStatus.FROZEN; }
    public void unfreeze() { requireStatus(AccountStatus.FROZEN);  status = AccountStatus.ACTIVE; }
    public void close() {
        if (status == AccountStatus.CLOSED) throw new DomainException("Already closed");
        if (balance != 0) throw new DomainException("Balance must be zero to close");
        status = AccountStatus.CLOSED;
    }

    public long balance() { return balance; }
    public AccountStatus status() { return status; }
    public java.util.List<LedgerEntry> ledger() { return java.util.List.copyOf(ledger); }

    private void requirePositive(Money m) {
        if (!m.currency().equals(currency)) throw new DomainException("Currency mismatch");
        if (!m.isPositive()) throw new DomainException("Amount must be greater than zero");
    }
    private void requireStatus(AccountStatus s) {
        if (status != s) throw new DomainException("Account is " + status);
    }
    private void requireActiveOrFrozenForCredit() {
        if (status == AccountStatus.CLOSED) throw new DomainException("Account is closed");
    }
    private void record(LedgerEntry.Type t, Money amt, String ref) {
        ledger.add(new LedgerEntry(ledger.size() + 1, t, amt, balance, java.time.Instant.now(), ref));
    }
}
```

`DomainException` is a simple `RuntimeException` subclass for business rule violations.

**How the design meets each requirement**

- **Invalid amounts rejected:** `Money` cannot be negative, and `requirePositive` rejects zero. This is checked inside `Account`, so no caller can skip it.
- **No withdrawal from frozen or closed accounts:** `withdraw` begins with `requireStatus(ACTIVE)`, so it throws for any other state, and the status can only change through `freeze`, `unfreeze` and `close`.
- **Overdraft respected:** `withdraw` checks that the balance after the withdrawal is not lower than minus the overdraft limit before it changes anything.
- **No direct modification:** all fields are `private`, there are no setters, `ledger()` returns an unmodifiable copy, and `Money`, `AccountNumber` and `LedgerEntry` are immutable. The only way to change the balance is `deposit` or `withdraw`, and each of them writes a ledger entry in the same method.

---

### Q4. Money transfer and aggregate consistency (6 marks)

**Problems from the crash.** Rs 5,000 has left Account A but never reached Account B. The money has effectively vanished, A's customer is short, B's customer is not paid, and the bank cannot tell from the data alone what happened.

**Invariants violated.** Money conservation (debit and credit must match, or neither happens), and balance equals ledger consistency across the system, since a debit entry exists with no matching credit.

**Design to prevent money loss.**

- Model the transfer as one atomic unit of work. A domain/application service `TransferService.transfer(from, to, amount)` runs inside a **single database transaction**: load both accounts, call `from.withdraw(...)`, call `to.deposit(...)`, save both. If anything fails, the whole transaction rolls back, so either both changes persist or neither does. A crash before commit leaves A unchanged.
- Always lock or load accounts in a fixed order (for example by account number) to avoid deadlocks, and use optimistic locking (a version column) so concurrent updates conflict instead of overwriting each other.
- Make it idempotent by giving each transfer a unique transfer ID, so a retry after a crash does not move the money twice.

**Handling failures.** Keep a `Transfer` entity with states `INITIATED -> DEBITED -> COMPLETED` or `FAILED/REVERSED`. If the process fails after debiting, a recovery job reads transfers stuck in `DEBITED` and either completes the credit or reverses the debit with a compensating ledger entry. Nothing is deleted, the fix is a new entry.

**Different banking systems.** A single database transaction is no longer possible, because the two banks have separate systems. Then the transfer becomes a distributed workflow: a **saga** with compensation, or a two-phase style hold. We debit A (or place a hold on the funds), send a message to the other bank with the transfer ID, wait for confirmation, and if it fails or times out, reverse the debit. This gives eventual consistency, so we need idempotent messages, retries, reconciliation reports between the banks, and a "pending" state visible to customers. In practice this is done through settlement networks such as UPI or NEFT/RTGS, which give confirmed or failed status for each transaction.

---

### Q5. Ledger management and design evaluation (4 marks)

**What a ledger entry contains:** entry ID or sequence number, account number, transaction type (deposit, withdrawal, transfer in or out), amount and currency, balance after the transaction, timestamp, a reference such as the transfer ID, and a description or initiator (customer, channel, user ID).

**Modify or delete?** No. Entries are append-only and immutable. Bank records are needed for audits, disputes and legal compliance, and editing history would allow fraud to be hidden. If an entry was wrong, we add a **reversal or correction entry** that cancels it, so the full trail remains.

**Keeping balance consistent with history.** Because the account changes its balance and appends the ledger entry in the same method and the same transaction, the two cannot get out of step. A rule can also be checked: balance equals opening balance plus credits minus debits. A scheduled reconciliation job recomputes balances from the ledger and flags any mismatch. The database can also enforce a unique, increasing sequence per account.

**Challenges with a very large history.** Loading the entire list into memory becomes slow and memory heavy, and recalculating a balance by summing millions of entries is expensive. Ways to handle it: do not load the full ledger into the `Account` object, but query it with pagination; keep the running balance on the account and verify it by reconciliation; use periodic snapshots or checkpoints (for example, end-of-day balance) so only recent entries need summing; index by account and date; and archive old partitions to cheaper storage while keeping them available for audit.

---

## UML class diagram (text form)

```
Customer 1 ---- 0..* Account
Account  *---- 0..* LedgerEntry        (composition, entries belong to the account)
Account  *---- 1    AccountNumber      (value)
Account  *---- 1    Money overdraftLimit (value)
Account  ----> AccountStatus <<enumeration>>  ACTIVE | FROZEN | CLOSED
LedgerEntry ----> Money amount, LedgerEntry.Type
TransferService ..> Account (uses two accounts)
Transfer (entity) ----> Account from, Account to, Money amount, TransferStatus
```

## Design justification

The design makes invalid states hard or impossible to create. `Money` cannot be negative, `AccountNumber` cannot be malformed, `Account` is the only way to change a balance and always writes a ledger entry with it, the status machine blocks withdrawals from frozen or closed accounts, and a transfer is one atomic, idempotent operation, so money cannot be lost between two accounts.
