# SwarmMarket launch adaptation

This launch deploys only `SwarmMarket` on Ethereum, chain id 1. Its existing nonpayable
`constructor(address registry, address imd, address treasury)` already satisfies the factory's
static-argument, empty-chain requirements and is unchanged. Constructor arguments, in order:

1. Registry: `0x53e1e2f33f05086086adf618ca92c3cb0ab111cd`, the existing launch #2099.
2. IMD: `0xd34a99bc0f67ae1bbd63c660e6d0b0dd03e263b7`.
3. Treasury: `0x571e17b13d8f6492bbc372a160520b20bfda320c`.

There is no owner/admin, ETH handling, initializer, proxy, upgrade, launch token, new dependency,
external library link or loop in SwarmMarket. No transaction was signed or broadcast. The separate
manifest/deployment steps select SwarmMarket and supply these values; this adaptation does not
create a `launch.json`. AskSwarmRegistry and SwarmWager production source are unchanged.

## Reproductions and decisions

The two supplied proof files were copied to disposable scratch tests and run against the original
contract. The abandon proof failed with `next call did not revert as expected`; the panel proof
failed with `two seats decided a 10,010 IMD market: 3 != 5`. Six additional scratch tests confirmed
the delayed resolve, resolve after the abandonment deadline, one-wei price rise, zero price,
completion-order-dependent payout and trusted writer pairing. Those behavior-characterization
tests asserted the original behavior; all six passed. Scratch files are not part of the deliverable.

### d2521c529966 — resolver-selected time (medium): reproduced, bounded

An unchanged body was sent at `resolveAfter + 41 hours`; resolve also succeeded after three days.
The contract cannot reproduce an offchain ETH price path, but this demonstrates the timing control
on which the finding depends. `src/SwarmMarket.sol` now exposes `RESOLVE_WINDOW = 1 hours` and
rejects later resolution with `TooLate`. The window's final second is inclusive. One hour permits
ordinary transaction inclusion but removes the multi-day option described by the report.

This is a bounded mitigation, not enforcement of fixed semantics. A caller still chooses when
within that hour to ask, and oracle processing is asynchronous. README and NatSpec therefore
require absolute evidence windows and fixed wording for changing facts. The market keeps the
existing opaque JSON API; it does not add a JSON parser, rewrite questions, or pin question hashes.
Tests: `test_resolveRejectsEarlyAndAfterOneHour`,
`test_resolveAtLastSecondPreservesBodyAndChargesExactlyOnce` and the resolve/abandon window fuzz.

### 2c68e2c29ddc — abandon/resolve race (low): reproduced, fixed

The supplied proof confirmed an affordable, two-sided market could be abandoned while resolve was
still legal. Resolve and abandon now have disjoint windows: abandon requires strictly after
`resolveAfter + RESOLVE_WINDOW`. This also avoids leaving an expired market locked for three days.
The finding's proposed price predicate would preserve unlimited delayed resolution, conflicting
with the medium timing finding. A temporal cutoff resolves both findings consistently, rather
than depending on the current price or on an available registry to release stakes.

Tests cover the last valid resolution second, the first abandonment second, all timestamps through
four days in a fuzz test, escrow return, and abandonment while the registry price call reverts.
The original proof's requested ability to resolve after three days is intentionally superseded by
the resolution cutoff, not retained as a regression expectation.

### 57ea9cc16237 — two-seat panel (low): reproduced, fixed

The proof settled Yes on a verified 2-of-2 answer. `MIN_PANEL = 7` now gates usable answers;
otherwise the market voids with reason 21. Existing quorum verification and strict-majority checks
remain in force. A seven-seat floor is the report's conservative minimum, not economic security
for unlimited value. No new admin or configurable policy is introduced.

Tests use the actual registry's signed callback (under 200,000 gas): 2-of-2 and 6-of-6 void without
market fees, 4-of-7 settles, and minority, tied and below-quorum answers still void. Both outcomes
and the original fee accounting remain covered by the existing market suite.

### 98db55a5a17c — oracle-price increase (low): reproduced, remediated

An increase from 0.5 IMD to 0.5 IMD plus one wei made resolve revert; action price zero did too.
`topUpFee(uint256 id, uint96 amount)` now lets anyone fund an Open market until the resolution
deadline. It pulls an explicitly approved amount, updates only that market's escrow, and emits
`FeeToppedUp`. No stake, payout rights or authority are created. Unspent donations return to the
creator, the original escrow's beneficiary; this is explicit in the function and README.

Resolve still approves and pays exactly the current price. It now explicitly rejects zero price.
Price increases above the uint96 escrow capacity, action retirement or a prolonged outage remain
external liveness limits; abandonment after the one-hour window releases stakes and unspent fees.

Tests cover a one-wei shortfall funded by a third party after staking closes, surplus donations,
price decreases, abandonment/one-sided refunds, zero price, the final allowed second, zero amount,
overflow, wrong phases, missing allowance, failed transfers with full rollback, and registry
rejection followed by retry. The multi-market conservation fuzz includes top-ups and price changes.

### f59677ea2f6c — late completion/timeout race (low): reproduced, fixed with explicit deadline

A snapshot test at `askedAt + 3 days + 1` produced Void when settle preceded completion, and Yes
when completion preceded settle. The same timestamps and signed answer led to different payouts.

`settle` now checks the three-day cutoff before reading the registry. After it, an Asked market
always voids with reason 3, regardless of transaction order or registry availability. At the exact
deadline a usable answer can still settle, while an unanswered ask still returns `StillPending`.

**Behavior change:** this is a settlement deadline, not only an answer-delivery deadline. An answer
delivered earlier but never settled within three days also leads to a void. Once settled, claims
remain available indefinitely. The registry's public record stores issuance/expiry, not arrival
time; issuance is not proof of timely delivery. Preserving a precise delivery cutoff would need
changes to the already-live registry or an additional authenticated delivery mechanism. A creator
grace privilege is unsuitable because the creator can also stake on either side; extending the
timeout merely moves the race. This small deterministic rule keeps the three-day duration and
permissionless operation. Clients/participants must settle promptly, as documented in README.

Regression tests cover both orders of late completion, exact deadline and one second later,
an early-delivered answer left unsettled, a registry outage, and a claim a year after timely
settlement. No test claims that an offchain mempool can be inspected by the market.

### 2d018e34cea4 — legacy registry manifest (info): confirmed, archived

`contracts/launch.json` named AskSwarmRegistry and the old registry arguments. It is renamed,
byte-for-byte unchanged, to `contracts/launch-2099.json`, so it is clearly historical rather than
the manifest for this launch. The original registry-only `script/Deploy.s.sol` is retained as
historical project tooling; README explicitly excludes it from this deployment. No registry
redeployment script or market broadcast path is introduced. The subsequent manifest step must
list exactly SwarmMarket and the three addresses above, not `$contract:AskSwarmRegistry`.

### 095b5312bce3 — writer/question pairing (info): confirmed trust assumption, unchanged

A local test signer and the actual registry show that a different signed `questionHash` delivered
under the market's request id settles the market. This is a model of privileged writer/signer
behavior, not an attack reproduced against the live key. An unprivileged user cannot manufacture
such a verified attestation. The report itself identifies this as protocol trust, not a market
defect. Pinning opaque relative-window questions cannot be added without changing the question
format and oracle integration. README records reliance on the writer, signer and panel.

No imported report was silently dismissed as non-reproducible. The two informational reports
were established by source inspection and/or local modeling, not by impersonating a live signer.

## Other files and compatibility

- `test/SwarmMarket.t.sol`: reuse the already-vendored token/Intake mocks when no fork is present,
  as the registry suite already does. Retain the real-contract fork branch and all 11 behavioral
  tests. Resolve helpers now use the market's stored resolveAfter; the abandonment assertion uses
  the new boundary. Baseline plain `forge test` previously failed in this suite's setup.
- `test/SwarmWager.t.sol`: the same offline fallback fixes its baseline setup failure. No wager
  behavior or assertions change; this is necessary for the project's full offline test command.
- `test/SwarmMarketReview.t.sol`: permanent reproduction/regression, boundary, malformed answer,
  price, accounting and maximum-pool tests, plus multi-market conservation fuzzing. It uses the
  actual registry contract and existing token/Intake fixtures, not a substitute settlement model.
- `test/SwarmMarketDeployment.t.sol`: deploys SwarmMarket through a CREATE2 probe with the exact
  mainnet argument values on an empty chain, checks predicted address, immutable values, size,
  forbidden opcodes, absence of child deployments/owner, zero-address guards and rejection of ETH
  at deployment and runtime. It deploys no registry and imports no removable protected inputs.
- `README.md`: launch handoff, constants, beneficiary rules, mandatory deadlines, immutable
  dependencies and oracle trust assumptions. The 2% fee, split, repeated/both-side staking,
  pull claims, payout formula and one-claim-per-address behavior are unchanged.

The existing `foundry.toml` already has `bytecode_hash = "none"`. All build configuration,
dependencies and protected paths are untouched. No installed package or network access is
required for verification. No new production question is hardcoded; test bodies are transport
fixtures and the existing factual question tests are retained.

## Verification

Validation uses the repository's own configuration and the required seed:

```sh
forge build
FOUNDRY_FUZZ_SEED=0x0f536a14fba389a9523f0a9ad8dcf2109d0df2a4968734af849c1184468ea88b forge test --summary
```

Both commands passed. The full suite reports **151 passed, 0 failed, 0 skipped across 12 suites**,
including the unchanged protocol conformance vector, all original market/wager tests, and both
registry invariant campaigns (24,576 and 12,288 calls, no handler reverts). Each of the three market
fuzz tests ran 256 cases with the required seed. The build retains lint warnings about timestamp
use, casts and events after external calls; it has no compilation errors.

The deployment rehearsal passed. SwarmMarket creation bytecode is 8,935 bytes (9,031 with the three
address arguments), runtime is 8,629 bytes, and library link references are empty. The historical
manifest was verified byte-for-byte against the original, and `git diff --check` passed.

This review uses source inspection and Foundry tests/fuzzing; it does not claim Slither/Mythril,
a fresh live-mainnet fork or independent security attestation. Mainnet signing, deployment and
explorer verification belong to the launch pipeline after acceptance, and remain outstanding here.
