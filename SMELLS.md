# reservation-service: Smells and One Fix

Fill in each section. One section per milestone. Keep it short and specific. Point at files
and methods, not adjectives.

---

## Milestone 1: Three smells

Three smells, each in a different part of the module. For each one, fill in all five parts.

### Smell 1

**The smell.** Duplication over reuse

**Classic or agent-specific.** Agent-specific, caused by missing context. This is because it is not copy-paste duplication. The rule was rebuilt from scratch, not copied. All five constants have the same values but different names and the structure differs too. It looks like the existing calculation was not in the agent's context, so the agent rebuilt it instead of reusing it.

**Where in the code.** ReportGenerator.priceOf re-derives the same pricing rule instead of reusing ReservationManager.calculatePrice.

**The principle it violates.** Single source of truth (DRY): one business rule should have one authoritative home.

**What it makes expensive.** Any change to a pricing rule now needs the same edit in two files, and forgetting one does not always fail a test.

- Concrete case: raise the premium surcharge from 15% to 20%. A developer or agent searches for "premium" and edits PREMIUM_MULTIPLIER in reservationManager.ts, then updates the booking pricing test. New premium bookings now store the higher priceCents, but ReportGenerator.revenue() still recomputes them at 1.15.
- What breaks first: the revenue report silently disagrees with what customers were actually charged. The suite stays green, because the revenue tests only use the non-premium room r1 and only two-hour bookings.

### Smell 2

**The smell.** God class: ReservationManager owns five unrelated responsibilities, containing 'Room registry', 'Booking lifecycle and conflict detection', 'Pricing', 'Notification', Text format'.

**Classic or agent-specific.** Classic, because it breaks a classic principle-cohesion.

**Where in the code.** src/reservationManager.ts, class ReservationManager

**The principle it violates.** High cohesion / single responsibility: a class should have one reason to change and one audience. This one answers to the front desk, finance, notifications and printing at once.

**What it makes expensive.** Unrelated changes are tangled together, and reusable pieces are locked inside the class.

- Concrete case: Switching the display to a 12-hour clock means editing formatClock. But createBooking also uses formatClock in its conflict error (line 74). So a presentation-only change also changes the text of a business exception, which affects any caller that reads it.
- What breaks first is reuse. Pricing, formatClock and formatMoney are reachable only through this class (the last two are private). Any other module must either depend on the whole manager or rewrite them.

### Smell 3

**The smell.** Speculative over-abstraction. Notifications are built as a pluggable-channel system: a global registry, a factory and a config object. Yet there is exactly one channel (email), and the one client, ReservationManager, cannot use the flexibility this layer promises.

**Classic or agent-specific.** Agent-specific, caused mainly by an underspecified request, with free volume on top. Compared with the classic smell - speculative generality which means someone added an extension point for a change they expected, and the change never came, the smell here shows nothing suggests a second channel was ever wanted. There is no second implementation, no removed one, and ChannelName has only ever been 'email'. The whole mechanism was written in one go for a agent request that only said "notify the organizer after a booking."

**Where in the code.** src/notifications/notifierFactory.ts; The only consumer: src/reservationManager.ts:40

**The principle it violates.** Decouple what varies: pay for decoupling only where change actually happens. Decoupling a change that never comes is complexity you pay for anyway.

**What it makes expensive.** The layer charges for flexibility now, and still charges again when that flexibility is needed.
- Concrete case: Adding a second channel (the change it was built for): the extension point is closed. registerChannel only accepts ChannelName, which is the union 'email', and NotifierConfig only has an email fromAddress. So adding SMS means editing notifierFactory.ts itself, plus the ReservationManager constructor, which hardcodes the default config.

---

## Milestone 2: One small fix

One fix, behavior preserved, suite green, zero test edits.

**Which smell you attacked.** And why that one.

**What changed.** Files and methods you touched, and what the code does differently now.

**What you deliberately did not touch.** Name the scope line you drew and why you drew it
there. "I ran out of time" is not a scope line.

**How you know behavior is preserved.** Point at the suite, say what it actually covers, and
say what it would not catch.

---

## Milestone 3: Two proposals and one false positive

One proposal for each milestone 1 smell you did not fix.

### Proposal A (not coded)

**The problem.** Name it.

**The decomposition.** What are the pieces, what does each own, and where do the rules live?

**One cost.** Something this actually costs. "No real downside" is not a cost.

### Proposal B (not coded)

**The problem.**

**The decomposition.**

**One cost.**

### The thing that looks smelly but is fine

**What it is.** File and method.

**Why it is fine.** Defend it with properties of the code, not with its line count.

**What would flip your verdict.** Name the change that would turn this into a real problem.
