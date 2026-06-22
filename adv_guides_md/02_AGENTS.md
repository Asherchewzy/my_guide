# AGENTS.md — Software Design Rules for AI Coding Agents

> Use this as an always-loaded, concise guide for coding agents. Optimize for correct, scoped changes that reduce complexity.

## Mission

Implement the requested behavior while preserving long-term modifiability.

Priority order:

1. Correctness and safety.
2. Preserve existing behavior unless asked to change it.
3. Keep the diff scoped.
4. Reduce complexity.
5. Keep interfaces simple.
6. Update tests and comments when behavior changes.

## Before editing

1. Understand the request and existing code.
2. Find nearby patterns before creating new ones.
3. Identify the design decision being changed.
4. Identify which module should own that decision.
5. For nontrivial interface/design changes, compare two designs briefly.

Do not start by rewriting. Inspect first.


## Brainstorming protocol

Use this only for nontrivial changes, especially new abstractions, public interfaces, cross-module behavior, or refactors. Keep it short.

```md
Goal:
- What behavior needs to exist?

Design decision:
- What rule, representation, or interface is being introduced/changed?

Current ownership:
- Where does this knowledge live now?
- Is it duplicated or leaked?

Complexity risks:
- Change amplification:
- Cognitive load:
- Unknown unknowns:

Options:
- Option A: smallest local change.
  - Pros:
  - Cons:
- Option B: better information hiding / deeper module.
  - Pros:
  - Cons:

Choice:
- Pick the design with the simpler caller interface and clearer ownership.

Verification:
- Tests to add/run:
- Edge cases:
```

Default to Option A for tiny safe changes. Use Option B when it clearly reduces future complexity without expanding scope too much.

## Design rules

### Reduce complexity

Watch for:

- **Change amplification:** one small change requires many edits.
- **Cognitive load:** callers must know too many details.
- **Unknown unknowns:** important behavior is hidden or surprising.

Prefer designs that reduce dependencies and make important facts obvious.

### Make modules deep

**Prefer deep modules over shallow ones.** A module should hide significant complexity behind a simple interface. Avoid splitting logic into many small functions or classes just for the sake of it — each abstraction should earn its existence.

A module should hide meaningful complexity behind a simple interface.

Smell test:

> If this abstraction were deleted and inlined, would the system become significantly worse?

If not, it is probably shallow.

Avoid wrappers that only rename or forward calls.

### Hide information

Each important design decision should live in one place.

Hide:

- parsing rules
- validation policy
- storage shape
- provider SDK details
- retry/caching behavior
- error normalization
- React API response shapes

### Different layer, different abstraction

- **Different layer, different abstraction.** Each layer (e.g. repository, service, route) should have its own vocabulary. Don't pass raw DB models up to the API layer, and don't leak HTTP concepts down into business logic.

A layer must translate concepts. Do not create pass-through services, methods, or components.

Bad:

```python
def create_payment_intent(amount, currency, customer_id):
    return stripe.PaymentIntent.create(amount=amount, currency=currency, customer=customer_id)
```

Better:

```python
def authorize_order_payment(order: Order) -> PaymentAuthorization:
    ...
```

### Pull complexity downward

If many callers repeat the same checks, defaults, retries, or conversions, move that work into the lower-level module.

### Keep general and special separate

**General-purpose over special-purpose.** When writing a module, ask whether a slightly more general version would serve multiple use cases without added complexity. Avoid one-off abstractions.

General utilities should not contain one-off product rules. Put special policy near the caller or in a policy object.

### Define errors out of existence

**Define errors out of existence where possible.** Prefer designs that make invalid states unrepresentable over scattered defensive checks. When errors must exist, handle them at the right layer — not everywhere.
If callers do not need to distinguish a case, make the operation safe/idempotent.

```python
def discard_coupon(cart, coupon_id):
    cart.coupons.discard(coupon_id)  # absence is already the desired state
```

Do not hide errors that callers genuinely need.

## Python rules

- Prefer clear domain/value objects over raw dictionaries when rules matter.
- Keep web/controller code thin; move domain decisions into domain/service modules.
- Do not leak provider SDK objects past adapter boundaries.
- Use precise names: `invoice_preview`, `order_total_cents`, `password_reset_token`.
- Avoid vague names: `data`, `info`, `manager`, `helper`, `processor`.

## React rules

- Keep raw API response shapes near the data boundary.
- Map API data into view models before passing deep into components.
- Avoid prop drilling that merely passes values through many layers.
- Keep generic UI components free of page-specific business rules.
- Use hooks/route components for data loading when that matches the project pattern.

Bad:

```jsx
<UserBadge user={apiResponse.data.relationships.user.attributes} />
```

Better:

```jsx
<UserBadge displayName={userView.displayName} avatarUrl={userView.avatarUrl} />
```

## Comments and names

Write comments for non-obvious contracts, invariants, tradeoffs, and surprising behavior. Do not repeat the code.

If a public interface is hard to describe simply, reconsider the design.

## Scope control

Do not:

- Rewrite unrelated code.
- Add new dependencies without approval.
- Change public APIs unless required.
- Perform broad refactors inside a local fix.
- Add abstractions that hide no complexity.
- Apply design patterns mechanically.
- Reformat unrelated files.

Ask before destructive changes, migrations, new dependencies, security-sensitive changes, or large refactors.

## Testing

- Add or update tests for changed behavior.
- Run the nearest relevant test first.
- Run broader checks if the change is cross-cutting.
- If tests cannot be run, explain why.

## Self-review before final response

Check:

- Is the change scoped to the request?
- Does one module own each design decision?
- Did I introduce shallow wrappers or pass-through layers?
- Did I duplicate policy or parsing rules?
- Did I expose implementation details to callers?
- Is the common case simple?
- Are names precise?
- Are comments/tests updated?

## Final response format

Respond with:

- What changed.
- Design notes or tradeoffs.
- Tests run.
- Risks or follow-up, if any.
