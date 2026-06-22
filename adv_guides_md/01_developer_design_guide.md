# Ousterhout Software Design Guide for Developers

*Concise practical guide for Python backend and JavaScript/React frontend work.*

> Based on John Ousterhout's *A Philosophy of Software Design*, adapted into an action-oriented guide. This is not a chapter summary; it is a coding checklist for reducing complexity.

## Core goal

Do not optimize only for code that works today. Optimize for code that future developers can understand and modify safely.

Complexity is anything that makes code harder to understand or change. Watch for:

- **Change amplification:** a small change requires edits in many places.
- **Cognitive load:** a developer must know too many details to make a safe change.
- **Unknown unknowns:** important facts are hidden where the developer will not know to look.

Complexity usually comes from:

- **Dependencies:** code cannot be understood or changed in isolation.
- **Obscurity:** important information is not obvious from names, interfaces, structure, or comments.

## Design mindset

Prefer **strategic programming** over tactical patches.

Tactical programming asks: "What is the fastest way to make this work?"

Strategic programming asks: "What design will make this work and keep future changes easy?"

Use small, continuous design investments. Do not wait for a giant refactor. When you touch code, try to leave the touched area slightly clearer.

## Design loop

Use this loop before and during implementation.

### 1. Name the design decision

Before coding, identify the decision being introduced or changed.

Examples:

- How request parameters are parsed.
- How payment providers are hidden.
- How discounts are selected.
- How React forms represent validation errors.
- How deleted users appear in old invoices.

A design decision should usually have one owner.

### 2. Decide what knowledge should be hidden

Ask:

- What should callers not need to know?
- Is this rule repeated anywhere else?
- Is this implementation detail leaking into higher-level code?
- Would future changes happen in one place or many?

Good modules hide knowledge such as parsing rules, storage shape, provider quirks, retry behavior, validation policy, caching, and representation details.

### 3. Design the interface first

A module's interface is everything a caller must know to use it correctly: names, parameters, return values, errors, side effects, order requirements, defaults, and invariants.

A good interface is:

- Simple for the common case.
- Hard to misuse.
- Focused on caller intent, not implementation mechanics.
- Smaller than the implementation it hides.

Write the public comment or docstring before implementation. If the contract is hard to describe, the design is probably muddy.

### 4. Prefer deep modules

A deep module provides a lot of functionality behind a simple interface. A shallow module adds an interface but hides little complexity.

Smell test:

> If I delete this abstraction and inline its implementation, would the system become significantly worse?

If the answer is "not really," the module is probably shallow.

Bad shallow Python:

```python
class UserRepositoryWrapper:
    def __init__(self, repo):
        self.repo = repo

    def save_user(self, user):
        return self.repo.save(user)
```

Better deep Python:

```python
class PasswordResetTokens:
    """Issues and verifies reset tokens without exposing storage, hashing, or expiry details."""

    def issue_for(self, user_id: str) -> str: ...
    def verify(self, token: str) -> str | None: ...
```

### 5. Keep layers at different abstractions

A layer should translate concepts. It should not merely forward arguments.
Each layer (e.g. repository, service, route) should have its own vocabulary. Don't pass raw DB models up to the API layer, and don't leak HTTP concepts down into business logic.

Bad:

```python
def create_payment_intent(amount, currency, customer_id):
    return stripe.PaymentIntent.create(
        amount=amount,
        currency=currency,
        customer=customer_id,
    )
```

Better:

```python
def authorize_order_payment(order: Order) -> PaymentAuthorization:
    ...
```

The caller thinks in application terms. Provider details stay inside the adapter.

### 6. Pull repeated complexity downward

If many callers repeat the same defaults, validation, retries, null checks, or provider-specific handling, move that complexity into the lower-level module.

```txt
Repeated complexity
↓
Hide it in one place
↓
Callers become simpler
```

Bad:

```python
try:
    profile = client.fetch_profile(user_id)
except TimeoutError:
    profile = None

name = profile["display_name"] if profile else "Unknown"
```

Better:

```python
profile = profiles.get_display_profile(user_id)
name = profile.display_name
```

### 7. Separate general mechanism from special policy

Mechanism = the generic capability
Policy = the business-specific rule
Put generic things in one place and keep business rules outside of them.
General modules should not know one-off business cases. Put reusable mechanisms lower and special-purpose policy higher.

Bad python: 
```py
class EmailService:
    def send_welcome_email(self, user):
        ...

    def send_password_reset_email(self, user):
        ...

    def send_marketing_email(self, user):
        ...
```


Better python: 
```py
# Mechanism
class EmailService:
    def send(template, recipient):
        ...

# Policy
if user.just_registered:
    email_service.send(
        template="welcome",
        recipient=user.email,
    )

if user.requested_reset:
    email_service.send(
        template="password_reset",
        recipient=user.email,
    )
```


Bad React:

```jsx
export function Button({ variant, isCheckoutPrimary, ...props }) {
  const className = isCheckoutPrimary ? "bg-green-600" : styles[variant];
  return <button className={className} {...props} />;
}
```

Better React:

```jsx
export function Button({ variant = "primary", ...props }) {
  return <button className={styles[variant]} {...props} />;
}

export function CheckoutSubmitButton(props) {
  return <Button variant="success" {...props} />;
}
```

### 8. Define errors out of existence when reasonable

Errors add complexity. If callers do not need to distinguish a case, make the operation safe by design.

Most engineers think: Errors happen. I need to handle them.
Ousterhout asks: Can I design the system so this error cannot happen in the first place?


Bad:

```python
def remove_cache_key(key: str) -> None:
    if key not in cache:
        raise KeyError(key)
    del cache[key]
```

Better:

```python
def remove_cache_key(key: str) -> None:
    cache.pop(key, None)
```

Do not hide errors that callers genuinely need to handle differently.

### 9. Make code obvious

Obvious code lets readers guess correctly. Use precise names, consistent patterns, direct control flow, and comments for non-obvious facts.

Bad names:

```python
data = get_data(user)
manager.process(data)
```

Better names:

```python
subscription_status = load_subscription_status(user.id)
renewal_notice_sender.send_if_due(subscription_status)
```

Good comments explain intent, invariants, or surprising behavior. They do not repeat the code.


## Lightweight brainstorming template

Use this for nontrivial design work before coding. Keep each answer short.

```md
Goal:
- What behavior needs to exist?

Design decision:
- What rule, representation, or interface am I changing?

Hidden knowledge:
- What should callers not need to know?

Complexity risks:
- Change amplification:
- Cognitive load:
- Unknown unknowns:

Two options:
- A: smallest local change.
- B: cleaner ownership / deeper module.

Choice:
- Which option keeps the caller interface simpler?

Verification:
- Tests and edge cases.
```

## Pre-PR checklist

Before opening a PR, ask:

- What design decision did I add or change?
- Does one module clearly own that decision?
- Is the common path simple?
- Did I create a shallow wrapper?
- Did I leak implementation details to callers?
- Did I add pass-through layers, methods, or React props?
- Did I duplicate nontrivial logic?
- Could avoidable errors be made normal/idempotent?
- Are names precise and consistent?
- Are comments useful and current?
- Did I add or update relevant tests?

## Red flags

- Shallow module.
- Information leakage.
- Temporal decomposition: splitting by execution order instead of hidden knowledge.
- Overexposed API: common callers must learn rare-case details.
- Pass-through method/layer/component.
- Repetition of nontrivial logic or policy.
- Special-purpose logic inside a general module.
- Conjoined methods that must always be read together.
- Vague names.
- Hard-to-describe interface.
- Non-obvious side effects.

## Design rule of thumb

Prefer the design that minimizes what future developers must know.
