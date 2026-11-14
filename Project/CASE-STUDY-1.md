# Case Study 1: Money and Pricing

### Q1. Why double is bad for money (4 marks)

`double` stores numbers in binary, and most decimal fractions like 0.10 have no exact binary form. So `0.1 + 0.2` gives `0.30000000000000004`. One error is tiny, but after many additions, discounts and tax steps they pile up, which is how the cart page and the invoice end up one cent apart.

If we store the amount as a `long` in minor units (cents for USD, yen for JPY, fils for KWD), every value is a whole number. Integer addition and multiplication are exact, so 19.99 USD is just `1999`. The number of minor units comes from the currency (2 for USD, 0 for JPY, 3 for KWD), so the precision matches the currency.

One decision still has to be made openly: the **rounding mode**. Taking 15% off 19.99 gives a fraction of a cent, so the model must say how it rounds (for example HALF_EVEN) and where (per line or on the total). We put it in one method so everyone gets the same answer.

### Q2. Entity or value object (6 marks)

| Concept | Kind | Justification |
| --- | --- | --- |
| Money | Value object | No identity. Two 10.00 EUR amounts are interchangeable, equality is by amount and currency. |
| Product | Entity | It has an identity (SKU). Its name and price can change but it is still the same product. |
| Cart | Entity (aggregate root) | Identified by cart ID. Items are added and removed and it is still the same cart. |
| Percentage | Value object | Fully defined by its number, so 15% equals any other 15%. |
| Quantity | Value object | Just a positive integer with some behaviour (add, subtract). Two quantities of 3 are the same thing. |

### Q3. Percentage value object (8 marks)

```java
import java.math.BigDecimal;
import java.math.RoundingMode;

public final class Percentage {

    private final int basisPoints;          // 0..10000, so 100.00% = 10000

    private Percentage(int basisPoints) {
        this.basisPoints = basisPoints;
    }

    public static Percentage ofBasisPoints(int bp) {
        if (bp < 0 || bp > 10_000) {
            throw new IllegalArgumentException(
                "Percentage must be between 0 and 100 percent, got " + bp + " bp");
        }
        return new Percentage(bp);
    }

    public static Percentage ofPercent(int percent) {
        return ofBasisPoints(Math.multiplyExact(percent, 100));
    }

    /** Returns the price after this discount has been taken off. */
    public Money applyTo(Money price) {
        BigDecimal discount = BigDecimal.valueOf(price.minor())
                .multiply(BigDecimal.valueOf(basisPoints))
                .divide(BigDecimal.valueOf(10_000), 0, RoundingMode.HALF_EVEN);
        long result = price.minor() - discount.longValueExact();
        return Money.ofMinor(result, price.currency());   // result is never below 0
    }

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Percentage)) return false;
        return basisPoints == ((Percentage) o).basisPoints;
    }

    @Override
    public int hashCode() {
        return Integer.hashCode(basisPoints);
    }

    @Override
    public String toString() {
        return (basisPoints / 100) + "." + String.format("%02d", basisPoints % 100) + "%";
    }
}
```

**Why the constructor is private:** if anyone could call `new Percentage(250)` the range check would be skippable and the invariant would mean nothing. With a private constructor the only way to get a `Percentage` is through the factory methods, and those always validate. It also lets us name the factories clearly (`ofPercent`, `ofBasisPoints`), and the class is `final` and immutable so nobody can subclass it and break the rule.

### Q4. Should Money.plus convert currencies silently? (6 marks)

I argue **against**.

- **Purity:** right now `plus` depends only on its two arguments, so the same inputs always give the same result. With silent conversion it would also depend on today's rate, which is hidden state that changes over time. `a.plus(b)` could give different answers on different days.
- **Testability:** tests would need a rate service or a clock, and any test that hits a real rate would break when the rate moves. A plain `plus` can be tested with two values and an expected result.
- **Invariants:** the rule is that money in different currencies cannot be added directly. Silent conversion hides that. It also adds a new failure mode (rate not available) to something that should never fail, and rounding in the conversion means `a.plus(b)` and `b.plus(a)` might not agree, so the cart and invoice could drift again.

Better design: keep `plus` strict (throw `CurrencyMismatch`) and do conversion in a domain service, e.g. `converter.convert(money, targetCurrency, rate, date)`, which returns a new `Money`. The caller converts first and then adds, so the rate and date are visible in the code.

### Q5. Refunds without negative prices (6 marks)

Split the idea of "a price" from "a ledger amount":

- `Money`: stays non-negative. Products, cart lines and prices use only this type, so a negative price cannot be constructed.
- `SignedMoney` (or `Adjustment`): holds a signed `long` minor amount plus a currency. It allows negatives and supports `plus`, `minus` and `negate`, with the same currency check.
- `Refund` (optional): wraps a `Money` and a reason. It is positive by construction and is turned into a negative `SignedMoney` only when posted to a ledger.

```java
final class SignedMoney {
    private final long minor;
    private final Currency currency;
    static SignedMoney credit(Money m) { return new SignedMoney(-m.minor(), m.currency()); }
    static SignedMoney charge(Money m) { return new SignedMoney(m.minor(), m.currency()); }
    Optional<Money> asMoney() { return minor >= 0 ? Optional.of(Money.ofMinor(minor, currency)) : Optional.empty(); }
}
```

`Product.price` is typed `Money`, so the compiler stops a negative price. Refunds live in `SignedMoney`, and converting back to `Money` is explicit and can fail, so a negative value cannot slip into a price by accident.

