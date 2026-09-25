---
name: code-ahmed-likes
description: Ahmed's code taste — data and types, modules, reading order, functions, naming, comments, errors, tests, PRs. Use when writing, reviewing, or refactoring code, or when asked for a taste pass.
---

# Code Ahmed likes

Apply these while writing, not after being told. A taste pass must preserve behavior — verify end-to-end afterwards.

Order: data first (it shapes everything), then structure, then the code line by line, then restraint, then verifying and shipping.

## State and types
- As little state as possible. Before adding a field, cache, flag, or stored copy, check whether it can be computed from existing state.
- One source of truth. Derive types from it: generated database types, schema inference (`z.infer`, pydantic models), generated API clients. A good design needs few hand-written types; a pile of interfaces is a smell.
- No mapper/adapter layers; consumers use the source shape directly.
- Typing is mandatory, in dynamic languages too. In Python don't tiptoe: every signature typed, no bare `dict`/`list`, no `Any` leaking, typed models at boundaries.

## Parse, don't validate
Turn raw data (HTTP bodies, queue payloads, JSON columns, LLM output, config) into a typed model ONCE at the boundary. Everything inside receives trusted typed values and never re-checks raw data. No `dict[str, Any]` travelling through the program.

```python
# DON'T
def handle(payload: dict) -> None:
    if "order_id" not in payload: raise ValueError(...)
    fulfil(payload)                   # still a raw dict inside

# DO
def handle(payload: bytes) -> None:
    order = OrderPlaced.model_validate_json(payload)
    fulfil(order)
```

## Make illegal states unrepresentable
Model mutually exclusive states as a discriminated union, not booleans + optional fields.

```python
# DON'T: status="paid" with a decline_reason is constructible
class Payment(BaseModel):
    status: str
    decline_reason: str | None = None

# DO
class Paid(BaseModel):
    status: Literal["paid"]
class Declined(BaseModel):
    status: Literal["declined"]
    decline_reason: DeclineReason
Payment = Annotated[Paid | Declined, Field(discriminator="status")]
```

## Modules
1. Name modules by what they do, not by what kind of code they hold.
2. Colocate by feature.
3. Promote to shared only when a second real consumer shows up.
4. Never create a folder called `util` (or `utils`, `helpers`, `common`, `misc`). No ceremonial `service` layers that only forward calls.

- A module reads top to bottom in one pass and is easy to review.
- A few substantial public functions with the logic visible inline. Not a swarm of `_private` helpers interleaved with the entry points.
- Deep modules (Ousterhout), not Clean Code's tiny functions.

## Reading order
- The entry point reads like the use case, close to English: a short sequence of well-named steps.
- Each step is a deep operation (it hides real work), never shrapnel like `prepare_x()` / `do_x()` / `finish_x()`. If a step would be shallow, keep it inline instead.
- Top-down: the public entry point comes first in the module, then the functions it calls in the order it calls them. A reader never scrolls up to find what happens next.
- No `_underscore` function names in Python. If a function is worth naming, give it a plain intention-revealing name; if it's an implementation fragment, it probably shouldn't be a function.

```python
# DON'T
def _prep(d): ...
def _run_inner(d, flag): ...
def process(d):
    x = _prep(d)
    return _run_inner(x, True)

# DO
def send_invoice(order: Order) -> Invoice:
    line_items = price_line_items(order.items, order.currency)
    invoice = issue_invoice(order.customer, line_items)
    return email_invoice(invoice)
```

## Happy path first
If the happy path is ~95% of runtime behavior, it is ~95% of the code a reader sees.
- Invalid conditions leave first via guard clauses (return / raise); the valid path stays flat, never nested inside defensive branches.
- Errors propagate to the existing boundary unless recovery is a product requirement.

```python
# DON'T
if order:
    if order.items:
        if customer.active:
            return charge(order)
return None

# DO
if not order.items:
    raise EmptyOrderError(order.id)
if not customer.active:
    raise InactiveCustomerError(customer.id)
return charge(order)
```

## Functions
- A function earns its existence: real reuse or a real named concept. Otherwise inline it.
- No one-liner helpers, no pass-throughs, no single-use serialize/deserialize wrappers.
- No nested function definitions. Hoist to module level or inline.
- Each transform lives in exactly one place.
- Don't return a value the only caller ignores.

## Naming
- Name the concept (`pending_refunds`, `allowed_payment_methods`), never a bare past participle (`processed`, `filtered`).
- Full words: `customer` not `cust`, `callbacks` not `cb`.
- No domain lies (a list of invoices called `orders`).
- One word never means both a thing and its id (`Invoice` + `invoice_id`, not `invoice` for both).
- No invented ids that collide with real ones.
- `*_json` means a JSON string, not a parsed structure.
- Don't rename persisted or contract keys for taste.

## Comments
A comment exists only as a guard: it tells the next person why something non-obvious is there so they don't break it (a workaround, an ordering constraint, a deploy dependency, an upstream bug). If deleting the comment wouldn't lead anyone to break anything, delete it.
- No docstrings that restate the signature or the code below.
- No `# ── Section ──` banners or field-group comments.
- One rationale in one place, not copied across files.

## Use the established primitive
Before writing custom code, look for the primitive the platform, framework, or library already provides and that the current code is hacking around. When you find a hack, replace it with the primitive rather than polishing the hack.
- Hand-rolled retry/backoff loops → the HTTP client's, gateway's, or task queue's retries.
- `if exists: skip` checks in app code → a unique constraint / upsert.
- Manual dict key checks → a schema library (pydantic, zod).
- Hand-typed database rows → generated types.
- Regex surgery on structured text (Markdown, HTML, JSON) → a real parser.
- Polling loops → webhooks, database triggers, or queue events.
- Custom rollout/percentage logic → the feature-flag system.
- Version pins and workarounds for fixed upstream bugs → upgrade and delete the workaround.

```python
# DON'T
for attempt in range(3):
    try:
        return client.fetch(...)
    except Exception:
        time.sleep(2 ** attempt)

# DO: the client owns retries
client = Client(retries=Retry(total=3, backoff_factor=1))
return client.fetch(...)
```

## Errors and runtime
- No fallback logic. If something is wrong, raise/throw.
- Fix the upstream decision, not the symptom.
- Mark-but-keep over deleting data.
- Each concern has one owner: one layer owns retries and timeouts, the schema owns types.
- Simple reliable metrics (counts, bools, ratios) over clever derived analytics.
- Before shipping an experiment, write the query that judges it. If you can't, the recording is incomplete.

## Evidence before complexity (lightly)
Don't write code for theoretical races or edge cases. Add handling when logs, a repro, or a user report prove the case, as the smallest fix at the owning boundary. "Could / might / what if" alone never justifies code. Don't overdo it: obvious operational failures still get handled.

## No premature abstraction
Extract when real repetition shows the shared shape, not before. Duplication is cheaper than the wrong abstraction.

## Tests
Test intention, not the language or the internals. A test proves a behavior a user or caller depends on, through a stable boundary (the use case, the endpoint, the job).
- Don't test that the language or libraries work: that a dataclass stores a field, a `Literal` rejects a bad string, a dict lookup returns the value, a constant equals itself.
- Don't test one-line helpers or re-implement the logic in the assertion.
- No large mock systems for small changes. Mock only the external edge (network, third-party APIs).
- A test that can't fail when the behavior breaks, or fails when only the implementation changes, gets deleted.

```python
# DON'T: tests the interpreter and the internals
assert build_invoice_key("abc") == "invoice:abc"
assert Order(id="x").id == "x"

# DO: tests what the caller relies on
payment = checkout(order, gateway=declining_gateway)
assert payment.status == "declined"
assert payment.decline_reason == "insufficient_funds"
```

## PRs
- Minimal fix first; don't expand scope.
- One concern per PR. Refactors that can land on `main` alone go to `main`, not into a stack. Keep stacks short.
- Contract changes deploy expand → contract, so merge order doesn't matter.
- Keep `main` green: a failing test gets fixed, skipped with a ticket, or deleted.
- Rollouts ship with a kill switch and an alert.
- **Title says what changes for the user or system, not how.** If the mechanism matters, it goes after the outcome with "so" — or in the body.

  | DON'T (how) | DO (what) |
  |---|---|
  | `fix(ocr): enable layout detection in the VLM` | `fix(ocr): stop phone photos parsing to a single word` |
  | `fix(worker): register the task through the Celery include list` | `fix(worker): run the nightly cleanup job again` |
  | `fix(editor): import SOURCES in the save route` | `fix(editor): save drafts again` |
  | `fix(ui): patch reactivity in useCartStore` | `fix(cart): keep items when switching tabs` |

  Mechanism after the outcome is fine: `fix(mobile): request JPEG from the photo picker so HDR photos upload readable`.
- **Body** (after Matt Pocock's `pr` skill / Humanlayer's `show-me`). No preamble, brief prose:

```markdown
## Summary
<one or two lines>

<ONE picture: a diff of the call tree / file tree / pseudocode, or a Mermaid diagram — the smallest view that makes the point>

## Evidence
- **Before:** <screenshot / failing test / output>
  **After:** <screenshot / passing test / output>

## Merge Danger
**Door:** <one-way | two-way>
**Blast radius:** <one word> — <what could break, deploy order if any>
```
- Stage files by name (never `git add -A` / `.`). No co-author or tool-attribution trailers.
- Never merge or comment on a PR unless asked for that PR.

## Sources
- John Ousterhout, *A Philosophy of Software Design*; [Ousterhout vs. Clean Code](https://github.com/johnousterhout/aposd-vs-clean-code)
- Alexis King, [Parse, don't validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/)
- Scott Wlaschin, [Making illegal states unrepresentable](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/)
- Sandi Metz, [The Wrong Abstraction](https://sandimetz.com/blog/2016/1/20/the-wrong-abstraction)
- Casey Muratori, [Semantic Compression](https://caseymuratori.com/blog_0015)
- Carson Gross, [Locality of Behaviour](https://htmx.org/essays/locality-of-behaviour/)
- [Code like Luke](https://gist.github.com/Hona/53142c07c9decb735392f132ace34003)
- Matt Pocock, [`pr` skill](https://github.com/mattpocock/skills/tree/main/skills/in-progress/pr); Humanlayer, [`show-me`](https://github.com/humanlayer/skills/blob/main/plugins/show-me/skills/show-me/SKILL.md)
