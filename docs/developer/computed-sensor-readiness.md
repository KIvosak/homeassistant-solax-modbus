# Computed Sensor Readiness

Computed sensors have no Modbus register of their own. Their `value_function`
derives a value from other entries in the hub data dictionary. They must not be
evaluated during entity registration, because the first successful Modbus poll
may not have populated their inputs yet.

## Shared evaluation path

All register-less computed sensors are evaluated by
`SolaXModbusHub.evaluate_computed_sensor`. The evaluator:

- leaves a new entity unknown until its readiness contract is satisfied;
- accepts numeric zero and boolean false as valid inputs;
- rejects missing, `None`, NaN and infinite required inputs;
- preserves the last coherent value briefly when inputs/results are invalid;
- expires the entity's availability after three configured input intervals
  without an accepted computation, including complete polling silence;
- requires every mandatory input in the current interval to be fresh;
  dependencies may span device groups in that refresh;
- permits raw inputs from a different interval only from that interval's latest
  accepted snapshot, at most three of its configured intervals old;
- prevents a stale computed intermediate from releasing a downstream calculation;
- resolves computed-on-computed chains in dependency order.

The Energy Dashboard refresh path uses the same evaluator. It resolves the
actual source selected by the parallel setting, rather than accepting an
unrelated alternative. Every contributing hub supplies its own accepted,
bounded-age interval snapshots. Missing contributors yield `None` (HA unknown),
never a partial sum or zero. A topology refresh does not authorize computing
from an unvalidated cache.

## Description contract

Every description with `register < 0` and a `value_function` must explicitly
declare `depends_on`, even when the list is empty.

```python
BaseModbusSensorEntityDescription(
    key="pv_power",
    value_function=value_function_pv_power,
    depends_on=["pv_voltage", "pv_current"],
)
```

The available fields are:

- `depends_on`: every applicable key is required. An empty list means there
  are no readiness inputs; it is appropriate for a true constant or a value
  obtained from external state captured by the function.
- `depends_on_any`: each tuple is an alternative group. At least one
  applicable, valid and fresh key from every group is required.
- `optional_depends_on`: only valid, accepted values are passed to the function;
  invalid or stale values are removed from its private input dictionary.
  Alternatives are filtered identically, so a stale preferred value cannot
  shadow a fresh fallback. When a sensor has no mandatory or alternative inputs, an
  applicable optional input must be fresh to trigger recalculation.
- `readiness_validator`: optional domain-specific validation after the generic
  checks. It receives the source data dictionary and must return a boolean.
- `recompute_each_poll`: use only for time-dependent calculations or values
  maintained by local control code, whose result can change without a Modbus
  dependency becoming fresh.
- `allow_none`: explicitly publish `None` as unknown, for example when local
  remote control is inactive. Use an empty dependency contract for identity
  values maintained by local code, not a self-reference requiring numeric input.

An optional correction requiring several measurements must explicitly check
that its complete correction input set survived filtering. Otherwise use the
documented base calculation, not zero-filled partial correction terms.

## Age and interval boundaries

The availability timer is renewed by publication of an accepted calculation,
not by an attempted poll. Invalid or discarded groups cannot renew it. Expiry
preserves the last numeric value internally but exposes `unavailable`; a later
accepted zero or nonzero measurement restores availability. The timer is
cancelled on entity removal. Its limit is three times the slowest configured
raw dependency interval (the default interval for local/no-input descriptions),
not the communication-failure slowdown interval.

The three communication diagnostics are an explicit exception: they describe
the local poll outcome, not a measured input. The hub publishes them after
recording each completed poll, outside the computed evaluator and its expiry
timer. They therefore remain available during slowdown/polling silence rather
than hiding `Degraded` or `Offline`. This exception does not disable expiry for
other time-dependent or dependency-free computed sensors.

Cross-interval input reuse is deliberately narrower than arbitrary cache use:
only raw registered dependencies qualify; a missing same-interval input or
failed computed intermediate still blocks the calculation. At least one
declared input must be freshly observed for an ordinary calculation. The
latest completed snapshot replaces the previous snapshot even on partial or
discarded reads, and rebuilding polling blocks clears all snapshots. Inputs
from independently scheduled groups/hubs are bounded in age, not simultaneous
physical measurements. No data dictionary or freshness set is modified by the
input overlay. `force=True` is not used to accept cross-hub cache data.

Dependencies not present in the active inverter's description set or data are
ignored. This lets one description cover model variants while still requiring
every input that is applicable to the detected inverter.

## Riemann energy integrals

Riemann energy sensors consume completed, accepted source-hub snapshots rather
than `hub.data` or a cached dashboard mirror. Their callbacks run after the
interval snapshot is published, including failed and discarded intervals.
They validate finite numeric power before and after filtering; zero remains
valid, while booleans and numeric strings are not power measurements.

Integration uses the monotonic observation time of the selected source, not
callback time. Duplicate callbacks or overlays from unrelated scan intervals
cannot re-date a sample, accumulate energy or renew its availability lease.
A newer invalid source interval cannot fall back to an older computed result
from another interval. Source selection and age limits use the actual source
hub, including slave PV variants and parallel-mode mappings.

Invalid/missing/stale power makes the integral unavailable while preserving its
total and breaking the integration interval. The first valid sample after a gap
only establishes a new baseline; the next valid sample resumes integration.
An expiry timer handles complete polling silence, and a sample-time gap check
also protects against delayed timer delivery. Source hub/key changes and
dashboard reactivation likewise break the interval. Missing energy is not
estimated or backfilled: the integral is incomplete across the outage.

The accumulated total and its local reset date are saved through HA's
`RestoreEntity.extra_restore_state_data`, independently of visible availability.
HA omits ordinary extra attributes when an entity is unavailable, so those
attributes alone cannot preserve the total through a restart during an outage.
Legacy numeric restored states remain supported. Cold startup without a saved
total stays unknown until a valid power sample; restart never integrates its
downtime, and the existing local-midnight reset remains in effect.

## Contributor checks

When adding or changing a computed sensor:

1. Declare every input read by the value function as required, alternative or
   optional. Do not rely on `dict.get(..., 0)` for startup readiness.
2. Use a readiness validator for domain rules; do not treat zero as missing in
   the shared evaluator.
3. Add focused tests for cold start, legitimate zero, invalid input and partial
   polling where relevant.
4. Keep direct sensor `value_function` calls out of the entity platform. The
   structural tests enforce both the explicit contract and shared startup path.
5. For energy integrals, test missing data, no-callback expiry, source-hub
   selection, duplicate callbacks and restart while unavailable. Assert both
   HA publication and preserved restore data, not just internal arithmetic.
