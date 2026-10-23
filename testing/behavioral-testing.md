# Behavioral Testing in a Simulated World

This note specifies how to build and write behavioral tests for web and native frontends that depend on external systems. Each test runs the real application against a controlled fake of the outside world, acts through the user interface, and checks both what the user sees and what the world recorded.

The rules below are normative. For the reasoning behind them, see [Design Principles for Behavioral Testing in a Simulated World](https://kardaj.com/posts/2026-10-11-testing-principles--behavioral-testing-in-a-simulated-world/).

Out of scope: unit tests of localized computation (parsers, calculators, migrations) and verification of provider internals.

## Model

### Glossary

- **Outside world:** anything the application doesn't fully control: backend APIs, third-party providers, platform services, time.
- **World:** the fake outside world for one scenario. It holds the authoritative external facts: users, permissions, domain entities, remote services and scheduled external events.
- **Fake:** the implementation of one external system inside the world. It computes answers from world state and implements the surface the application consumes.
- **Boundary:** the point where application code ends and an external system begins. The fake intercepts calls there.
- **Device:** a simulated phone or browser. It holds local storage, permissions and connectivity, and hosts one application instance.
- **Application instance:** the running app on a device, with its own session, state, cache and navigation.
- **Scenario:** one test, written as a product story: facts, user actions, expected outcomes.
- **Seed:** a `world.seed…` call that declares one domain fact. The builder supplies consistent defaults for everything else.
- **Handle:** the value a seed returns. It is the only way a scenario references generated entities.
- **Probe:** a read-only view of the world or of a device's platform services, used to assert what the world recorded.
- **Runtime:** where a scenario executes: in-process (a JS test runner rendering the app in a simulated environment) or UI (a real browser or a native simulator).
- **Driver:** the per-runtime implementation of `env.app`, the shared interface for acting on and observing the application.
- **Scoped condition:** a descriptive control that runs an interaction under an external condition, such as a pending request or an unavailable network.
- **Request handle:** a lower-level control over one in-flight request, used when the ordering itself is the behavior under test.

### Example

The running example is a bikeshare app. A rider ends a trip, sees a loading indicator while the request is in flight, then sees the confirmation, and the world records the trip as ended:

```ts
describe("ending a trip", () => {
  let env: TestEnv;

  beforeEach(async () => {
    env = await createEnv();
  });

  it("shows a loading indicator while ending, then a confirmation", async () => {
    const bike = env.world.seedBike({ isOnTrip: true });

    await env.app.launch();
    await env.app.logIn(bike.rider);
    await env.app.openCurrentTrip();

    await env.world.withRequestPending("endTrip", async () => {
      await env.app.tapEndTrip();
      await env.app.expectEndTripLoading();
    });

    await env.app.expectEndTripConfirmation();
    expect(env.world.probeTrip(bike.trip).status).toBe("ended");
  });
});
```

Every scenario follows this shape: seed the facts, launch, reach the starting state through the product, act under explicit external conditions, then assert the screen and the world separately.

## 1. Give each scenario a fresh world, and each device its own app instance

**Why:** a test that shares state with its neighbors fails or passes depending on what ran before it. Giving each scenario its own world and each device its own app removes that dependency.

A world holds the authoritative external facts: users, permissions, domain entities, remote services and scheduled external events. A device hosts one application instance and holds its own local storage, permissions and connectivity; the application instance holds its own session, state, cache and navigation.

**In a scenario**

- Use one device unless the behavior needs more. Create additional devices through the same API; several devices may belong to the same user without sharing local state.
- Seed one fresh world per scenario. Never depend on another scenario or on execution order.

**The harness guarantees**

- Relaunching a device keeps its persistent storage and rebuilds its application runtime. Resetting a device and resetting the world are separate operations.
- The world outlives the application process, so a relaunch, including a real process restart in the UI runtime, sees the same world.
- Scenario-owned listeners, scheduled work, storage and interception are disposed of when the scenario ends.

## 2. Fake at the lowest boundary you can intercept

**Why:** every layer you replace is a layer you stop testing. Faking at the transport boundary keeps URL construction, headers, serialization, retries and cancellation in the loop; faking at a high-level adapter silently removes them.

The boundary is where the application's code ends and an external system begins; the fake intercepts calls there. Keep the production path intact down to the lowest boundary the test can own, and intercept there. For an HTTP API that is the network. For a platform framework such as CloudKit it is the framework's client boundary (the container and database interface), because the network beneath it cannot practically be intercepted. For sending an SMS it is the messaging provider's API or the platform's message composer. For notifications it is the platform's notification center: registration and scheduled local notifications go out through it, and a push arrives as an event the world delivers to the device. When a user interaction is under test, the real screen, the real logic, the real cache and the real API client all run; only the outgoing request is caught and turned into an operation on the fake world.

**In a scenario**

- Read a passing test against the boundary the app is faked at: everything above it is exercised, everything below it is not.

**The harness guarantees**

- Production URL construction, headers, request bodies, response decoding, error handling, retries and cancellation all run.
- Other external effects are replaced at explicit boundaries: clocks, identity providers, clipboard, location, camera, purchases, outgoing messages (SMS, email, push), notifications and platform services as applicable.
- Implementations are selected at composition or launch. Production business logic never branches on whether a test is running.
- Production persistence stays in place where practical, on isolated temporary stores. An in-memory substitute is used only when the behavior under test does not depend on semantics it omits.
- A framework that cannot be intercepted at the network is substituted at its application-facing boundary.

**Maintaining the fake**

- Record which boundary each external system is faked at and what that substitution excludes. Replacing a high-level adapter can give useful coverage, but it omits the production transport behavior below it; that coverage limit is written down, not assumed.

## 3. Make fakes compute answers from state, not canned responses

**Why:** a fake that returns whatever the test expects can never disagree with the test, so it can never catch a bug. A fake that computes its answer from state will refuse an impossible operation, and that refusal is what finds the defect.

A fake operation computes its result from the current world state, the request input, the caller's identity and any explicitly configured conditions. Implement the external surface the application consumes, with enough semantics to exercise its relevant behaviors and failure cases. For example, in a bikeshare app, ending a trip succeeds only if the bike is on a trip and the caller is its rider; afterward, every service that reports on bikes reflects the ended trip. The fake implements both the business rule and the relationship between bike and rider that the rule depends on.

**In a scenario**

- Expect failures to follow from the facts you seeded: a different rider, a missing grant, a bike that is not on a trip. Transport faults come from the timing and fault controls in [P7](#7-control-when-external-events-happen-never-sleep).

**The harness guarantees**

- Successful mutations change authoritative state. Later reads through every relevant service reflect them according to the modeled consistency rules.
- Each authoritative fact is stored once and endpoint projections are derived from it. Related services may remain separate implementations, coordinated through explicit ownership and relationships.
- Ordinary failures derive from domain facts: invalid identity, insufficient access, conflicting entities, invalid input or forbidden state transitions. An impossible operation never succeeds because a test expects it.
- Relationships and transitions are validated at both seed and runtime mutation boundaries.
- Authorization is expressed as testable predicates over the authenticated identity, requested tenant or system, resource ownership, grants and applicable denies, preserving the provider's role semantics, deny precedence, error shapes and validation order.
- Consumed contract details such as pagination, limits, filtering, ordering and response envelopes are preserved where they affect application behavior.

**Maintaining the fake**

- The fake models externally observable contracts. It need not reproduce the provider's architecture or implement unused endpoints; semantics are added as the application comes to depend on them.

## 4. Keep the world's truth apart from each device's view

**Why:** if the fake server and the app's cache share the same objects, a broken cache invalidation can never fail a test, because both sides always agree. Keeping them apart is what lets a scenario show a device holding a stale or optimistic view while the world holds the truth.

The fake's data must be independent of each device's cache and local storage. The rider's phone may still show the trip as active after the world has ended it, or show it as ended before the world has accepted the request. The environment must be able to represent both, and tests must be able to inspect either side on its own.

**In a scenario**

- Assert "the world accepted the change" and "this device observed the change" separately. Each can be true without the other.
- To test a stale or optimistic view, change the world behind a device, or hold the device's write, and observe what the device shows.

**The harness guarantees**

- A device learns remote facts only through the production read or subscription path.
- A client mutation reaches the world only through the boundary that production uses.
- Values are independent across simulated serialization boundaries. Mutating a returned object on the client never changes the world.
- The world can change while a device keeps an older view, and a device can show an optimistic change while the world still holds the previous value.

## 5. Seed only the facts that matter; let defaults fill in the rest

**Why:** a scenario should read as a list of facts that matter, not as the plumbing required to make them exist. One `world.seed…` call per fact keeps setup short, and the builder, not the test author, is responsible for a consistent world.

Each `world.seed…` call applies defaults, resolves relationships, validates and registers the entity, and returns its handle. Probes are the counterpart: they read back the world or a device's platform services without changing it.

```ts
const bike = world.seedBike({ isOnTrip: true });
```

This declares that a bike is on a trip. The builder creates only what that fact requires (a bikeshare system, a rider, an ongoing trip record), and every service that reports on the bike reflects the same trip.

**In a scenario**

- Declare facts with `world.seed…` in domain terms. Never build response payloads or plumb identifiers in a test.
- Override only what the scenario is about and rely on the documented defaults for the rest. Never depend on an incidental entity a default happened to create.
- Stage setup through the runner's ordinary lifecycle: `beforeEach` creates a fresh environment and seeds common entities; nested setup and individual cases add their own facts before launch. Share the recipe, never a world created once for the suite, and keep returned handles so later setup reuses earlier entities explicitly.

**The harness guarantees**

- Omitted supporting facts get minimal, deterministic defaults. Time and generated identifiers are controlled; variation, where useful, comes from a reproducible seed. A named shared base may supply required context.
- Explicit overrides are honored. A default never conceals a condition the scenario asked for.
- Explicit references are resolved; missing or contradictory ones fail clearly. Omitted dependencies are generated, explicitly invalid references are never repaired.
- Generated supporting entities are inspectable through returned handles or probes.
- Singular `seed…` operations are additive: existing entities are preserved, a fresh identity is created when none is given, and duplicate explicit identities are rejected rather than updated. A named update or replacement operation has a different contract, and each collection operation declares whether it replaces, appends or merges.
- Response payload construction lives in the fake's transport projection layer, never in a seed.

## 6. Write each scenario once, run it in every runtime

**Why:** the same story (log in, end the trip, see the confirmation) should be told once and run anywhere. If each runtime has its own copy of the scenario, the copies drift and the UI suite ends up testing a different product than the in-process suite.

One interface, exposed under a name natural to scenario authors such as `env.app`, sits between the scenario and any runtime. It is the vocabulary for acting on the application (tap, type, open a screen) and observing it (what is shown, what is pending); a portable scenario is written in that vocabulary alone. Each runtime implements it once, as a runtime driver: in process, by rendering production UI and delivering input through rendered controls; in the UI, by operating real browser or native controls. Feature-specific sequences such as logging in are shared helpers over the interface, so a new feature adds helpers, never a driver.

**In a scenario**

- Write the story in the shared vocabulary only. No runner imports, selectors, rendering setup or platform automation in the scenario.
- Build feature-specific sequences as shared helpers over `env.app`. A login helper fills and submits the real form; a screen-opening helper follows the real navigation controls. A helper's name reveals its purpose and never hides the tested decision, an important intermediate state or the scenario's principal assertions.

**The harness guarantees**

- Neither driver shortcuts a journey by invoking navigation or application-state operations directly.
- Drivers implement no product decisions, never mutate application state to achieve an outcome, and never substitute a fake success for an unsupported interaction.
- An action completes when the interaction has been submitted. It does not implicitly await every resulting network operation and hide pending states.
- Every observation method has an explicit contract. Both runtimes establish the same claimed fact, even through different mechanisms; where one name would hide different strengths of evidence, visible and external expectations are separate methods.
- Runner capabilities and exclusions are declared explicitly. An unsupported expectation fails loudly; it never becomes a silent no-op.

## 7. Control when external events happen; never sleep

**Why:** the interesting states are the ones in between: the request is in flight, the server has accepted it, the client has not heard back. A sleep cannot reliably land on any of them. Explicit controls over when each external event happens make those states observable and the test deterministic.

Provide controls for the ordering and completion of external events. Typical capabilities: hold a request, wait until it is received, release it, withhold a response, deliver a remote change or a push notification, advance virtual time. For the running example, a scenario can hold the end-trip request to check the pending state on screen, release it to check the confirmation, or lose the response to check that the app recovers without ending the trip twice.

**In a scenario**

- Prefer descriptive scoped conditions for ordinary cases: run an interaction with the network unavailable, a request rejected, or a response held or lost. Reach for request handles and explicit ordering when the ordering itself is the behavior under test: retries, cancellation, replacement, duplicate submissions, concurrent device actions.
- Never synchronize on a fixed sleep. Wait for readiness and progress on an application or domain signal, and use bounded timeouts to diagnose missing progress, not to establish that a state must have occurred.

**The harness guarantees**

- Holding a request before processing is distinct from withholding its response after processing. The second is a remote write that succeeded while the client remains uncertain.
- A fault is specified as preventing processing, changing the operation's outcome, or preventing delivery after processing. A lost response never undoes an accepted mutation. Connectivity reports are separate inputs from actual request success or failure.
- Fault and delivery controls are scoped by device and operation, and by individual request where concurrent operations need it. The scoped-condition helpers are built on these same controls and shared across runtimes.
- Time-dependent business behavior runs on a controlled clock; which platform timing remains real is stated explicitly.
- Waiting for the environment to settle completes runnable work and delivered events, and nothing more. It never releases intentionally held traffic or advances past an intermediate state the scenario is inspecting.
- On failure, pending operations and the device or world state needed to identify the missing transition are reported.

## 8. Check what the user sees and what the world recorded

**Why:** a success message does not prove a write happened, and a correct backend record does not prove the user can see it. Each half can be right while the other is broken, so a scenario that changes something checks both.

When the rider ends a trip, the scenario asserts that a confirmation is shown *and* that the world now records the trip as ended.

**In a scenario**

- Assert separately what the user sees, through the driver, and what the world recorded, through a probe.
- Assert every effect the behavior promises, such as a link copied to the clipboard, an SMS sent or a notification scheduled.
- Assert request counts only when the count itself is promised, such as never sending end-trip twice; otherwise assert outcomes.
- Write expected values independently of the production logic under test: expect a fare of €2.50 rather than computing it with the app's fare function. Reusing the logic reproduces its mistakes on both sides.
- Anchor negative assertions to a defined point: to prove no SMS is sent when ending a trip fails, check once the failure is shown or while the request is held, not right after the tap. An immediate absence check does not prove the effect will not happen.

**The harness guarantees**

- Probes and driver observations are read-only. They may wait for a state or report it; they never move the world or the application into it.

## 9. Test and maintain the fake like production code

**Why:** every scenario trusts the fake, so a wrong fake makes every scenario wrong in the same direction. The fake earns that trust by being tested, grounded in provider evidence, and corrected whenever production proves it mistaken.

When production shows that ending a trip can fail with a conflict the fake never produced, the fake learns that conflict, a scenario reproduces it, and the fix lands in the application.

**Maintaining the fake**

- Ground modeled behavior in provider contracts, canonical documentation and observed production behavior. Record the source or reason for significant semantics and known limitations.
- Fail explicitly on unknown requests and unsupported capabilities. Keep deterministic scenarios closed to accidental real external traffic.
- Where an authoritative machine-readable API contract exists, validate requests and responses against it automatically at the common fake transport boundary in every runtime. Use a pinned, checked-in contract with an explicit refresh process; violations fail the test. This establishes agreement with the contract's representation, separately from the fake's behavioral correctness.
- When an incident or scenario exposes a discrepancy, record expected and observed outcomes with reproduction evidence. The task owner resolves the intended behavior and implementation scope. Update the behavior specification, journeys, affected tests and coverage records together; distinguish a resolved decision from implemented and verified behavior. Extend the fake with independently established provider semantics and keep regression coverage for the resolved outcome.
- Update fake behavior and its contract checks when the consumed provider contract changes. Preserve explicitly supported older behavior where compatibility is part of the application's contract.
- Fake tests establish internal consistency; agreement with the real provider requires the provider-verification suite. Report unsupported semantics and verification gaps rather than treating a passing suite as parity.

## In practice

With the setup in place, two questions come up on every change: what does a passing run prove, and does this scenario follow the principles?

### Put each test in the suite that proves its claim

A passing run proves exactly what its kind of test can observe, and nothing beyond it. Four kinds of evidence, each answering a different question:

| Evidence              | Question it answers                                      | How it is gathered                                           | When it runs                                                 |
| --------------------- | -------------------------------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| Behavioral matrix     | Does the application behave correctly against the world? | Every scenario, through the in-process driver and the fake world | On every change, fast enough for immediate feedback          |
| UI slice              | Does it still hold on the real platform?                 | The same scenarios, all of them or a selected subset, through the UI driver, plus layout, hit testing, focus, accessibility and native integration checks | A later CI stage, parallelized; representative journeys may also run on every change |
| Fake tests            | Does the fake implement the modeled rules correctly?     | Tests of its invariants, transitions, authorization, error semantics and transport projections | On every change, alongside the matrix                        |
| Provider verification | Does the fake agree with the real provider?              | A small suite against a controlled real provider or equivalent authoritative evidence | On a schedule, when the consumed contract changes, and when an incident questions the fake |

Every behavioral scenario runs in the matrix and can run in the UI slice, since both use the same scenarios, drivers and fake world; the slice is never a second copy. How much of the matrix the slice repeats is a cost decision: at minimum the representative journeys and every check whose claim needs the real platform, at most everything. Because it drives real controls, the slice can also run on a physical device and leave a visual trace of each journey, screenshots or a recording, which no in-process run can produce. Unit tests of localized computation (parsers, fare calculators, record migrations) sit beneath the matrix: they cover input spaces a scenario would be too expensive to enumerate, and they are outside this contract.

### Review every scenario against the checklist

Use this when writing or reviewing a scenario; each item names the principle it enforces, so a reviewer can start here and read the principle only when a line is unclear. The target is a test body that reads as the product interaction: every line establishes a relevant fact, describes an action or external condition, or states an expected outcome. Apply the checks to the scenario's actual claim; do not add unrelated interactions or assertions to satisfy a template.

- **The name states the behavior.** Name the meaningful condition and expected outcome, not an implementation method or ticket identifier alone.
- **The case shows what makes it different** ([P5](#5-seed-only-the-facts-that-matter-let-defaults-fill-in-the-rest)). Common prerequisites sit in a small, nearby suite setup recipe, run afresh for each test. Case-specific facts stay visible in the case.
- **Defaults are declared and predictable** ([P5](#5-seed-only-the-facts-that-matter-let-defaults-fill-in-the-rest)). Omitted relationships use a suite's explicitly designated context or documented minimal defaults. Tests name alternative users, systems or trips when those distinctions matter; setup never picks an arbitrary entity or relies on a previous test.
- **Seeding speaks the domain language** ([P5](#5-seed-only-the-facts-that-matter-let-defaults-fill-in-the-rest)). `world.seed…` declares meaningful facts, with coherent prerequisites supplied automatically. No DTO construction, response fixtures, identifier plumbing or separate materialization in scenarios.
- **Helpers name understandable actions** ([P6](#6-write-each-scenario-once-run-it-in-every-runtime)). Shared sequences such as logging in or preparing an operation execute real app interactions. Their names reveal their purpose; they do not hide the tested decision, the important intermediate state, or the scenario's principal assertions.
- **The app reaches its state through the product** ([P6](#6-write-each-scenario-once-run-it-in-every-runtime)). Suite setup, case setup and helpers use user interactions and world or platform events. No injected navigation stack, route, session, Redux state, cache or local-storage write skips an application workflow.
- **Relaunch tests the application's restoration** ([P1](#1-give-each-scenario-a-fresh-world-and-each-device-its-own-app-instance)). Reach the state first, recreate the runtime against its naturally persisted data and the same world, then observe normal startup. The shared scenario works in both eligible runtimes without test-driven route restoration or repeated navigation to the expected screen.
- **External conditions are descriptive** ([P7](#7-control-when-external-events-happen-never-sleep)). Scoped conditions such as a lost completion response for ordinary cases; request handles, timing details and delivery order only when those mechanics are themselves the behavior under test.
- **The important expectations remain visible** ([P8](#8-check-what-the-user-sees-and-what-the-world-recorded)). State the promised UI outcome and the relevant independent external effect separately. No broad helpers such as `expectEverythingWorked()`, and no expectations calculated by the production logic under test.
- **Ordering and evidence match the claim** ([P7](#7-control-when-external-events-happen-never-sleep), [P8](#8-check-what-the-user-sees-and-what-the-world-recorded)). Wait for actual observable progress, keep meaningful pending-state assertions, and anchor negative assertions to a defined boundary. Distinguish requests from committed effects, and settled absence from absence throughout an interval.
- **Runtime machinery stays outside the story** ([P6](#6-write-each-scenario-once-run-it-in-every-runtime)). Lifecycle boilerplate, selectors, renderer setup, runner branches and native automation mechanics live in fixtures and adapters.
- **The scenario stands on its own** ([P1](#1-give-each-scenario-a-fresh-world-and-each-device-its-own-app-instance), [P6](#6-write-each-scenario-once-run-it-in-every-runtime)). It owns a fresh world, cleans up on failure, and uses shared actions and expectations across eligible runtimes. Any simulated capability or excluded behavior is explicit; no helper silently turns an unsupported check into success.