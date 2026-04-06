---
name: abizer-code-review
description: >
  Review code the way abizer does — applies to arini-ai/hire-ai codebases. Use this skill
  whenever reviewing a pull request, leaving inline comments, checking code quality, or
  being asked to do a code review on Python backend code. Also use when asked to evaluate
  whether code follows team conventions, or when writing new Python code for the arini-ai
  codebase that should conform to established patterns.
---

## Core Lens

Three questions, in order, before writing any comment:

1. **Is it necessary?** Can this be deleted?
2. **Is it honest?** Does the name, type, or abstraction tell the truth?
3. **Will the next person be surprised?** If something demands explanation but
   offers none, flag it.

For deeper context on the philosophy behind these principles, see
`references/engineering-sop.md`.

---

## I. Code is a liability — earn every line

*"The goal is to delete as much of it as we can."*

Every line of code is a maintenance burden. It has to be read, understood,
tested, and debugged by every person who touches the file after you. The bar
for adding code should be high, and the instinct to remove it should be strong.

### Delete what doesn't earn its existence

Passthrough wrappers — functions that forward parameters without modification —
should be deleted; call the underlying function directly.

> "seems like a no-op passthrough function that should be deduped"

> "it's also unclear what exactly this function is helping with — we're
> accepting the same parameters as the underlying method call and directly
> forwarding them without modification."

### Iterate a collection once

Never run the same computation twice on unchanged data. Two passes over the
same collection to extract different things is a logic error dressed as style
— combine into a single loop. Nested inner loops that iterate over the same
data redundantly are especially suspect.

**Before** *(from PR #3595)*: three separate loops over `protocol_steps` to
check for missing steps and build `filtered_steps`.

**Comment:** "iterating over the same loop three times. should be:"

**After**
```python
filtered_steps = []
missing_closing = True
missing_end_call = True

for step in protocol_steps:
    match step.step_id.name:
        case "end_call":
            missing_end_call = False
        case "closing_remarks":
            missing_closing = False
    filtered_steps.append(step)

if missing_closing and missing_end_call:
    if end_call := steps_by_id.get(DefaultStepID.END_CALL):
        filtered_steps.append(end_call)
```

**Before** *(from PR #3575)*: iterating `appointment_slots_by_protocol` once
to extract uids, again to mutate slots.

**Comment:** "why iterate over this dict twice? we throw away uid in the first
iteration and use it in the second, we could just:"

**After**
```python
for uid, slots in appointment_slots_by_protocol.items():
    for slot in slots:
        slot.provider_name = provider_id_to_name.get(slot.provider_id, "")
    store.add_slots_for_protocol(uid, slots)
```

Use the walrus operator for guard-check-act patterns. Use `match/case` over
`if/elif` chains for dispatch on a finite set of cases. Use standard library
tools (`defaultdict`, `Counter`, `itertools.chain`) rather than reimplementing
them.

### Right-size everything: density matches complexity

Nothing we are doing is too complicated. If code looks hacky, convoluted, or
difficult, there needs to be a good reason — otherwise it needs to be redone
in a simpler way.

**Too big:**
- Functions longer than ~200 lines need a good reason.
- Functions with more than three levels of indentation are too complex.
- Files longer than ~1000 lines almost never have a good reason.
- Classes that accumulate methods from unrelated concerns should be split.
- Code that branches widely is extremely hard to test — keep it narrow.

**Too small:**
- Files and classes split apart for no reason add indirection without value.
- A file named the same as the single function inside it is not a meaningful
  organizational unit — that function probably belongs in a neighboring file.

The goal is tasteful density: code structure should match the inherent
complexity of what's being accomplished.

### Names earn their existence; aliases must buy clarity

Use the domain term, not the structural term. Names should be specific enough
to grep for. Aliases that don't shorten or clarify are Chekhov's gun — future
readers will spend time trying to understand what business purpose they serve.
If you alias, actually shorten:

```python
# bad — doesn't earn the alias
scheduling_preferences = pms_preferences.scheduling_preferences

# good — meaningful shortening
sched_prefs = pms_preferences.scheduling_preferences
```

In internal methods, abbreviations and shortform names are fine to economize
on line width. Verbose names serve public APIs; internal code optimizes for
the reader who is already in context.

Naming corrections from the corpus:
- `non_resolved` → `unresolved`
- `general` → `carestack` (when it specifically handles CareStack)
- `formatting_date` → `hydrated_date`
- `input_list` → `appt_list`
- `org_name` → `office_name`

> "not going to block on this but why are we aliasing these variables?
> `pms_api_from_prefs` is almost as long as `pms_preferences.pms_api`"

Inngest naming scheme — matters for dashboard navigability:
- Events: `namespace/subject.action` (sort by subject — what changed)
- Functions: `namespace/action-subject` (sort by action — what they do)

File and directory names should be short and concern themselves with high-level
organization. Never repeat the directory in the filename:
`/pms/denticon/models.py` not `/pms/denticon/denticon_pms_models.py`. A file
should rarely share the exact name of the single function inside it — that's
a sign the file doesn't represent a meaningful organizational unit.

### Log the interesting case; write log lines as prose

Don't log the base case, skip path, or default — that's noise. Log meaningful
state transitions. Write log lines as natural sentences; the function name and
log level are already in the formatter. Log identifiers and stats, not
serialized objects. Match level to severity — a breaking error is not a
warning:

> "this shouldn't be a warning — it's a breaking error — and therefore we
> should use `get_retell_call_or_raise`"

> "this isn't a warning, it's a critical error, so should be logged as
> `ctx.logger.exception(...)` so that we see the traceback in the logs"

> "don't log on skip. skipping is the base case so this will create a lot of
> log spam. log on accept"

> "prob just log the intake.uid to avoid dumping a lot of json into the logs"

---

## II. Types tell the truth

*"Think about interfaces and abstractions before writing the implementation."*

Types are documentation that the compiler enforces. When a type says `Optional`
but the value is never null, the type is lying — and every downstream reader
pays the tax of believing it. When a function returns `dict[str, Any]`, the
type has abdicated its responsibility. Make types honest, and the code that
uses them gets simpler automatically.

### Nullable fields are a contract, not a default

Every `Optional[T]` field propagates a null-check tax to every callsite. Only
mark a field nullable when `None` is a distinct, meaningful state — not as a
safe default. When absent means empty, `list[T] = Field(default_factory=list)`
is equivalent to `Optional[list[T]] = None` but requires no downstream
null-checking. The payoff compounds: a model with four nullable fields forces
null checks in every function that touches it.

> "why are all of these nullable? when would call_id, intake_id, created_at,
> or updated_at be null while name is non-null?"

> "4 / 7 lines are null checking. we should just not allow config or reasoning
> effort to be null by correctly setting defaults/default_factory"

> "blocks of code that look like this should immediately raise alarms when you
> see them. it's a very clear sign of poor abstractions" *(said of a function
> where 4 out of 7 lines were null checks)*

When a query already filters on `IS NOT NULL` or a primary key, the returned
rows are guaranteed non-null — don't re-check in Python:

> "how is it possible for us to get a null `workflow` from this query? we
> populate `workflow_ids` from the event data and then query specifically for
> those ids. shouldn't we just iterate over the returned workflows?"

> "the query filters on `wsr.node_id IS NOT NULL` and `wsr.workflow_id` seems
> like a primary key so checking `and row['workflow_id'] and row['node_id']`
> doesn't do anything"

When a null check is provably unreachable but the typechecker insists, add
`# pyright: ignore[reportOptionalMemberAccess]` rather than a defensive guard.
The comment communicates "this is intentionally non-null at runtime"; a runtime
guard implies genuine uncertainty.

### Pydantic models are the canonical data boundary

`dict[str, Any]` in a function signature or return type eliminates static
analysis and validation guarantees. If a model exists, use it. If one doesn't,
create it. Type fields as enums rather than strings — Pydantic enforcement on
the field replaces `try/except ValueError` in the body. Use
`Model.model_validate(data)`, not `Model(**data)`. Use modern Pydantic
conventions: `model_config = ConfigDict(populate_by_name=True)`, PEP 695
generics (`class Response[T: BaseModel]`), and `Field(default=False)` instead
of `Optional[bool] = None`.

> "we should be returning validated pydantic types from these methods, not
> `dict[str, Any]` — it seems we already have the models defined, we need to
> use them to validate the API responses before returning them"

> "if this is a BaseModel anyways why are we reimplementing `.model_dump()` and
> `.model_validate()` here and returning it as a `dict[str, Any]`? if we want
> a regular python dictionary from a BaseModel that's exactly what `.model_dump()`
> does and it handles nested models already"

### Enums: compare against the member or the value, never both

Checking `x == SomeEnum.FOO or x == SomeEnum.FOO.value` means the type
boundary upstream is broken — sometimes it's an enum, sometimes a string. Fix
the type at the boundary so only one comparison is ever needed.

> "why are we checking for matches against both the enum and the enum value?
> it should be one or the other, if it can be either there is a bug upstream
> that should be fixed"

> "`action_to_execute.kind` is an enum of some type, probably a StrEnum. why
> do we have to nullcheck the existence of `.value` if the alternative is just
> stringifying it anyways?... this behavior is confusing, seems ill-defined."

### Consistent interfaces within modules

Methods that do similar things should have similar signatures and return types.
If three functions generate prompts, they should all return `str` — not one
returning `str`, another returning a legacy `Prompt()` object, and a third
returning `list[ChatMessage]`. The simplest common type wins.

This extends to parameter names (use the same name for the same concept across
functions), error handling patterns (don't raise in one and return None in
another), and naming schemes within a module.

---

## III. Respect boundaries

*"Dependency direction is non-negotiable."*

Code is organized into modules, layers, and files for a reason. Each boundary
is a contract: this file owns this concern, this layer calls down but never up,
this function validates before it proceeds. When code crosses boundaries — a
CRUD file doing orchestration, an import hiding inside a function body, an
error swallowed at the wrong layer — the system becomes harder to reason about
and easier to break.

### Module files own one concern, co-located with their consumer

CRUD files touch only the table they own. A function that crosses table or
module boundaries belongs in the composing layer above. Prompts live in their
module's `prompts/` subfolder; inngest functions in module-specific files named
for their event. The agent knows nothing about adaptations; the adapter knows
nothing about orchestration.

> "this method does not belong in retell/crud/call.py — this file's single
> responsibility is to provide interfaces to the `retell_calls` database table.
> this function interacts with the legacy calls table, workflows tables, and
> does myriad other things that aren't related to reading and writing from the
> `retell_calls` table"

> "the agent/adapter layering scheme has to work in both directions: the agent
> knows very little about the underlying business logic and mostly only concerns
> itself with orchestration, while the adapter doesn't know anything about the
> upstream orchestration/retell and only concerns itself with handling the
> business logic"

Fix reverse coupling by forking, not importing across the boundary.

### Guards come first, before expensive work

Validity checks and idempotency guards must fire before any database call or
network request. Early return to flatten nesting — the happy path should sit at
the left margin.

> "since this is an idempotency check we should do it first at the top of the
> function, else we're going to try re-importing the number first before
> exiting early"

> "if we're going to null check and early return here why not do it before
> we've made all the db calls to load providers/operatories etc"

> "it looks like most of the blocks in here are conditional on this existing,
> so we should early return if this is None and remove the extra indent blocks
> downstream"

### Exceptions percolate up; never reraise with `raise e`

Don't overwrite errors from upstream — it impedes debugging if we don't know
what the actual error is. Catch at the highest layer where something meaningful
can be done. In the lowest layer (CRUD, HTTP client), let exceptions propagate
— they carry information the caller needs. The specific anti-pattern to flag:

```python
# before — splits the traceback at the catch site
try:
    result = await some_operation()
    return result
except Exception as e:
    raise e  # or: logger.error(f"...: {e}"); raise e
```

> "this is functionally a no-op but actually has the negative side effect of
> breaking the traceback in half. as far as the python runtime is concerned the
> `e` raised here is a brand new exception. to implicitly reraise a caught
> exception you would just want to call `raise` without an argument, but in
> this case the try/except should be removed entirely because it's already not
> doing anything"

If logging and reraising is genuinely needed:
```python
except Exception:
    logger.exception("Failed to ...")  # no variable capture, no message duplication
    raise
```

`logger.exception` auto-captures `exc_info` via reflection — don't put the
exception in the message or it gets double-written. Never catch
`BaseException` — it intercepts `SystemExit`, `KeyboardInterrupt`, and asyncio
cancellation signals.

### Imports live at module scope

Mid-function imports hide missing dependencies from startup checks. If a
dependency is deleted, the server starts fine but crashes at runtime when the
code path is hit — possibly in production, possibly at 2 AM.

> "importing mid-function? this is actually a dangerous pattern — if modules
> are being imported inside functions at runtime, then we could potentially
> delete dependencies and not be able to check if it's safe by simply starting
> the server"

This also applies to using Python private internals for functionality that
should be straightforward:

> "`Language._value2member_map_`? we should not be using python's private
> internal API. furthermore we are needlessly translating back and forth between
> the language code and an enum"

---

## IV. Know your network calls

*"Each database call is a network call. In localdev the DB is running on your
laptop with microsecond latency. In prod it's AWS Aurora with 2-5ms per query."*

The gap between "works on my laptop" and "works in production" is almost always
network calls. Every query, every HTTP request, every inngest step is a
round-trip with real latency, real failure modes, and real cost. Code that
ignores this — fetching everything and filtering in Python, querying in loops,
wrapping reads in inngest steps — will be slow, fragile, and expensive at
scale.

### Filter at the data source, not in Python

If we can do work in SQL, we should do it in SQL, not bring the data back to
process it in Python. Python is slow for CPU-bound work; MySQL is written in
C++ and has been optimized by wizards. Fetching everything and filtering in
Python wastes bandwidth, increases latency, and hides the actual intent.

> "it seems like this is pulling /all/ the providers from carestack across all
> locations and then filtering to the current location afterwards? this seems
> problematic, we should only request the providers for the org location we are
> working with"

> "why not add `AND c.primary_status != 'resolved'` and skip doing that check
> in python?"

> "`get_call_conversation_state` is a very expensive function. if all we need
> to do is get the last protocol id, we should use
> `get_call_conversation_state_str` and parse just the field we need"

The same applies to fetching entire records when only one field is needed:

> "seems like we're pulling the call_record just to extract the org_id, and
> we're only using the org_id to create the task? but we can get the org_id
> from the OrgLocation"

**Related patterns to flag:**
- **N+1 queries**: Querying in a loop when a single batched query would work.
- **Over-fetching columns**: `SELECT *` when only `COUNT(*)` or a single field
  is needed.
- **Redundant data loads**: Reading the same data twice when it hasn't changed
  since the last read. Cache immutable data from network calls.
- **`await` in loops**: Making sequential network calls when `asyncio.gather`
  or `asyncio.TaskGroup` can fan them out concurrently.
- **Unwrapped writes**: Multiple DB writes in the same logical operation that
  aren't in the same transaction — inconsistent state if one write fails.

### Inngest steps are reliability boundaries, not every operation

Each `step.run` costs two HTTP round-trips (yield + resume). Only wrap
operations that must not re-run on retry — mutating network calls. Never wrap
deterministic reads. Never put steps inside loops; use `inngest.parallel` for
fan-out. An inngest function with a single step is worse than none.

> "it's unnecessary to wrap these deterministic db reads in steps. each step
> incurs its own http request. why spend multiple requests when one will do"

> "an inngest function with a single internal step is worse than just calling
> the working function directly because it costs 2 additional http requests:
> one to trigger the step run, and another to resume the function with the step
> result, neither of which adds any reliability benefit"

Avoid writes outside of steps — they re-execute on every function replay:

> "this write is outside a step so it will run again after the invoke returns.
> we write again after the invoke so the pending state will be overwritten
> again but we should try to avoid writes outside steps"

Avoid database calls inside inngest branches — they re-run on every yield:

> "we should try to avoid making database calls inside inngest branches because
> these will get re-run every time this function yields because inngest has to
> restart evaluation from the top of the function"

Don't emit events for work you know you won't do:

> "i dont think we should evaluate and early return inside the function. this
> creates a lot of no-op events and function runs in the inngest queue. we
> shouldn't trigger work that we know we aren't going to do"

Pass identifiers in event payloads; load data on the receiving side from the
DB. Transcripts, full call records, and anything containing PHI must never
appear in event payloads.

### Clients are module-level singletons

Never pass clients through constructors or function parameters. Import the
singleton directly. Constructor injection implies the client varies per
instance, which it doesn't. Long-lived async resources (auth refresh loops,
connection pools) must be managed via `asynccontextmanager` in the FastAPI
lifespan handler, not initialized inside a request handler.

> "we should just import the module-level singleton redis client instead of
> passing it through the constructor" *(said twice on the same PR)*

> "passing a `db` parameter for dependency injection is legacy behavior. import
> the mysql client singleton and use it directly"

---

## V. Understand what you're landing

*"DO NOT LAND CODE YOU DO NOT UNDERSTAND."*

The models are smarter than you, which means they can handle more complexity
than you, and their code reflects that. The models do not know what good code
looks like and will lie to you about it out of total ignorance. If you can't
tell whether code is good or bad, read more code from widely-used projects
that are accepted as having good practices.

### AI-generated code: scrutinize before merging

Patterns to flag:
- Comments describing what the next line does
- Docstrings that describe the wrong behavior ("retries on failure" but raises `NonRetriableError`)
- Spurious union types (`str | UUID`) on inputs that only ever receive one type
- `_get()` / `_post()` HTTP wrapper methods mirroring the underlying client
- Debug `print()` statements
- Unnecessary generics or complex indirection — someone will have to debug this

> "please remove ai slop comments before committing code. the only changes to
> this file should be the addition of 1 line and indenting a block"

> "this union should be removed. claude has a pathological obsession with adding
> these kinds of things for nominal backwards compatibility even when doing so
> would be incorrect as a result of RL reward hacking"

### Tests must assert something real; fixtures must not mutate

A test that asserts `result is not None` on a value that cannot be `None`
provides false confidence and should be deleted.

> "claude loves writing these kinds of looks-real-until-you-actually-look-at-it
> fake tests"

Mutating fixtures forces function-scope caching, regenerating them on every
test call and slowing the suite. Use factory fixtures with module scope and
`@pytest.mark.parametrize` to cover multiple cases without duplication.

**Before** *(from PR #3802)*
```python
@pytest.fixture
def adapter_config():
    config = AdapterConfig(protocol=CallProtocol(name="Test"))
    return config

def test_new_patient_instructions(adapter_config):
    adapter_config.protocol.caller_type = "new_patient"  # mutates fixture
    result = get_how_to_take_call_instructions(adapter_config)
    assert result is not None  # asserts nothing
```

**Comment:** "generally speaking we should avoid mutating fixtures because they
get cached. right now the test suite mutates fixtures in several tests so we
have to set the default fixture scope to 'function', which makes running the
test suite slower since python has to regenerate the fixtures every time a test
requests them."

**After**
```python
@pytest.fixture(scope="module")
def make_instructions():
    def instructions(intent: CallerIntentType | None = None, bookable: bool = True) -> str:
        protocol = CallProtocol(
            name="Bookable" if bookable else "Unbookable",
            caller_intent=[intent],
            can_book_appointment=bookable,
        )
        params = AgentCallInstructionsParams(
            current_protocol=protocol,
            caller_intent=intent,
            appointment_slots=[{"start": "2024-01-16T10:00:00", ...}],
        )
        return get_how_to_take_call_instructions(params)
    return instructions

@pytest.mark.parametrize("intent, bookable, expected", [
    pytest.param(None, True, False, id="intent_none"),
    pytest.param(CallerIntentType.OTHER, True, False, id="intent_other"),
    pytest.param(CallerIntentType.MAKE_A_NEW_APPOINTMENT, True, True, id="intent_new_appointment"),
    pytest.param(CallerIntentType.RESCHEDULE_APPOINTMENT, False, False, id="intent_reschedule_unbookable"),
])
def test_agent_instructions(make_instructions, intent, bookable, expected):
    prompt = make_instructions(intent, bookable)
    assert has_appointment_slot_instructions(prompt) == expected
```

Tests in `prompts/` are live LLM tests making network calls. Pure logic tests
belong in `adaptations/tests/`.

---

## Tone

One sentence naming the problem, then write the fix. Use questions to invite
reconsideration rather than directives: "why do we check both the enum and the
enum value?" rather than "don't check both." Proportional alarm: `?` for mild
curiosity; "definitely don't" for hard rules; full explanation for architectural
decisions. Tag domain owners when a comment touches logic you don't own. For
any substantive comment, include a concrete code suggestion — this is not
optional.
