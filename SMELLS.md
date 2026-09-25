# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplicated pricing policy. The premium surcharge and the long-booking and
evening discounts appeared once in `ReservationManager` and again, under different constant
names, in `ReportGenerator`.

**Classic or agent-specific.** Classic: duplicated code with duplicated domain knowledge,
rather than duplication forced by two different representations.

**Where in the code.** Before the fix: `src/reservationManager.ts`, `calculatePrice` and
`applyDiscounts`; and `src/reportGenerator.ts`, `priceOf`.

**The principle it violates.** DRY / single source of truth for the room-pricing policy.

**What it makes expensive.** Changing a pricing rule, such as adding a weekend discount,
would require edits in both paths. Missing either makes a booking's stored price disagree
with the revenue report.

### Smell 2

**The smell.** A speculative cache abstraction that is never wired into a caching flow.
`listBookingsForRoom` calls `get`, but no production code calls `set` or invalidates a key.

**Classic or agent-specific.** Agent-specific dead scaffolding / speculative generality:
the generated code supplied configuration, expiry, eviction, and an abstraction before the
service had a working cache use case.

**Where in the code.** `src/cache/queryCache.ts`, `QueryCache`; and
`src/reservationManager.ts`, `listBookingsForRoom`, `createBooking`, and `cancelBooking`.

**The principle it violates.** YAGNI and truthful abstractions: an enabled cache should
either affect the query path or not exist.

**What it makes expensive.** A maintainer investigating stale schedules must understand
cache lifetime and invalidation code that cannot currently affect a result. Wiring it later
also requires deciding whether callers may observe cached arrays and precisely when writes
invalidate them.

### Smell 3

**The smell.** Low cohesion in the application service: `ReservationManager` owns room
registration, booking workflow, pricing, receipt rendering, daily-summary rendering, and
notification dispatch.

**Classic or agent-specific.** Classic shotgun-surgery risk from a class with several
unrelated reasons to change. This is not merely a line-count complaint: it combines distinct
policies and output concerns.

**Where in the code.** `src/reservationManager.ts`: `createBooking`, `calculatePrice`,
`formatReceipt`, `formatDailySummary`, and `dispatchNotification`.

**The principle it violates.** Single Responsibility Principle and separation of domain
workflow from presentation and delivery.

**What it makes expensive.** Changing receipt wording or adding a second report format
requires modifying and retesting the booking orchestrator, even though booking validation
and conflict behavior have not changed.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** Smell 1, duplicated pricing policy. It had one pure,
well-bounded responsibility and an existing suite that exercises the surcharge and both
discount rules.

**What changed.** I added `src/pricing.ts` with `calculateBookingPrice(room, start, end)`.
`ReservationManager.calculatePrice` and `ReportGenerator.priceOf` now delegate to it. The
calculation and the public `ReservationManager.calculatePrice` API stay the same; the two
consumers now share one policy implementation.

**What you deliberately did not touch.** Scope line: only the duplicated arithmetic and
its constants moved. I did not change booking storage, report-window semantics, cache
behavior, dependency injection, notification design, or tests. Those are separately useful
design questions and would make this behavior-preserving refactor harder to review.

**How you know behavior is preserved.** `npm test` passes all 39 tests and `npm run
typecheck` passes. The tests cover plain, premium, long, and evening prices, and revenue
totals that must agree with created bookings. They do not independently enumerate every
combination of discounts or prove a future pricing-rule edit uses the shared function.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Smell 2's inert cache / speculative abstraction.

**The decomposition.** If profiling establishes a repeated-read problem, replace the
manager-owned `QueryCache` with a `CachingBookingRepository` decorator around
`StorageProvider`. The decorator owns cache keys, copies, expiry, and invalidation; writes
invalidate the room key there, while `ReservationManager` only asks its storage boundary for
bookings. Until that need exists, the simpler alternative is to remove the cache entirely.

**One cost.** A decorator adds construction and invalidation complexity, and stale-read
semantics must be documented and tested. Removing it instead costs a future reintroduction
if performance measurements later show it is necessary.

### Proposal B (not coded)

**The problem.** Smell 3's mixed booking, presentation, and delivery responsibilities.

**The decomposition.** Keep `ReservationManager` as the booking application service that
coordinates validation, conflicts, pricing, and storage. Move receipt and daily-summary
formatting to a `ReservationFormatter`; have a notification service receive a formatter and
channel, then own dispatch and the notification log. Booking rules stay in the manager and
the shared pricing function; formatting rules live only in the formatter.

**One cost.** More collaborators and constructor dependencies make this small in-memory
example less direct, and callers must decide where formatter/channel configuration belongs.

### The thing that looks smelly but is fine

**What it is.** `src/validation.ts`, `validateReservationRequest`, which has many sequential
guard clauses and error strings.

**Why it is fine.** It is a single, ordered policy: validating one reservation request and
returning the first actionable reason. The checks share the same inputs, have no side
effects, and the parameterized tests exercise the order and messages. Splitting each guard
into a separate class would obscure the order that determines the public error.

**What would flip your verdict.** It becomes a real problem if different buildings, room
tiers, or customer plans need independently selectable validation rule sets. Then the rules
would need named policy objects or composable validators rather than one fixed sequence.
