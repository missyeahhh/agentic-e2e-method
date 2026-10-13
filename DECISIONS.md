# Decisions

Closed decisions behind the method. Each one says what was decided and why. Reopening one means writing a new entry, not editing the old one.

## D1. Playwright, not Cypress

**Decided 2026-10-03.**

**Why:** it is what the market asks for. Of 78 QA roles I reviewed in 2026, my notes record Playwright in 8 postings and Cypress in 3.

| tool | roles that mention it, out of 78 |
|---|---|
| Playwright | 8 |
| Selenium | 4 |
| Cypress | 3 |
| BDD | 2 |
| Gherkin | 1 |
| Cucumber | 0 |

Counted on 2026-10-04. My notes summarize each posting rather than reproduce it, so these counts are conservative. They show a direction, not a market share.

## D2. Gherkin is NOT the writing layer

**Decided 2026-10-03.**

**Why:** writing tests in Given, When, Then adds a translation layer between the intent and the code, and that layer is rarely kept in sync. In the same sample, 1 role out of 78 mentions Gherkin.

## D3. Gherkin IS the agent's output

**Decided 2026-10-03.**

**Why:** the planning agent needs to show a human what it intends to test, in a format that person can approve in one read. Gherkin does that well.

So Gherkin stays, with a different job: **a review contract, not a syntax.** The agent proposes scenarios in Gherkin, a human approves them, and only then are they translated into Playwright code. Nobody maintains the Gherkin by hand.

## D4. Ghost as the reference application

**Decided 2026-10-07.**

**Why:** the method needs one real application, and one that was not built to suit the harness. Of the candidates (Ghost, Cal.com, RealWorld, an application of my own), Ghost provides what the three pieces need without any purpose-built behavior: authentication, server-side state, an active repository with real pull requests to use as diffs, an existing Playwright end-to-end suite the planner can read, fixtures and a data factory. Cal.com offers similar infrastructure but adds scheduling, time zones and availability, complexity unrelated to the method. RealWorld does not offer that combination in one maintained implementation. An application of my own would weaken the demonstration, because both sides of the test boundary would be under my control.

## D5. The generated test is approved and frozen, not trusted

**Decided 2026-10-07.**

**Why:** an LLM turns the approved scenario into Playwright code, and that translation is where approved intent can get lost. Three options: a deterministic step library (Cucumber by another name, and inconsistent with D2), a second model checking the generated code, or a human approving the generated test. A second model can flag possible mismatches, but it cannot replace the external approval that fixes intent, so human approval remains the gate. A coverage manifest checked mechanically and a hash that freezes the approved test make omissions and post-approval changes detectable. See METHOD, "From contract to code".

## D6. No self-healing in v1

**Decided 2026-10-07.**

**Why:** bounded self-healing is well defined but supports none of the method's claims. Implementing it correctly is a separate problem: detecting selector drift, establishing that the new element is the same one, avoiding false repairs. The limit stays written (repair how an element is found, never what is asserted); the implementation is a future extension.

## D7. No MCP server of its own

**Decided 2026-10-07.**

**Why:** the planner needs to read a diff, files and existing tests. Plain read tools do that. Building an MCP server would take hours from the judge and its calibration and add nothing the method needs.
