# Parity with the `ha-sugar-valley-neopool` integration

This document tracks the functional parity between two related projects:

- **This repo** — `HA-NeoPool-MQTT-Package` — a Home Assistant YAML
  package that defines all NeoPool MQTT entities purely through HA's
  built-in MQTT integration. Lightweight, no Python code; users drop
  the YAML into their HA config and reload.
- **Companion repo** —
  [`ha-sugar-valley-neopool`](https://github.com/alexdelprete/ha-sugar-valley-neopool)
  — a HACS custom integration in Python. Adds device-info,
  diagnostics, repairs, options flow, and per-entity disabled-by-default
  capabilities the YAML cannot express.

Both projects expose the same underlying Tasmota NeoPool MQTT
telemetry. The integration leads on architecture decisions; the
package follows where it can. This doc lists the current deltas and
the intentional divergences.

For driver-level details on what the firmware emits/accepts, see
[`TASMOTA_NEOPOOL_DRIVER_REFERENCE.md`](./TASMOTA_NEOPOOL_DRIVER_REFERENCE.md).

## Methodology

Parity is judged on the **observable entity surface in Home Assistant**:

- Every JSON path under `NeoPool.*` exposed in either project should
  ideally be exposed in both.
- Naming may differ (the integration has a `YAML_TO_INTEGRATION_KEY_MAP`
  in `const.py` for migration users), but the entity exists in both.
- Domain may differ when the integration leverages capabilities the YAML
  cannot (e.g. `select` vs `sensor` for filtration mode, dynamic-max
  `number`).

## Availability convention (this package)

Every entity reading from a **module-dependent subtree** uses an
`availability_template` checking `value_json.NeoPool.<Subtree> is
defined`, with `availability_topic: tele/SmartPool/SENSOR` so the
template is re-evaluated on each telemetry message. The template also
references `states('sensor.neopool_mqtt_system_model') != 'unavailable'`
to indirectly tie availability to LWT (system_model uses
LWT-based availability and is always present in telemetry).

Subtrees that get the gate:

| Subtree | Gate expression | Entity count |
|---|---|---|
| `NeoPool.pH.*` | `value_json.NeoPool.pH is defined` | 7 |
| `NeoPool.Redox.*` | `value_json.NeoPool.Redox is defined` | 3 |
| `NeoPool.Hydrolysis.*` | `value_json.NeoPool.Hydrolysis is defined` | 14 |
| `NeoPool.Conductivity` (flat scalar) | `value_json.NeoPool.Conductivity is defined` | 1 |
| `NeoPool.Ionization.*` | `value_json.NeoPool.Ionization is defined` | 2 |

Why: the Tasmota driver **omits the whole subtree** when the
corresponding module is not present, rather than emitting an empty
object. Without the gate, HA would show stale or `unknown` values
instead of marking the entity unavailable. See
[`TASMOTA_NEOPOOL_DRIVER_REFERENCE.md`](./TASMOTA_NEOPOOL_DRIVER_REFERENCE.md)
section "Conductivity deep dive" — point 5 explains the absent-key
behavior; the same applies to pH/Redox/Chlorine/Hydrolysis/Ionization
subtrees.

Entities that **do not** get the gate (use plain LWT only):

- `neopool_mqtt_system_model`, `neopool_mqtt_water_temperature`,
  `neopool_mqtt_powerunit_*` — always present in telemetry.
- `neopool_mqtt_modules_*` (the 6 module-presence binary sensors) —
  the `Modules` object is always present.
- `neopool_mqtt_relay_*` — `Relay` object is always present; named
  relay keys naturally absent (unavailable) when GPIO not assigned.
- AUX switches, filtration switches, light switch, selects, button.

The integration follows the same logic in code (its `NeoPoolEntity`
base class explicitly returns `available=False` when the JSON path is
absent, per [9dae48b](https://github.com/alexdelprete/ha-sugar-valley-neopool/commit/9dae48b0c83e4179655436776d83313ff6729194)
in v0.2.16). YAML had no equivalent until this version.

## Entity parity matrix

Only entities that meaningfully differ between the two projects are
called out below. For the full enumeration see the parity-gap analysis
in the project history (or re-derive by enumerating
`ha_neopool_mqtt_package.yaml` and `custom_components/sugar_valley_neopool/`).

### Sensors

| JSON path | Package `unique_id` | Integration `key` | Notes |
|---|---|---|---|
| `NeoPool.Type` | `neopool_mqtt_system_model` | `system_model` | parity |
| `NeoPool.Time` | `neopool_mqtt_controller_time` | `controller_time` | parity — both diagnostic, raw ISO string (no timestamp device_class because no TZ in firmware string) |
| `NeoPool.Temperature` | `neopool_mqtt_water_temperature` | `water_temperature` | parity |
| `NeoPool.pH.Data` | `neopool_mqtt_ph_data` | `ph_data` | parity |
| `NeoPool.Redox.Data` | `neopool_mqtt_redox_data` | `redox_data` | parity |
| `NeoPool.Chlorine.Data` | `neopool_mqtt_chlorine_data` | `chlorine_data` | parity — ppm |
| `NeoPool.Hydrolysis.Percent.Data` | `neopool_mqtt_hydrolysis_data` | `hydrolysis_percent` | naming differs |
| `NeoPool.Hydrolysis.Data` | `neopool_mqtt_hydrolysis_data_gh` | `hydrolysis_data` | naming differs |
| `NeoPool.Hydrolysis.Setpoint` | `neopool_mqtt_hydrolysis_setpoint_gh` | `hydrolysis_setpoint_gh` | parity — read-only g/h companion |
| `NeoPool.Hydrolysis.Max` | `neopool_mqtt_hydrolysis_max` | `hydrolysis_max` | parity — nominal device capacity |
| `NeoPool.Hydrolysis.Unit` | `neopool_mqtt_hydrolysis_unit` | `hydrolysis_unit` | parity — string `"%"` / `"g/h"` |
| `NeoPool.Ionization.Data` | `neopool_mqtt_ionization_data` | `ionization_data` | parity |
| `NeoPool.Conductivity` | `neopool_mqtt_conductivity_data` | `conductivity_data` | parity — flat scalar % |
| `NeoPool.Hydrolysis.Runtime.Changes` | `neopool_mqtt_hydrolysis_runtime_pol_changes` | `hydrolysis_polarity_changes` | naming differs |
| `NeoPool.Powerunit.NodeID` | `neopool_mqtt_powerunit_nodeid` | `system_id` | integration marks as `DIAGNOSTIC`; package does not |
| `NeoPool.Filtration.Mode` | `neopool_mqtt_filtration_mode` (sensor) | `filtration_mode` (sensor + select) | package read-only; integration adds select |
| `NeoPool.Filtration.Speed` | `neopool_mqtt_filtration_speed` (sensor) | `filtration_speed` (sensor + select) | same |
| `NeoPool.Hydrolysis.Boost` | `neopool_mqtt_hydrolysis_boost_mode` (sensor) | `boost_mode` (sensor + select) | same |
| `NeoPool.Connection.*` | `neopool_mqtt_conndiag_*` | `connection_*` | naming differs (mapped in `YAML_TO_INTEGRATION_KEY_MAP`) |

### Binary sensors

| JSON path | Package `unique_id` | Integration `key` | Notes |
|---|---|---|---|
| `NeoPool.Modules.*` (6 modules) | `neopool_mqtt_modules_*` | `modules_*` | parity |
| `NeoPool.Relay.State[0..2]` | — (removed) | — (not present) | indexed mapping unsafe; removed from both as of this change |
| `NeoPool.Relay.State[3..6]` (AUX1..4) | `neopool_mqtt_relay_aux1..4_state` | — (not present, integration uses switches) | package keeps for now to back the AUX switches |
| `NeoPool.Relay.Acid` | `neopool_mqtt_relay_acid_state` | `relay_acid_state` | integration disables-by-default |
| `NeoPool.Relay.Base`/`Redox`/`Chlorine`/`Conductivity`/`Heating`/`UV`/`Valve` | `neopool_mqtt_relay_<name>_state` | `relay_<name>_state` | integration disables-by-default per release notes; YAML cannot disable-by-default |
| `NeoPool.pH.FL1` | `neopool_mqtt_ph_ctrl_fl1` | `ph_fl1` | naming differs |
| `NeoPool.Hydrolysis.FL1` | `neopool_mqtt_hydrolysis_ctrl_fl1` | `hydrolysis_fl1` | naming differs |
| `NeoPool.pH.Tank` / `Redox.Tank` | `*_tank_level` | `*_tank_level` | both use device_class=problem; package uses `payload_on: "0"`, integration uses `invert=True` flag — semantically equivalent |
| `NeoPool.Hydrolysis.Cover` / `.Low` | `hydrolysis_cover` / `_low_production` | `hydrolysis_cover` / `_low_production` | parity |
| `NeoPool.Hydrolysis.Redox` | `neopool_mqtt_hydrolysis_redox_controlled` | `hydrolysis_redox_controlled` | parity — `1` if cell is redox-controlled |

### Switches

| JSON path | Package `unique_id` | Integration `key` | Notes |
|---|---|---|---|
| `NeoPool.Filtration.State` + `NPFiltration` | `neopool_mqtt_filtration_switch` | `filtration` | parity |
| `NeoPool.Light` + `NPLight` | `neopool_mqtt_light_switch` | `light` | parity |
| `NeoPool.Relay.Aux[0..3]` + `NPAux<n>` | `neopool_mqtt_aux<n>_switch` | `aux<n>` | **state read works, write does NOT** — `NPAux<n>` is not in `kNPCommands`. See driver-internals doc, "Known firmware quirks". Bug present in both projects |

### Selects (integration only as control; package has these as sensors)

| JSON path | Package | Integration `key` | Notes |
|---|---|---|---|
| `NeoPool.Filtration.Mode` + `NPFiltrationmode` | (sensor only) | `filtration_mode` | integration controllable; package read-only |
| `NeoPool.Filtration.Speed` + `NPFiltrationSpeed` | (sensor only) | `filtration_speed` | same |
| `NeoPool.Hydrolysis.Boost` + `NPBoost` | (sensor only) | `boost_mode` | same |

### Numbers (setpoint sliders)

| JSON path | Package `unique_id` | Integration `key` | Notes |
|---|---|---|---|
| `NeoPool.pH.Min` + `NPpHMin` | `neopool_mqtt_ph_min` | `ph_min` | parity |
| `NeoPool.pH.Max` + `NPpHMax` | `neopool_mqtt_ph_max` | `ph_max` | parity |
| `NeoPool.Redox.Setpoint` + `NPRedox` | `neopool_mqtt_redox_setpoint` | `redox_setpoint` | parity |
| `NeoPool.Chlorine.Setpoint` + `NPChlorine` | `neopool_mqtt_chlorine_setpoint` | `chlorine_setpoint` | parity — 0–10 ppm |
| `NeoPool.Hydrolysis.Percent.Setpoint` + `NPHydrolysis` | `neopool_mqtt_hydrolysis_setpoint` | `hydrolysis_setpoint` | parity |
| `NeoPool.Ionization.Setpoint` + `NPIonization` | `neopool_mqtt_ionization_setpoint` (static `max: 10`) | `ionization_setpoint` (dynamic max from `NeoPool.Ionization.Max`) | **divergent**: HA MQTT `number` does not support `max_template`; package uses static `max: 10` (AquaScenic HD1 nominal). Firmware silently rejects out-of-range commands. |

### Buttons

| Command | Package `unique_id` | Integration `key` | Notes |
|---|---|---|---|
| `NPEscape` | `neopool_mqtt_clear_error_state` | `clear_error` | naming differs |
| `NPTime 0` | `neopool_mqtt_sync_controller_time` | `sync_controller_time` | parity — sends `0` to sync controller clock to Tasmota's current time |

## Known parity blockers

These are things the YAML package **cannot** express, so they will
remain integration-only:

1. **Dynamic `max` on a `number` entity.** HA MQTT `number` only
   accepts static `min`/`max`/`step`. The integration reads
   `NeoPool.Ionization.Max` per device and clamps; the package hard-codes
   `max: 10` and lets firmware silently reject overshoots.
2. **`entity_registry_enabled_default: false` per entity.** The
   integration disables the 8 named relay binary sensors by default and
   lets users enable the ones their controller actually wires. The
   YAML schema has no per-entity `enabled_by_default` field for MQTT
   entities.
3. **`entity_category: diagnostic`.** Some HA `mqtt` platforms accept
   this (e.g. `sensor`), but inconsistently across domains. The package
   does not currently apply diagnostic category to any entity; the
   integration does for `system_id` and the connection counters.
4. **Reactive cleanup of stale entities.** When entities are removed
   from the YAML (e.g. the 3 indexed `Relay.State[0..2]` sensors
   removed in this version), HA leaves them in the registry as
   `unavailable` until manually purged. The integration runs a
   migration routine (`_migrate_to_canonical_nodeid`) on startup to
   clean these up automatically.
5. **Repair issues / device triggers / options flow.** Integration-only
   concepts. The package has no equivalent surface.

## Only-in-package (intentional)

- AUX1..4 relay binary sensors (`neopool_mqtt_relay_aux<n>_state`).
  Reads from `NeoPool.Relay.State[3..6]`. The integration uses
  `aux<n>` switches reading from `NeoPool.Relay.Aux[]` instead and
  does not expose separate binary sensors. The package keeps the
  binary sensors because removing them in the package would also
  remove the state-read source for the AUX switches without an obvious
  replacement.

## Only-in-integration (porting targets when feasible)

As of 2026-05-26, **all module measurements and setters are now mirrored
in the package**. Remaining integration-only items are architectural,
not data-coverage:

- **`system_id`** as a separate diagnostic-categorized sensor
  (`NeoPool.Powerunit.NodeID` is already in the package as
  `powerunit_nodeid`; the integration just adds the diagnostic
  category and a different name).
- **Diagnostic category** for connection counters
  (`neopool_mqtt_conndiag_*`). Possibly portable — MQTT sensor accepts
  `entity_category`. The package already applies it on
  `controller_time` since v4.0+ — could be extended to the conn-diag
  sensors as well.
- **Filtration mode / speed / Boost as `select` entities.** Could be
  added to the package YAML as a parallel set under `mqtt.select:`.
- **Repair flow / device triggers / options flow / config flow.** Not
  portable to YAML.
- **Disabled-by-default for the 8 named relay binary sensors.** Not
  portable to YAML (no `enabled_by_default` field for MQTT entities).
- **Dynamic max for `ionization_setpoint`.** YAML `number` has no
  `max_template`; package uses static `max: 10`.

## How to use this doc

When making changes to either project:

1. Update the entity table above when adding/removing an entity.
2. Move items between "only-in-X" and the parity matrix as gaps are
   closed.
3. If you discover a new firmware quirk, document it in
   [`TASMOTA_NEOPOOL_DRIVER_REFERENCE.md`](./TASMOTA_NEOPOOL_DRIVER_REFERENCE.md)
   not here — this doc is about HA-side parity, not firmware behavior.
4. Cross-link any related issues / PRs in both repos so users on either
   project can find the discussion.

## See also

- Driver-internals reference:
  [`TASMOTA_NEOPOOL_DRIVER_REFERENCE.md`](./TASMOTA_NEOPOOL_DRIVER_REFERENCE.md)
- Integration repo:
  <https://github.com/alexdelprete/ha-sugar-valley-neopool>
- Integration's user-facing MQTT reference:
  <https://github.com/alexdelprete/ha-sugar-valley-neopool/blob/main/docs/TASMOTA_NEOPOOL_MQTT_REFERENCE.md>
