# Method

**v0.2, draft, 2026-10-07.**

Two parts. First, a taxonomy to place any AI testing tool, this one included. Second, the three-piece architecture this method uses, and the rule that holds it together: no component validates its own output, enforced at three boundaries.

## Part 1: three generations of AI in testing

Marketing blurs them. They are different in one specific way: **who decides the next step.**

| generation | roughly | what the AI does | who decides the next step |
|---|---|---|---|
| 1. AI-assisted | 2022 to 2024 | self-healing selectors, suggested test cases | a human designs and triggers each step |
| 2. AI-generated | 2024 to 2025 | writes full scripts from requirements or from the UI | a human builds and maintains the pipeline they run in |
| 3. Agentic | 2025 onwards | plans, executes, reads results, chooses what to do next | the agent, inside a loop; the human sets the goal |

The line between generation 2 and 3 is not "more AI". It is that the AI moves from **suggesting** to **deciding and acting** inside a loop.

### What the taxonomy leaves out

The sources at the end describe **planning** and **execution** in detail and say little about **judgment**: who decides that a result is correct, against what, and with which evidence.

When the agent that planned and ran the test also declares it passed, the system grades itself. That is the gap this method is built around.

### Where this method sits

By the table above, this method is not pure generation 3. That is on purpose.

- **Planning is agentic.** The agent explores the change, reads the code and the existing suite, decides what is worth testing and proposes it.
- **The contract is human.** A person approves each new or changed scenario, and then the generated test that implements it, before it runs.
- **Execution is fixed, not agentic.** Playwright runs the approved, frozen test without adapting its assertions or its logic.
- **The verdict is separate.** Neither the agent nor the person who approved grades the run. The judgment layer does.

The reason is the rule that runs through the whole method: no component validates its own output. An agent that chooses what to test and also declares the result has no outside check. Reading a handful of scenarios, and then the tests that implement them, is cheap. It is where a person fixes intent before anything runs.

The rule is enforced at three boundaries:

- **Intent independence.** A human fixes what the test is supposed to prove.
- **Execution integrity.** Approved intent cannot change silently during translation to code or during execution.
- **Judgment separation.** The component that proposes or executes the test does not issue the semantic verdict.

The planning agent reaches the repository through plain read tools: the diff, the files, the existing tests. No MCP (Model Context Protocol) server is built for it. If the agent SDK in use already speaks MCP, that is used as is.

Approval is per scenario, not per run. Once approved, a scenario and its frozen test join the suite and run on every change in CI without asking again.

## Part 2: the three-piece architecture

A thin harness that orchestrates existing tools. Three pieces. Code generation sits between the first two.

```
change or requirement
        |
        v
[1] PLANNING AGENT  ---> Gherkin scenarios
        |
        v
   human approval  (the contract: intent)
        |
        v
   code generation  ---> Playwright test + coverage manifest
        |
        v
   human approval, then freeze  (implementation matches the contract)
        |
        v
[2] PLAYWRIGHT EXECUTION  ---> results + evidence
        |
        v
[3] JUDGMENT LAYER  ---> verdict per criterion, with evidence
        |
        v
   report
```

### Piece 1: planning agent

**Job:** turn a change or a requirement into a short list of scenarios worth testing.

**Input:** a diff, a ticket or a written requirement.

**Output:** scenarios in Gherkin (Given, When, Then), each one tagged with the risk it covers.

**Hand-off:** a human approves, edits or rejects each new or changed scenario. Nothing new runs without that step.

Gherkin is used here as a **review contract**, not as a writing layer: it is the most readable format for a person who has to say "yes, this is what we meant". Reasons in [DECISIONS.md](DECISIONS.md), D2 and D3.

### From contract to code

An LLM turns an approved scenario into a Playwright test in TypeScript. That translation is where approved intent can get lost, so it has its own guard, in three parts.

- **A mapping per clause.** Each Given, When and Then lists what implements it: a fixture, an action, one or more assertions. The relation is many to many, so the mapping is per clause, never line by line.
- **A coverage manifest, checked mechanically.** The harness refuses a generated test when a Then has no assertion, a clause has no implementation, or code appears with no clause behind it. The manifest is written by the same model that writes the code, so this check proves structural coverage: every approved clause has a declared implementation and every Then has a declared assertion. It does not prove the mapping is right: that is what the second approval is for.
- **A second approval, then a freeze.** The person approves the generated test with its manifest. The approved test is stored with its hash, the scenario version, the approval date and the generating model. Any regeneration changes the hash and needs a new approval.

Scenarios are approved first and translated after, so no code is generated for a scenario that gets rejected.

### Piece 2: Playwright execution

**Job:** run the approved tests and collect evidence.

**Input:** approved, frozen tests.

**Output:** pass or fail per assertion, plus evidence: every assertion result, the relevant trace steps, screenshots at visual assertions and at meaningful state transitions, and a network summary when it matters. The judge receives the minimum evidence that lets it decide, not a dump of the run.

**No self-healing in v1.** A selector that breaks is a failing test. Bounded self-healing is a future extension, with its limit already fixed: it may repair how an element is found, never what is asserted about it, and every repair is logged with the old and the new selector.

### Piece 3: judgment layer

**Job:** evaluate whether the observed outcome satisfies each applicable criterion, independently of the planning agent.

**Input:** the approved scenario, the execution evidence, a versioned set of criteria and a versioned judge configuration.

**Output:** a verdict per criterion, each one linked to its evidence.

**Implementation:** a separate LLM call that receives no planner reasoning. It sees the approved scenario, the execution evidence and the active judge configuration, and it reads Playwright's assertion results as evidence, not as the verdict. This separates responsibilities and context. It does not make the judge unbiased: the judge is a model too, and it can be wrong in its own way. The method does not claim an impartial judge. It claims a judge the planner cannot steer, whose errors are measured (rules 1 and 8 below).

The two answer different questions:

- **Playwright:** did each step do what the code checked?
- **Judge:** does the evidence show that the intent was met, criterion by criterion?

When they disagree, the report shows both. Those are the cases worth reviewing.

This is where the method differs from a runner with an agent on top. The rules below come from auditing agentic systems in the [chatbot observatory](https://github.com/missyeahhh/chatbot-observatory):

1. **Criteria and judge are versioned.** A verdict names the criteria version and the judge configuration it used: model and snapshot, judge prompt, evidence schema, output schema, temperature. Changing any of them is a new version, never a silent edit. Temperature zero reduces variation; it does not remove it, and the method never claims it does.
2. **Every verdict carries its evidence.** No evidence, no verdict.
3. **Scale per criterion: 2 passes, 1 partial, 0 fails, `n/a` does not apply, `NE` not enough evidence.** `n/a` means the criterion does not apply to the scenario. `NE` means it applies but the evidence does not allow a decision. Neither counts as 0. Both leave the denominator and both stay visible next to the score: a result that hides what it could not judge looks better than it is. A judge that punishes what it could not measure reports a failure it never observed.
4. **An execution pass is not a verdict.** Playwright assertion results are evidence for the judge; they do not determine its criterion-level verdicts.
5. **Check one thing that was not the goal.** A sentinel runs before and after the whole suite: one known function outside the change, such as a known account signing in. A scenario adds a neighbor of its own only when the planner tags the risk as worth it. A test can meet its goal and break something next to it.
6. **An execution failure is confirmed by one rerun** of the same approved scenario and the same frozen test. E2e runs are flaky: one run cannot tell a failure from a variation. Passes are not re-run. If the rerun passes, the execution is reported as flaky, not as failed. The judge's own stability is not measured by reruns but by calibration (rule 8).
7. **A run with zero verdicts is a failed run, not an empty one.**
8. **The judge is calibrated against human labels.** Fifteen to thirty hand-labelled cases, chosen to be hard: an assertion that passes while the outcome is wrong, an ambiguous screenshot, contradictory evidence, a criterion that cannot be observed. Agreement between judge and human is published per criterion with each criteria version.

## A worked example

**Illustrative.** No harness has run this yet: it shows the shape of each hand-off.

**The change:** the login form now locks an account after five failed attempts.

### 1. The agent proposes

```gherkin
# risk: the lockout never triggers
Scenario: account locks after five failed attempts
  Given a registered user
  When they enter a wrong password five times
  Then the account is locked

# risk: the counter never resets
Scenario: a successful login resets the counter
  Given a registered user with four failed attempts
  When they log in with the right password
  Then the failed attempt counter is back to zero

# risk: the lockout reveals which emails have an account
Scenario: an unknown email gets the same message
  Given an email with no account
  When someone enters a wrong password five times
  Then they see the same message a registered user sees
```

### 2. The human approves

| scenario | decision | note |
|---|---|---|
| account locks | approved | |
| counter resets | rejected | already covered at a lower level; no extra e2e scenario justified |
| unknown email | approved | |

The two approved scenarios are then translated into tests. The person approves each generated test with its manifest, and the tests are frozen.

### 3. The criteria, v1.0

Versioned for the whole suite, not written per scenario:

- **C1. Outcome:** the end state in `Then` is visible in the evidence.
- **C2. Next step:** when the outcome requires user action, the evidence shows what happened and what the user can do next.
- **C3. Suite sentinel** (rule 5): a known valid account logs in before and after the suite. Reported once at run level; it does not count in scenario scores.

### 4. The verdict, scenario "account locks"

Playwright: all assertions passed.

| criterion | score | evidence |
|---|---|---|
| C1 outcome | 2 | screenshot after attempt five shows "Account locked"; trace step 14 |
| C2 next step | 1 | the message says the account is locked, with no way to unlock it |

**Scenario result: 3 out of 4 applicable, criteria v1.0. Run sentinel: pass.**

The code checked that a lock message appears, and it did. The judge scored C2 as partial, because the message leaves the user stuck. That gap between "the assertion passed" and "the intent was met" is what this layer exists to catch.

For the scenario "unknown email", the screenshot after the fifth attempt was taken before the message rendered, so C2 cannot be judged from the available evidence and is `NE`. The result reads "2 out of 2 applicable, 1 NE": never "2 out of 4", and never a bare "2 out of 2" that hides the gap. `NE` does not trigger a rerun: the evidence capture is corrected before a later run.

## Failure modes this method is designed against

Taken from the observatory's [taxonomy of failures in an agentic system](https://github.com/missyeahhh/chatbot-observatory/blob/main/metodo/taxonomia-de-fallas.md):

| failure | how it shows up in testing | what guards against it |
|---|---|---|
| stating state without verifying it | the agent reports "all green" from its own model of the run | rule 4: an execution pass is evidence, not a verdict |
| verifying the goal is not verifying the system | the test passes and a neighboring flow breaks | rule 5: a sentinel before and after the suite, plus a neighbor where the risk justifies it |
| the workaround that switches off the detector | a regenerated test quietly drops or weakens an assertion | the coverage manifest and the frozen hash: nothing approved goes missing or changes without a new approval |
| execution silence | a run shows as executed and delivered nothing | rule 7: zero verdicts is a failed run |

## Proof of concept

The harness will be built against [Ghost](https://github.com/TryGhost/Ghost): a real, actively maintained application with authentication, server-side state, a Playwright end-to-end suite, fixtures and a data factory. The generated tests will live in a fork of Ghost, next to its own suite, so they reuse its fixtures and page objects. The harness itself (planner, judge, manifests) will live in this repository and point at the fork.

It will be demonstrated on two or three real Ghost pull requests, one per kind: a straightforward change, where the planner finds the obvious scenario; a change with lateral risk, where the diff touches something the existing suite does not cover; and a judgment case, where every assertion passes and the judge still scores a criterion as partial.

## Sources for Part 1

- [TestQuality: Agentic QA Architecture](https://testquality.com/agentic-qa-architecture-autonomous-testing-2026/)
- [Autify: How Agentic AI Improves QA and Testing in 2026](https://autify.com/blog/ai-agent-testing)
- [Katalon: What Is Agentic QA?](https://katalon.com/resources-center/blog/what-is-agentic-qa-the-complete-guide-for-2026)
- [Model Context Protocol, official documentation](https://modelcontextprotocol.io)
