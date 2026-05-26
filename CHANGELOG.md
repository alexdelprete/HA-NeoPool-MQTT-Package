# Changelog

All notable changes to the **HA NeoPool MQTT Package**
([ha_neopool_mqtt_package.yaml](ha_neopool_mqtt_package.yaml)) are documented
in this file.

The format is loosely based on [Keep a Changelog](https://keepachangelog.com/),
and the project loosely follows the upstream Tasmota NeoPool firmware versions
for context.

For the under-the-hood firmware reference (verbatim driver source map),
see [`docs/TASMOTA_NEOPOOL_DRIVER_REFERENCE.md`](docs/TASMOTA_NEOPOOL_DRIVER_REFERENCE.md).
For the companion Python integration parity matrix,
see [`docs/PARITY_WITH_INTEGRATION.md`](docs/PARITY_WITH_INTEGRATION.md).

## [Unreleased] — 2026-05-26

### Added

- **Chlorine module** — full read+write support:
  - `sensor.neopool_mqtt_chlorine_data` reading `NeoPool.Chlorine.Data` (ppm)
  - `number.neopool_mqtt_chlorine_setpoint` writing `cmnd/SmartPool/NPChlorine`
    (range 0–10 ppm, step 0.1)
- **Hydrolysis** — completed exposure of driver-emitted fields:
  - `sensor.neopool_mqtt_hydrolysis_setpoint_gh` — read-only g/h companion to
    the existing `hydrolysis_setpoint` (%) slider
  - `sensor.neopool_mqtt_hydrolysis_max` — nominal max production (g/h),
    static device capability
  - `sensor.neopool_mqtt_hydrolysis_unit` — current unit string (`"%"` or
    `"g/h"`) from firmware
  - `binary_sensor.neopool_mqtt_hydrolysis_redox_controlled` — `1` when the
    cell is configured for redox-driven production
- **Ionization module** — added in previous unreleased iteration:
  - `sensor.neopool_mqtt_ionization_data` reading `NeoPool.Ionization.Data`
  - `number.neopool_mqtt_ionization_setpoint` writing
    `cmnd/SmartPool/NPIonization` (range 0–10, step 0.1). YAML cannot express
    dynamic max from `NeoPool.Ionization.Max`; the package uses static
    `max: 10` (firmware silently rejects out-of-range commands).
- **Conductivity (CD) module** — measurement sensor:
  - `sensor.neopool_mqtt_conductivity_data` reading `NeoPool.Conductivity`
    (% scalar, 0–100). Note: firmware emits CD as a flat integer, **not** an
    object like pH/Redox/Chlorine — template reads
    `value_json.NeoPool.Conductivity` directly. See driver reference for
    details.
- **Controller clock** — diagnostic + sync:
  - `sensor.neopool_mqtt_controller_time` reading `NeoPool.Time` (diagnostic
    category; raw ISO string, no `device_class: timestamp` because the
    firmware string lacks timezone info)
  - `button.neopool_mqtt_sync_controller_time` sending
    `cmnd/SmartPool/NPTime 0` to sync the controller's clock to Tasmota's
    current time
- **Module-presence + relay-state availability gating** for all
  module-dependent entities. Each entity reading a `NeoPool.<Module>.*`
  subtree now uses an `availability_template` checking the parent subtree
  `is defined`, so HA marks entities `unavailable` (rather than showing
  stale values) when the module is removed or the controller goes offline.
- **`docs/` folder** with two new technical references:
  - `docs/TASMOTA_NEOPOOL_DRIVER_REFERENCE.md` — verbatim driver source map
    (JSON paths, command table, status bits, known firmware quirks)
  - `docs/PARITY_WITH_INTEGRATION.md` — entity-by-entity parity tracking
    with the companion Python integration `ha-sugar-valley-neopool`
- **`CHANGELOG.md`** (this file).

### Removed

- 3 indexed relay binary sensors: `neopool_mqtt_relay_ph_state`,
  `_filtration_state`, `_light_state` (reading `NeoPool.Relay.State[0..2]`).
  These assumed a fixed physical relay-to-function mapping that does not
  hold across installations — the mapping is configurable per controller.
  The **named** relay binary sensors (`relay_acid_state`, `_base_state`,
  `_redox_state`, `_chlorine_state`, `_conductivity_state`, `_heating_state`,
  `_uv_state`, `_valve_state`) introduced in v4.0 are the correct
  functional-state source. Mirrors the fix made in commit
  [`9dae48b`](https://github.com/alexdelprete/ha-sugar-valley-neopool/commit/9dae48b0c83e4179655436776d83313ff6729194)
  of the companion integration.

  **Migration**: after reloading the package YAML, the three deleted
  entities will appear as `unavailable` in HA's entity registry. They can
  be removed via Settings → Devices & Services → MQTT → Entities, or
  ignored.

### Known issues (carried over)

- **AUX1–4 switches** (`neopool_mqtt_aux<n>_switch`) send
  `cmnd/SmartPool/NPAux<n>` — that command name does **not** exist in the
  Tasmota NeoPool driver's command table (verified against `kNPCommands` at
  driver lines 1301–1340). State reads work (from
  `NeoPool.Relay.Aux[<n-1>]`), but writes are silently ignored by the
  firmware. This bug is present in both this package and the companion
  Python integration.

  Workaround: use the underlying Tasmota relay numbers via
  `cmnd/SmartPool/Power<gpio>` topics in your automations.

## [v4.0] — 2026-03-23

### Added (upstream)

- 7 new **named relay binary sensors** for the functional-state of each
  named relay output: `Base`, `Redox`, `Chlorine`, `Conductivity`,
  `Heating`, `UV`, `Valve`. Each is only emitted by the firmware when its
  corresponding `MBF_PAR_*_GPIO` register is non-zero, so the HA entity
  goes `unavailable` automatically when the relay is not wired.
  Contributed by @curzon01 in commit `952328a`.

### Changed

- Version header bumped from `v3.5` to `v4.0`.

## [v3.5] — 2025-06-22

### Added (upstream)

- `neopool_mqtt_redox_tank_level` binary sensor (`NeoPool.Redox.Tank`).
  Requires Tasmota >= v15.0.1.1. Contributed by @curzon01 in commit
  `90e1a5e`.

## Earlier history

For changes before v3.5, see the `git log` of
[ha_neopool_mqtt_package.yaml](ha_neopool_mqtt_package.yaml) in the
[upstream repository](https://github.com/alexdelprete/HA-NeoPool-MQTT-Package).
Key milestones:

- **2024-07-06** — Connection diagnostic sensors added
  (`NeoPool.Connection.*`) when Tasmota firmware gained data-validation +
  connection-statistics support.
- **2024-05-19** — Water-flow binary sensor (`hydrolysis_ctrl_fl1_water_flow`)
  added.
- **2024-02-29** — File renamed from `ha-neopool-mqtt-package.yaml` to
  `ha_neopool_mqtt_package.yaml` (HA naming convention).

## Versioning

The version string in the YAML header (line 1) reflects significant entity
or behavior changes. Bump policy: major bump for breaking entity-id
changes; minor bump for additive entities; cosmetic/non-functional changes
do not bump the header.

## Reporting issues

- Package-specific issues:
  <https://github.com/alexdelprete/HA-NeoPool-MQTT-Package/issues>
- Firmware/driver issues:
  <https://github.com/arendst/Tasmota/discussions/19811>
- Python integration:
  <https://github.com/alexdelprete/ha-sugar-valley-neopool/issues>
