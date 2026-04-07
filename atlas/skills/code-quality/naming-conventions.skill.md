---
name: naming-conventions
description: >
  Naming rules derived from Microsoft .NET Naming Guidelines and Martin's "Clean Code"
  Chapter 2. Covers casing rules per construct type, intent-revealing names, avoiding
  disinformation, meaningful distinctions, and domain-language alignment. Required by
  all Dev Agents and the Architecture Compliance agent (D05) reviewing code quality
  across any stack.
applies-to: [dev-agent, qa-agent, architecture-compliance, software-architect]
stack: none
layer: none
version: 1.0
---

# Naming Conventions

<context>
This skill applies whenever code is written or reviewed. Names are the primary means
by which code communicates intent to human readers. A name MUST reveal purpose, context,
and usage without requiring readers to consult implementation details. Sources: Microsoft
.NET Naming Guidelines (learn.microsoft.com) and Martin — "Clean Code" Chapter 2.
</context>

<rules>
## Casing (Language-Agnostic Principles)
- Types (classes, interfaces, structs, enums) MUST use PascalCase: `OrderRepository`, `PaymentStatus`.
- Methods and functions MUST use PascalCase in C# / camelCase in Python and TypeScript: `PlaceOrder()` / `place_order()`.
- Local variables and parameters MUST use camelCase in C#/TS: `orderTotal`, `customerId`; snake_case in Python: `order_total`.
- Constants MUST use UPPER_SNAKE_CASE in Python: `MAX_RETRY_COUNT`; PascalCase in C#: `MaxRetryCount`.
- Interfaces in C# MUST be prefixed with `I`: `IOrderRepository`, `IEmailService`.
- Async methods in C# MUST be suffixed with `Async`: `FindByIdAsync`, `SaveAsync`.
- Boolean variables and properties MUST use a positive-form predicate: `isActive`, `hasPermission`, `canRetry` — not `notDeleted`, `disabled`.

## Intent-Revealing Names (Clean Code Rule 1)
- Names MUST reveal intent: a reader MUST be able to understand purpose and usage from the name alone.
- Single-letter names MUST NOT be used except as loop counters (`i`, `j`) or in mathematical contexts (`x`, `y` in geometry).
- Abbreviated names MUST NOT be used unless the abbreviation is universally understood in the domain: `customerId` not `cid`, `orderTotal` not `orTot`.
- Names MUST NOT require a comment to explain what they mean — if a comment is needed, the name is wrong.

## Avoid Disinformation (Clean Code Rule 2)
- A name MUST NOT suggest a type that it is not: `userList` for a `HashSet` is disinformation; use `users`.
- A name MUST NOT include misleading qualifiers: `Manager`, `Processor`, `Handler`, `Info`, `Data` by themselves convey nothing — qualify with the domain concept: `OrderProcessor` not `Processor`.
- Names that differ only in small ways MUST NOT be used in the same scope: `getActiveUser` and `getActiveUsers` in the same class cause confusion.

## Meaningful Distinctions (Clean Code Rule 3)
- Names MUST differ in a meaningful way: `source` and `destination` instead of `a1` and `a2`.
- Noise words (`Data`, `Info`, `The`, `Object`) SHOULD NOT be added when they add no distinction: `OrderData` and `Order` are the same thing.

## Domain Language Alignment
- Names MUST use the Ubiquitous Language of the domain as defined in the OpenSpec or constitution.md.
- Technical layer names MUST NOT bleed into domain names: `OrderEntity`, `OrderDto` in the domain layer are violations — domain names are just `Order`.
- Names MUST be consistent across the codebase: if the domain calls it `Customer`, every layer must call it `Customer`, not `Client`, `User`, or `Buyer`.

## Function and Method Names
- Methods that change state MUST be named with an imperative verb: `Place()`, `Confirm()`, `Cancel()`, `Send()`.
- Methods that return data without side effects MUST be named with a noun or `Get`/`Find`/`List` prefix: `GetOrderById()`, `FindActiveUsers()`.
- Boolean-returning methods MUST be named as predicates: `IsEligible()`, `HasPendingItems()`, `CanBeDeleted()`.
</rules>

<patterns>
## Intent-Revealing vs. Opaque Names

```python
# OPAQUE — what does d, r, and s mean?
def calc(d, r, s):
    return d * r * (1 - s)

# INTENT-REVEALING — immediately readable
def calculate_final_price(unit_price: Decimal, quantity: int, discount_rate: Decimal) -> Decimal:
    return unit_price * quantity * (1 - discount_rate)
```

## Domain Language Alignment (C#)

```csharp
// WRONG — technical noise and inconsistent naming
public class UserEntity         { }  // "Entity" suffix pollutes domain name
public class CustomerDTO        { }  // DTO in domain — layer bleed
public class BuyerManager       { }  // "Manager" is meaningless
public class ClientInfoObject   { }  // "Info" + "Object" are noise words

// CORRECT — Ubiquitous Language, clean names
public class User               { }  // Domain entity — just the concept
public class CustomerSummary    { }  // DTO in Application/API layer — qualified by layer concern
public class OrderPlacer        { }  // Domain service — named by what it does
public class Customer           { }  // Consistent with OpenSpec terminology
```

## Boolean Predicate Naming

```csharp
// WRONG — negative form and vague
bool notDeleted;
bool disabled;
bool flag;

// CORRECT — positive predicate
bool isActive;
bool canBeDeleted;
bool hasExpired;

// Method equivalents
public bool IsEligibleForDiscount() => IsActive && LoyaltyPoints > 100;
public bool HasPendingPayments()    => _payments.Any(p => p.Status == PaymentStatus.Pending);
```

## Async suffix (C#)

```csharp
// WRONG — no Async suffix on awaitable methods
public Task<Order> GetOrder(OrderId id);
public Task Save(Order order);

// CORRECT
public Task<Order?> FindByIdAsync(OrderId id, CancellationToken ct);
public Task SaveAsync(Order order, CancellationToken ct);
```

## Python — snake_case with intent

```python
# WRONG
def proc(u, x):
    tmp = get_usr_inf(u)
    return tmp["val"] * x

# CORRECT
def calculate_invoice_total(user_id: str, quantity: int) -> Decimal:
    pricing = get_user_pricing(user_id)
    return pricing.unit_price * quantity
```
</patterns>

<anti-patterns>
- **Single-letter variables outside loops**: `u`, `p`, `r` as function parameters — impossible to understand without reading the body.
- **`Manager`, `Processor`, `Handler` as standalone class names**: convey no domain meaning; name what the class specifically manages/processes.
- **Type encoding in names**: `userString`, `priceInt`, `customerList` — type is already in the type signature; the name should convey purpose, not type.
- **Inconsistent domain terminology**: `Customer` in Domain, `Client` in Application, `User` in API — three names for the same concept fractures the Ubiquitous Language.
- **Noise word suffixes in domain layer**: `OrderEntity`, `ProductObject`, `InvoiceData` — suffixes add no meaning; domain classes are named by the concept alone.
- **Negative boolean names**: `notActive`, `disabled`, `notExpired` — logical negation of `!notActive` is a double negative; always use positive form.
- **`temp`, `data`, `result` as variable names**: used to describe every temporary variable — meaningless; name what the value represents.
- **Abbreviations that save 2 characters**: `ord` for `order`, `cust` for `customer` — savings negligible, cost in readability is real.
</anti-patterns>

<checklist>
- [ ] All names reveal intent without requiring a comment to explain them.
- [ ] No single-letter names except loop counters.
- [ ] No `Manager`, `Processor`, `Data`, `Info`, `Object` standalone suffixes without qualifying context.
- [ ] Domain names match the Ubiquitous Language defined in constitution.md / OpenSpec.
- [ ] Boolean variables/properties use positive predicate form (`isActive`, not `notDeleted`).
- [ ] Async C# methods suffixed with `Async`.
- [ ] C# interfaces prefixed with `I`.
- [ ] No type encoding in variable names (`userString`, `priceDecimal`).
- [ ] Consistent terminology across all layers (Domain, Application, API).
</checklist>
