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

**Which smell you attacked.** 

I attack the pricing functionality - 'Duplication over reuse' smell. There are three reasons:
- **It sits on the most likely path of change.** Changing prices and discounts is the most common business change in this system. As I showed in Milestone 1, this kind of change can already go wrong today without any warning, and the tests do not catch it.
- **The fix can be small and keep behavior exactly the same.** I compared the two pricing versions, and they calculate in the same steps: compute the base price and round, then apply premium, long-booking and evening multipliers in that order, rounding after each one. So merging them into one does not change the result for any input. The behavior is truly preserved.
- **The other two smells cannot be fixed with a small change.**
  - Smell 2 (God class): it needs to be split into several classes, which touches a lot of code.
  - Smell 3 (factory): registerChannel, createNotificationChannel and the rest are exported public API. Removing them, or changing the ReservationManager constructor signature, would change behavior for outside callers.

**What changed.** Files and methods you touched, and what the code does differently now.

I merged the two copies of the pricing rule into one new module, and both old callers now use it.

- New file src/pricing.ts: Holds one set of pricing constants; Adds priceFor(room, start, end), which uses the same steps as before;
- src/reservationManager.ts: Removed the five pricing constants and the private method applyDiscounts. Kept the public method calculatePrice(room, start, end) with the same signature. Its body is now one line: return priceFor(room, start, end).
- src/reportGenerator.ts: Removed the five renamed duplicate constants and the private methods priceOf and durationOf.
revenue() used to call this.priceOf(room, booking). It now calls priceFor(room, booking.start, booking.end).

The **difference code does** is that the pricing rule is written only once, in pricing.ts. Changing any rule (for example, premium from 15% to 20%) now takes one edit, and the stored booking price and the revenue report always stay in sync. It is no longer possible to update one side and forget the other. Also, ReportGenerator no longer keeps its own pricing logic. It depends on the small pricing module, not on ReservationManager.

**What you deliberately did not touch.** Merge the two copies of one rule, "how much a booking costs", into one. Do not change who calls it, when it is called, or any public signature. Anything that changes behavior, belongs to a different rule, or prepares for a future need is out of scope.

- **No PricingPolicy interface or Strategy pattern.**
There is only one pricing rule set and no request for a second one. Adding an interface would create the same speculative over-abstraction as Smell 3. A plain function is enough to remove the duplication.
- **The God class's other jobs (formatting, notifications, room registry) stay where they are.**
This fix does move pricing out of ReservationManager, but only because removing the duplication needed a shared home. Moving out formatReceipt, formatClock, formatMoney or notifications would be splitting the God class. That is a different fix, aimed at different changes (display format, notification channel).
- **revenue() still recomputes the price instead of reading booking.priceCents.**
Reading the stored price looks cleaner, but it changes behavior: if a room's rate is changed after a booking was made, the two versions report different revenue for past bookings. Whether the report should show "what was actually charged" or "what it would cost at today's rate" is a business decision, not a refactoring. It is outside "behavior preserved", so I left it.

**How you know behavior is preserved.** 

After the change, all 39 tests in npm test pass, npm run typecheck is clean, and no file under tests/ was edited. The change also only moves code. So every input should give the same result as before.

**What the suite actually covers:**
- The pricing group in booking.test.ts checks each rule on its own, through createBooking:
  - base price: 12000
 - long-booking discount: 16200, using exactly 180 minutes, so it hits the edge
 - premium surcharge: 18400
 - evening discount: 11400, starting exactly at 17:00, also on the edge
- The revenue tests in reporting.test.ts check that the report total equals the sum of booking.priceCents, and compare against the fixed values 23400, 11700 and 11400. This shows that after the merge, the manager and the report still agree on the base price and the evening discount.
- The formatDailySummary test indirectly checks stored prices (Confirmed total: $234.00).
- The typecheck confirms that the public calculatePrice signature did not change and every call site still fits.

**What it would not catch:**
- **Rules combined.** No test triggers two or three of premium, long-booking and evening at once. If the merge had changed the order of multiplying or rounding, some combinations would be off by one cent and the suite would still pass.
- **Premium and long-booking rules on the report side.** The revenue tests only use the non-premium room r1 and only two-hour bookings, so those two rules are never tested through the report.
- **The other side of each edge.** The tests only check "just reaches the limit" (180 minutes, 17:00), not "just misses it" (for example 165 minutes, or a 16:45 start). If >= became >, only the first case would show it.
- **Calling calculatePrice directly.** It is public, but no test calls it directly. It is only used through createBooking.

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
