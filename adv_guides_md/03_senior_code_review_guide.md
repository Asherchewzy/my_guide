# Ousterhout Code Review Guide for Senior Developers

*Concise review guide for protecting long-term codebase quality.*

> Based on John Ousterhout's *A Philosophy of Software Design*, adapted into a compact PR review checklist.

## Review goal

Do not review only for correctness. Review for future modifiability.

A good PR should:

- Preserve or improve clarity.
- Keep design decisions in clear owners.
- Minimize caller knowledge.
- Avoid unnecessary abstractions.
- Make future related changes easier.

## Review sequence

### 1. Understand the design decision

Ask:

- What behavior changed?
- What design decision was introduced?
- Which module owns that decision?
- Would a future developer know where to change it?

Review prompt:

> What complexity does this PR remove, hide, or add?

### 2. Check change amplification

A small rule change should not require edits across controllers, services, React components, analytics, and tests.

Ask:

- Is the same rule repeated?
- Is the same constant repeated?
- Is the same parsing/validation logic repeated?
- Is frontend duplicating backend policy?

Review comment:

> This rule appears in multiple places. Can we move ownership to one policy/helper/module so future changes are one edit?

### 3. Check module depth

Ask:

- What complexity does this abstraction hide?
- Is the interface much simpler than the implementation?
- If we delete and inline it, would the system become significantly worse?

Bad:

```python
class UserService:
    def __init__(self, repo):
        self.repo = repo

    def save(self, user):
        return self.repo.save(user)
```

Better:

```python
class UserRegistration:
    """Registers users while hiding email normalization, password hashing, profile creation, and events."""

    def register(self, command: RegisterUser) -> RegisteredUser:
        ...
```

Review comment:

> This wrapper looks shallow. What knowledge does it hide? If none, can we remove it or give it real ownership?

### 4. Check information hiding

Ask:

- Are callers exposed to storage shape, API response shape, provider SDK details, or internal state?
- Are getters/setters exposing representation instead of behavior?
- Does a React component depend on raw API data?

Bad React:

```jsx
<UserBadge user={apiResponse.data.relationships.user.attributes} />
```

Better:

```jsx
<UserBadge displayName={userView.displayName} avatarUrl={userView.avatarUrl} />
```

Review comment:

> This component now depends on the API response shape. Can we map to a view model at the data boundary?

### 5. Check layer boundaries

A layer should add a different abstraction, not just pass calls through.

Ask:

- Does this service API mirror the provider SDK?
- Does this component only pass props through?
- Does this method only call another method with the same arguments?

Review comment:

> This layer seems pass-through. Can we either remove it or reshape it around the application concept?

### 6. Check together vs apart

Splitting code is good only when the pieces hide different knowledge.

Ask:

- Were classes/functions split by execution order only?
- Do separated methods share the same assumptions?
- Must callers call methods in a fragile sequence?
- Is general-purpose code mixed with special-case policy?

Review comment:

> These modules are split by phase, but all know the same schema. Could one importer own the schema and use private helpers internally?

### 7. Check error design

Ask:

- Are callers forced to handle avoidable errors?
- Can the operation be idempotent?
- Are low-level exceptions leaking upward?
- Is the same error handling repeated in many callers?

Bad:

```python
def remove_coupon(cart, coupon_id):
    if coupon_id not in cart.coupons:
        raise CouponNotFound(coupon_id)
    cart.coupons.remove(coupon_id)
```

Better:

```python
def discard_coupon(cart, coupon_id):
    cart.coupons.discard(coupon_id)
```

Review comment:

> If the desired final state is "coupon absent," absence may not need to be an error.

### 8. Check obviousness

Ask:

- Can a reader guess behavior correctly from names and structure?
- Are side effects explicit?
- Is control flow direct?
- Is state mutation localized?
- Are comments explaining non-obvious intent?

Bad comment:

```python
# Save user
user.save()
```

Better comment:

```python
# Keep disabled users for audit history; login filtering happens in AuthPolicy.
user.disable()
```

### 9. Check names

Vague names often hide unclear design.

Question names like:

- `data`
- `info`
- `result`
- `manager`
- `helper`
- `processor`
- `handleThing`

Prefer names like:

- `invoice_preview`
- `payment_authorization`
- `order_total_cents`
- `selected_text_range`
- `password_reset_token`

Review comment:

> `data` changes meaning in this function. Can we name the concept directly?

### 10. Check tests as design feedback

Ask:

- Are tests hard to write because the design is hard to use?
- Do tests patch many internals to exercise one behavior?
- Are tests coupled to implementation details?
- Did behavior change without test updates?

Review comment:

> This test needs to patch several internals. Is there a deeper public interface we should test through?

## Red flag table

| Red flag | Smell test | Review action |
|---|---|---|
| Shallow module | Would inlining be almost as good? | Remove it or give it real ownership. |
| Information leakage | Does one rule appear in many places? | Move the rule to one owner. |
| Pass-through layer | Does it only forward arguments? | Delete, merge, or change abstraction. |
| Temporal decomposition | Is code split by execution phase? | Group by hidden knowledge instead. |
| Overexposed API | Must common callers know rare details? | Make common path simple. |
| Special/general mixture | Does generic code know one-off policy? | Move policy upward or into policy object. |
| Conjoined methods | Must methods always be read/called together? | Combine or enforce sequencing. |
| Vague name | Could the name mean many things? | Rename or redesign. |
| Hard-to-describe API | Is the docstring full of caveats? | Simplify the interface. |
| Non-obvious code | Are side effects or dependencies hidden? | Make them explicit or document intent. |

## Approve when

- Behavior is correct.
- Tests are sufficient.
- The diff is scoped.
- The common case is simple.
- Design decisions have clear owners.
- New abstractions hide real complexity.
- Names/comments reduce cognitive load.

## Request changes when

- The PR spreads a rule across many modules.
- It introduces shallow abstractions.
- It exposes provider/storage/API details to callers.
- It creates pass-through layers.
- It makes rare cases dominate common usage.
- It hides side effects or important dependencies.
- Tests reveal awkward design.

## Best review question

> What should future developers not have to know because of this design?
