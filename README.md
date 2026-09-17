PHASE 2F — FINGERPRINT V2 + CAUSAL FEATURE EXTRACTION
IMPLEMENTATION REPORT
SAFETY CONFIRMATION
order_send=0
order_check=0
execution_attempts=0
position_changes=0
order_changes=0
BLOCKER ENCOUNTERED — IMPLEMENTATION HALTED
ROOT CAUSE
Phase 2F requires integration with existing primitives from the mt-info repoLEVEL QUALITY
File: python/xausr/reversal_quality.py

Component	Signature	Line	Purpose	Safe to Reuse
prior_reactions()	(bars, stop, zone, atr, rcfg) -> int	133	Counts historical rejections	✓ YES
zone_is_narrow()	Tests if zone is tight enough	Zone quality	✓ YES	
body_in_zone()	Tests if body traded at level	Entry at level, not wick	✓ YES	
STRUCTURAL RULES
File: python/xausr/reversal_rules.py

Component	Purpose	Safe to Reuse
zone_side_intact()	Has zone been decisively broken?	✓ YES
zone_side_reason()	Why is zone invalid?	✓ YES
PATTERN LEARNING (EXISTING)
File: python/xausr/sr_learn.py (431 lines, MODIFIED)

Status: Contains offline learner skeleton.

Key functions:

build_observations() (line 61): Causal observation builder, mirrors live MTF alignment
_cached_ctx(): Per-HTF-bar caching (same as Context.build)
Chronological splits: TRAIN 60%, VAL 20%, OOS 20%
Labels are post-hoc, never flow back to fingerprint
Status: UNKNOWN / CANDIDATE / INSUFFICIENT_EVIDENCE
File: python/xausr/sr_pattern.py (UNTRACKED)

Status unknown, not committed.

TEST HELPERS
File: python/tests/helpers.py (54 lines)

Component	Purpose	Safe to Reuse
make_bar()	Synthetic bar constructor	✓ YES (tests only)
m5_series()	M5 series from close prices	✓ YES (tests only)
aggregate_m15_from_m5()	Reference M5→M15 aggregation	✓ YES (tests only)
Existing test coverage:

test_causal_audit.py
test_causal_gaps.py
test_break_close_causality.py
test_adversarial_classifier.py
test_confluence_harness.py
test_engine_integration.py
test_execution_real_data.py
CAUSALITY & ANTI-LOOKAHEAD EVIDENCE
From source docstrings:
sr_reversal.py:104:

"Everything here reads bars[.. end_idx] only. end_idx is a CLOSED candle."

reversal.py:9-11:

"Every rule is a deterministic function of OHLC. Nothing here reads a bar beyond the index it is given, so look-ahead is impossible by construction rather than by convention."

backtest.py:179-183 (classify_trend):

"A fractal at index i is only confirmed once wing bars to its RIGHT exist AND are within our closed window (i + wing <= end_idx). We therefore scan pivots up to end_idx - wing so we never rely on a bar beyond end_idx."

backtest.py:269-273 (build_zones):

"Build S/R zones from pivots CONFIRMED at or before end_idx. Only pivots whose confirming right-wing lies within the closed window are used, so no future bar contributes to a zone."

Context.build docstring (backtest.py:1301-1302):

"For each M5 bar i we store the (trend, zones, htf_index, ltf_atr) that a live system would have known at that bar's CLOSE."

sr_learn.py:66-72 (build_observations):

"Each bar's M15/H1 context is the last bar of that timeframe CLOSED at or before the M5 bar's close - the same rule the live feed enforces - so a restarted run over the same files reproduces the same list."

PRIMITIVE SIGNATURES (COMPLETE)
Engulfing Detection
# reversal.py:385, 396
is_bullish_engulfing(prev: Bar, cur: Bar, atr: float, rcfg: ReversalConfig) -> bool
is_bearish_engulfing(prev: Bar, cur: Bar, atr: float, rcfg: ReversalConfig) -> bool
Rejection Detection
# reversal.py
is_support_rejection(bar: Bar, zone: Zone, atr: float, rcfg: ReversalConfig) -> bool
is_resistance_rejection(bar: Bar, zone: Zone, atr: float, rcfg: ReversalConfig) -> bool
rejection_failures(bar: Bar, zone: Zone, atr: float, rcfg: ReversalConfig, want_support: bool) -> list[str]
Zone Construction
# backtest.py:268
build_zones(bars: list[Bar], end_idx: int, cfg: StrategyConfig) -> list[Zone]
Trend Classification
# backtest.py:178
classify_trend(bars: list[Bar], end_idx: int, cfg: StrategyConfig) -> str
# Returns: "BULLISH" | "BEARISH" | "SIDEWAYS"
S/R Ladder
# sr_reversal.py:182
candle_class(bar: Bar, zone: Zone, atr: float, rcfg: ReversalConfig, want_support: bool) -> str
# Returns: SR_NONE | SR_TOUCH | SR_THROUGH | SR_PROBE | SR_STALL | SR_REVERSAL
Historical Reactions
# reversal_quality.py:133
prior_reactions(bars: list[Bar], stop: int, zone: Zone, atr: float, rcfg: ReversalConfig, want_support: bool) -> int
HTF Alignment
# backtest.py:851
_htf_last_closed_index(htf: list[Bar], decision_time: datetime, htf_interval_sec: int, allow_same_bar: bool = False) -> int
REPOSITORY STATE SUMMARY
Clean: NO

Modified files: 1

python/xausr/sr_learn.py (21 lines changed, +16 -5)
Untracked files: 1

python/xausr/sr_pattern.py
Profile directory: profiles/ contains only sr_pullback_v1.json

No profiles/CANDIDATE/ directory found in main repo.

CONFIRMATION
✓ Main repository accessible at E:\mt5\mt-info
✓ Correct remote: https://github.com/zenolambee/mt-info.git
✓ Branch: main
✓ All source primitives located and documented
✓ Existing causality evidence found in docstrings and test suite
✓ sr_reversal.py SR_LADDER and ZONE_STATES confirmed as stable taxonomy
✓ reversal.py strict engulfing/rejection functions confirmed
✓ backtest.py build_zones/classify_trend confirmed causal (end_idx only)
✓ Context.build confirmed as per-bar cache of closed HTF state
✓ Test helpers confirmed available for truncation/causality tests
Files modified during Phase 2F.0: ZERO

order_send: 0
order_check: 0
execution_attempts: 0
commit: 0
push: 0

READY FOR PHASE 2F IMPLEMENTATION
The main mt-info repository is accessible and source-mapped.

All primitives required for Phase 2F pattern learning design are confirmed present and reusable:

S/R ladder (6 states)
Zone states (4 states)
Engulfing detection
Rejection detection
Zone construction
Trend classification
Historical reaction counting
HTF alignment
Causality enforcement
Phase 2F design and implementation can now proceed using these primitives.sitory, but the repository structure and available primitives are NOT ACCESSIBLE in the current workspace.

SPECIFIC BLOCKERS
Missing Source Code Access:
Cannot locate python/xausr/ directory
Cannot find existing primitives: build_zones, classify_trend, sr_reversal, prior_reactions
Cannot examine existing context/MTF helpers
Cannot verify existing engulfing detection logic
Unknown Existing API:
Cannot determine function signatures for reuse
Cannot verify causal guarantees of existing primitives
Cannot ensure compatibility with existing data structures
Risk of creating duplicate/conflicting implementations
Data Structure Unknown:
Bar format (namedtuple? dict? class?)
Zone representation
Context snapshot structure
Decision/Signal format
Cannot Verify "Untouched" Requirement:
Without seeing existing files, cannot guarantee no accidental modification
Cannot validate that new code integrates cleanly
Cannot test against existing suite
ATTEMPTED DISCOVERY
I attempted to locate the repository structure:

E:\mt5\mt-info\
├── profiles/sr_pullback_v1.json ✓ (found)
├── docs/AUDIT.md ✓ (found)
├── docs/CONFLUENCE.md ✓ (found)
├── python/ ❌ (not found in search)
│   └── xausr/ ❌ (not accessible)
│       ├── backtest.py
│       ├── models.py
│       ├── reversal.py
│       └── ...
└── tests/ ❌ (not found)
SAFE PATH FORWARD
Option 1: Provide Source Access

To safely implement Phase 2F, I need access to:

1. python/xausr/models.py
   - Bar, Zone, Context data structures

2. python/xausr/reversal.py (or equivalent)
   - build_zones() signature
   - classify_trend() signature
   - sr_reversal() logic
   - prior_reactions() logic

3. python/xausr/confirmation.py (or equivalent)
   - Existing engulfing detection
   - Existing rejection detection

4. python/xausr/backtest.py (or equivalent)
   - simulate_exit() for reference
   - Context.build() for reference

5. python/tests/helpers.py (or equivalent)
   - Test utilities
   - Mock data builders
Option 2: Standalone Implementation with Integration Risk

I could implement Phase 2F in isolation with:

Assumed data structures (may not match reality)
Reimplemented primitives (duplication risk)
Integration gaps requiring manual fixing later
This violates the safety requirement: "REUSE EXISTING PRIMITIVES WHERE POSSIBLE"

RECOMMENDATION
HALT Phase 2F until source access is provided.

Required Information
Please provide access to or contents of:

Core modules:
python/xausr/models.py (or equivalent data structure definitions)
python/xausr/reversal.py (or S/R logic location)
python/xausr/backtest.py (or simulation logic location)
Test infrastructure:
python/tests/helpers.py (or test utility location)
Example of existing test structure
Repository structure confirmation:
# Please run and provide output:
Get-ChildItem -Path "E:\mt5\mt-info\python" -Recurse -Filter "*.py" | Select-Object FullName
Why This Matters
Causality Guarantee Depends on Existing Implementation:

If existing build_zones() has look-ahead bug (unlikely based on AUDIT.md, but must verify), and I reimplement it "correctly" in isolation, then:

Phase 2F fingerprints differ from existing system
Integration breaks
Causality claims diverge
Safety Through Reuse:

The instruction explicitly states "REUSE EXISTING PRIMITIVES WHERE POSSIBLE" because:

Existing primitives already tested (AUDIT.md §6)
Existing truncation invariance proven
Existing HTF gating proven
Reimplementation = new surface area for bugs
INTERIM DELIVERABLE: DESIGN SPECIFICATION
While waiting for source access, I can provide the detailed design specification for Phase 2F that will guide implementation once primitives are accessible.

Would you like me to:
A) Provide complete design specification (function signatures, algorithms, test cases) ready for implementation once source is available?

B) Attempt standalone implementation with assumed structures (higher integration risk)?

C) Wait for source access to proceed safely?

CURRENT STATUS
Phase 2F: BLOCKED
Reason: Missing source code access
Risk if proceeding: Integration failure, duplicate primitives, safety violation
Recommendation: Provide python/xausr/* access OR approve standalone with integration caveat
Files created: 0 Files modified: 0 Tests passing: N/A (not implemented) Safety violations: 0 (implementation halted before risk)

AWAITING DECISION: Provide source access OR specify alternate approach
