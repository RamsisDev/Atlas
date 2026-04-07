---
name: refactoring-patterns
description: >
  Named refactoring patterns from Fowler's "Refactoring" 2nd edition and Feathers'
  "Working Effectively with Legacy Code." Covers when and how to apply the most
  commonly needed refactorings: Extract Function, Extract Class, Replace Conditional
  with Polymorphism, Introduce Parameter Object, and dependency-breaking techniques.
  Required by Dev Agents cleaning up existing code and the Architecture Compliance
  agent (D05) reviewing structural quality.
applies-to: [dev-agent, software-architect, architecture-compliance]
stack: none
layer: none
version: 1.0
---

# Refactoring Patterns

<context>
This skill applies when improving the internal structure of existing code without
changing its observable behavior. Refactoring is always preceded by a green test suite
and followed by re-running that suite. Never apply refactoring and behavior changes
in the same commit. Source: Fowler — "Refactoring" 2nd ed. (refactoring.com/catalog)
and Feathers — "Working Effectively with Legacy Code."
</context>

<rules>
## Preconditions
- Refactoring MUST only begin when tests are passing (green).
- Each refactoring step MUST be small enough to keep tests green continuously.
- Behavior-changing fixes MUST NOT be combined with structural refactoring in the same commit.
- A refactoring MUST be motivated by a code smell, not by personal preference.

## When to Refactor
- Code SHOULD be refactored when: it is difficult to understand, it is difficult to change, or it duplicates logic that already exists elsewhere.
- Refactoring SHOULD NOT be performed on code that will not be changed or extended in the near future (premature cleanup).
- Refactoring MUST be scoped to the code touched by the current task; adjacent code SHOULD be left as found unless it directly impedes the task.

## Safety
- Before refactoring untested legacy code, characterization tests MUST be written to document existing behavior (Feathers' technique).
- Automated refactoring tools (IDE rename, extract method) SHOULD be preferred over manual edits to reduce error risk.
- Each refactoring step MUST be committed independently so it can be reverted without reverting behavior changes.
</rules>

<patterns>
## Extract Function — when a code block has a comment describing it

```python
# BEFORE — inline logic with explanatory comment
def process_order(order):
    # calculate discount
    if order.customer.is_premium and order.total > 100:
        discount = order.total * 0.1
    else:
        discount = 0
    final_price = order.total - discount
    # send confirmation
    email_client.send(order.customer.email, f"Your order total: {final_price}")

# AFTER — extracted functions, comment replaced by name
def calculate_discount(order) -> float:
    if order.customer.is_premium and order.total > 100:
        return order.total * 0.1
    return 0.0

def process_order(order):
    final_price = order.total - calculate_discount(order)
    send_order_confirmation(order.customer.email, final_price)
```

## Replace Conditional with Polymorphism — when switch/if-else dispatches on type

```python
# BEFORE — type switch that grows with every new type
def render_widget(widget):
    if widget.type == "button":
        return f'<button>{widget.label}</button>'
    elif widget.type == "input":
        return f'<input placeholder="{widget.placeholder}"/>'
    elif widget.type == "checkbox":
        return f'<input type="checkbox" {"checked" if widget.checked else ""}>'

# AFTER — polymorphism; new types add a new class, not a new branch
from abc import ABC, abstractmethod

class Widget(ABC):
    @abstractmethod
    def render(self) -> str: ...

class Button(Widget):
    def __init__(self, label: str): self.label = label
    def render(self) -> str: return f'<button>{self.label}</button>'

class InputField(Widget):
    def __init__(self, placeholder: str): self.placeholder = placeholder
    def render(self) -> str: return f'<input placeholder="{self.placeholder}"/>'

def render_widget(widget: Widget) -> str:
    return widget.render()
```

## Introduce Parameter Object — when multiple parameters always travel together

```csharp
// BEFORE — DateRange as loose parameters passed everywhere
public IEnumerable<Order> GetOrdersBetween(DateTime from, DateTime to, string status) { ... }
public decimal SumRevenueBetween(DateTime from, DateTime to) { ... }

// AFTER — DateRange value object encapsulates the concept
public record DateRange(DateTime From, DateTime To)
{
    public bool Contains(DateTime date) => date >= From && date <= To;
}

public IEnumerable<Order> GetOrdersIn(DateRange range, string status) { ... }
public decimal SumRevenueIn(DateRange range) { ... }
```

## Extract Class — when one class has two sets of unrelated responsibilities

```csharp
// BEFORE — Order handles both order state AND shipping address logic
public class Order
{
    public string Street { get; set; }
    public string City { get; set; }
    public string PostalCode { get; set; }
    public string FormatShippingLabel() => $"{Street}\n{City} {PostalCode}";
    // ... order state methods
}

// AFTER — Address extracted as a Value Object
public record Address(string Street, string City, string PostalCode)
{
    public string FormatLabel() => $"{Street}\n{City} {PostalCode}";
}

public class Order
{
    public Address ShippingAddress { get; private set; }
    // ... order state methods only
}
```

## Characterization Test (Feathers) — before refactoring untested legacy code

```python
# Before touching legacy_process(), write a test that documents current behavior
def test_legacy_process_characterization():
    """Documents current behavior. Do not change this test — change the code."""
    result = legacy_process(input_fixture)
    # Whatever it returns now, lock it in:
    assert result == "expected current output"
    # Now refactor legacy_process safely with a green test
```
</patterns>

<anti-patterns>
- **Refactoring + bug fix in same commit**: if a regression occurs, impossible to tell whether the refactoring or the fix caused it — separate commits always.
- **Refactoring without tests**: high risk of silent behavior change; write characterization tests first for untested legacy code.
- **Extracting a function to give it a generic name**: `do_stuff()`, `process_data()` — extraction without a meaningful name makes things worse.
- **Extracting too eagerly**: extracting a 3-line block into a function it will only ever be called from once — DRY applied incorrectly; wait for the second usage.
- **Refactoring unrelated code "while you're here"**: scope creep that makes the diff unreadable and harder to review — refactor only what the task touches.
- **Replacing a conditional with polymorphism when there are only 2 cases**: polymorphism adds indirection without value for simple binary conditions — use it when the number of cases is 3+ and growing.
</anti-patterns>

<checklist>
- [ ] Tests are green before refactoring begins.
- [ ] Each refactoring step committed separately from behavior changes.
- [ ] Motivation is a named code smell (long function, duplicated logic, feature envy, etc.).
- [ ] Refactoring scoped to code touched by the task — no opportunistic cleanup of unrelated code.
- [ ] Characterization tests written for untested legacy code before refactoring.
- [ ] Automated IDE refactoring tools used where available (rename, extract method).
- [ ] Tests remain green after every individual refactoring step.
</checklist>
