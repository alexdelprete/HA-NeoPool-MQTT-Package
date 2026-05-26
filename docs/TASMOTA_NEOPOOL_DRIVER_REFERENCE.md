# Tasmota NeoPool Driver — Internals Reference

This document captures the **driver-source-level** details of the Tasmota
NeoPool sensor module (`xsns_83_neopool.ino`) so that questions about what
the firmware emits over MQTT and which commands it accepts can be answered
without re-reading 172 k characters of C every time.

It complements the user-facing
[`TASMOTA_NEOPOOL_MQTT_REFERENCE.md`](https://github.com/alexdelprete/ha-sugar-valley-neopool/blob/main/docs/TASMOTA_NEOPOOL_MQTT_REFERENCE.md)
(in the companion integration repo) by adding **the why and the where**:
verbatim `ResponseAppend_P` lines, the `kNPCommands` table, the Modbus
register addresses each JSON path maps to, the status-bit masks, and the
i18n macro origins.

## Source of truth

- File: `tasmota/tasmota_xsns_sensor/xsns_83_neopool.ino`
- Repo: <https://github.com/arendst/Tasmota>
- Driver author: @curzon01 (Norbert Richter)
- Line numbers below refer to the `development` branch as analyzed at
  ~172 k chars (the full source is too large to fit in a single chat
  fetch — see `docs/` history for the snapshot SHA when this file was
  written).

## How telemetry is built

The TELE/SENSOR JSON is constructed by `NeoPoolShow(bool json)` (driver
function starting around line 2032). Top-level object key is
`D_NEOPOOL_NAME = "NeoPool"`. Sub-keys are emitted only when the
corresponding module/relay is detected — when not detected the key is
**absent** (not present-with-zero), so HA `availability_template`s
should check `is defined` rather than relying on a 0 value.

### Format macros (driver lines 991–999)

```c
#define NEOPOOL_FMT_PH          "%*_f"
#define NEOPOOL_FMT_RX          "%d"
#define NEOPOOL_FMT_CL          "%*_f"
#define NEOPOOL_FMT_CD          "%d"
#define NEOPOOL_FMT_ION         "%*_f"
#define NEOPOOL_FMT_HIDRO       "%*_f"
#define D_NEOPOOL_UNIT_PERCENT  "%"
#define D_NEOPOOL_UNIT_GPERH    "g/h"
```

`%*_f` is Tasmota's extended printf where the `*` argument is the number
of decimals (per-module resolution flag stored in `NeoPoolSettings.flags`).

## JSON telemetry map

### `NeoPool.Time` (line 2047)

- Source: `MBF_PAR_TIME_LOW`/`_HIGH` (0x0408/0x0409, 32-bit unix timestamp)
- Format: ISO string from `GetDT()`
- Verbatim: `ResponseAppend_P(PSTR("\"" D_JSON_TIME "\":\"%s\","), GetDT(NeoPoolGetDataLong(MBF_PAR_TIME_LOW)).c_str());`

### `NeoPool.Type` (line 2050)

- Source: `MBF_PAR_UICFG_MACHINE`
- Format: string from `kNeoPoolMachineNames`

### `NeoPool.Modules` object — `NeoPoolAppendModules()` lines 2018–2027

All 0/1 module-present flags:

| JSON path | Source check |
|---|---|
| `NeoPool.Modules.pH` | `MBF_PAR_MODEL & MBMSK_MODEL_PH1/PH2` |
| `NeoPool.Modules.Redox` | `MBF_PAR_MODEL & MBMSK_MODEL_RX` |
| `NeoPool.Modules.Hydrolysis` | `MBF_PAR_MODEL & MBMSK_MODEL_HIDRO` |
| `NeoPool.Modules.Chlorine` | `MBF_CL_STATUS & MBMSK_CL_STATUS_MEASURE_ACTIVE` |
| `NeoPool.Modules.Conductivity` | `MBF_CD_STATUS & MBMSK_CD_STATUS_MEASURE_ACTIVE` |
| `NeoPool.Modules.Ionization` | `MBF_PAR_MODEL & MBMSK_MODEL_ION` |

### `NeoPool.Temperature` (line 2056)

- Emitted only if `MBF_PAR_TEMPERATURE_ACTIVE != 0`
- Source: `MBF_MEASURE_TEMPERATURE` (0x0106), scaled `/10`, then `ConvertTemp()` to HA system unit
- Unit: °C

### `NeoPool.Powerunit` object (lines 2067–2107)

| JSON path | Source | Scale | Unit |
|---|---|---|---|
| `Powerunit.Version` | `MBF_POWER_MODULE_VERSION` | `"V<hi>.<lo>"` | — |
| `Powerunit.NodeID` | `MBF_POWER_MODULE_NODEID` | 6×16-bit hex (obfuscated unless `flag6.neopool_outputsensitive`) | — |
| `Powerunit.5V` | `MBF_VOLT_5` | `/620.69` | V |
| `Powerunit.12V` | `MBF_VOLT_12` | `/1000` | V |
| `Powerunit.24-30V` | `MBF_VOLT_24_36` | `/1000` | V |
| `Powerunit.4-20mA` | `MBF_AMP_4_20_MICRO` | `/100` | mA |

### `NeoPool.pH` object (lines 2111–2141, only if pH module present)

| JSON path | Source | Scale | Notes |
|---|---|---|---|
| `pH.Data` | `MBF_MEASURE_PH` (0x0102) | `/100` (float) | resolution from `flags.ph` |
| `pH.Min` | `MBF_PAR_PH2` | `/100` | low setpoint |
| `pH.Max` | `MBF_PAR_PH1` | `/100` | high setpoint |
| `pH.State` | `MBF_PH_STATUS & MBMSK_PH_STATUS_ALARM` | raw bits | enum 0..6 — see `kNeoPoolpHAlarms` |
| `pH.Pump` | derived from `MBF_PH_STATUS` (CTRL_ACTIVE + ACID/BASE_PUMP_ACTIVE) | 0=idle, 1=pumping, 2=waiting | |
| `pH.FL1` | `MBF_PH_STATUS & MBMSK_PH_STATUS_CTRL_BY_FL` | inverted bit | flow-switch present |
| `pH.Tank` | `MBF_PH_STATUS` alarm == `ALARM6` | 0=empty, 1=OK | |

### `NeoPool.Redox` object (lines 2145–2150, only if Redox module present)

| JSON path | Source | Format | Unit |
|---|---|---|---|
| `Redox.Data` | `MBF_MEASURE_RX` (0x0103) | raw int | mV |
| `Redox.Setpoint` | `MBF_PAR_RX1` | raw int | mV |
| `Redox.Tank` | `MBF_RX_STATUS & MBMSK_RX_STATUS_ALARM == ALARM6` | 0=empty, 1=OK | — |

### `NeoPool.Chlorine` object (lines 2154–2160, only if Chlorine module present)

| JSON path | Source | Scale | Unit |
|---|---|---|---|
| `Chlorine.Data` | `MBF_MEASURE_CL` (0x0104) | `/100` (float) | ppm |
| `Chlorine.Setpoint` | `MBF_PAR_CL1` | `/100` (float) | ppm |

Notable asymmetry: **no `Tank` field** for Chlorine, even though
`MBMSK_CL_STATUS_*` bits exist.

### `NeoPool.Conductivity` — FLAT SCALAR (lines 2162–2165)

This is the most important "gotcha" in the driver, so it gets its own
section below. Short version: emit line is

```c
if (NeoPoolIsConductivity()) {
  ResponseAppend_P(PSTR(",\"" D_NEOPOOL_CONDUCTIVITY "\":" NEOPOOL_FMT_CD),
                   NeoPoolGetData(MBF_MEASURE_CONDUCTIVITY));
}
```

resolving to `,"Conductivity":<int>`. Note: **not** an object —
unlike pH/Redox/Chlorine, there is no `.Data`/`.Setpoint`/`.Tank`
sub-tree.

### `NeoPool.Ionization` object (lines 2169–2178, only if Ionization module present)

| JSON path | Source | Format |
|---|---|---|
| `Ionization.Data` | `MBF_ION_CURRENT` | float, resolution `flags.ion` |
| `Ionization.Setpoint` | `MBF_PAR_ION` | float |
| `Ionization.Max` | `MBF_PAR_ION_NOM` | float — device's nominal max |

### `NeoPool.Hydrolysis` object (lines 2182–2234, only if Hydrolysis module present)

| JSON path | Source | Scale | Notes |
|---|---|---|---|
| `Hydrolysis.Data` | `MBF_HIDRO_CURRENT` | `/10` (float) | unit depends on mode |
| `Hydrolysis.Unit` | derived from `NeoPoolIsHydrolysisInPercent()` | `"%"` or `"g/h"` | |
| `Hydrolysis.Setpoint` | `MBF_PAR_HIDRO` | `/10` (float) | |
| `Hydrolysis.Max` | `MBF_PAR_HIDRO_NOM` | `/10` (float) | nominal max production |
| `Hydrolysis.Percent.Data` | derived `data*100/max` | int | always % |
| `Hydrolysis.Percent.Setpoint` | derived `setpoint*100/max` | int | always % |
| `Hydrolysis.Runtime.Total` | `MBF_CELL_RUNTIME_LOW/HIGH` 32-bit | duration string | |
| `Hydrolysis.Runtime.Part` | `MBF_CELL_RUNTIME_PART_LOW/HIGH` | duration string | |
| `Hydrolysis.Runtime.Pol1` | `MBF_CELL_RUNTIME_POLA_LOW/HIGH` | duration string | |
| `Hydrolysis.Runtime.Pol2` | `MBF_CELL_RUNTIME_POLB_LOW/HIGH` | duration string | |
| `Hydrolysis.Runtime.Changes` | `MBF_CELL_RUNTIME_POL_CHANGES_LOW/HIGH` | long int | |
| `Hydrolysis.State` | derived from `MBF_HIDRO_STATUS` | string: `"off" / "Flow" / "Pol1" / "Pol2"` | |
| `Hydrolysis.Cover` | `MBF_HIDRO_STATUS & MBMSK_HIDRO_STATUS_COVER` | 0/1 | |
| `Hydrolysis.Boost` | `MBF_HIDRO_STATUS & SHOCK_ENABLED` + `MBF_CELL_BOOST & NO_REDOX_CTL` | 0=off, 1=on no-redox, 2=on with-redox | JSON key is `D_NEOPOOL_JSON_SHOCK = "Boost"` |
| `Hydrolysis.Low` | `MBF_HIDRO_STATUS & MBMSK_HIDRO_STATUS_LOW` | 0/1 | |
| `Hydrolysis.FL1` | `MBF_HIDRO_STATUS & MBMSK_HIDRO_STATUS_FL1` | inverted | |
| `Hydrolysis.Redox` | `MBF_HIDRO_STATUS & REDOX_ENABLED` | 0/1 | |

### `NeoPool.Filtration` object (lines 2237–2243, only if `MBF_PAR_FILT_GPIO != 0`)

| JSON path | Source | Format |
|---|---|---|
| `Filtration.State` | `MBF_RELAY_STATE` bit at `MBF_PAR_FILT_GPIO-1` | 0/1 |
| `Filtration.Speed` | `NeoPoolGetFiltrationSpeed()` | 1/2/3 (only if speed != 0) |
| `Filtration.Mode` | `MBF_PAR_FILT_MODE` | enum 0..6 — see `kNeoPoolFiltrationMode` |

### `NeoPool.Light` scalar (line 2246, only if `MBF_PAR_LIGHTING_GPIO != 0`)

- Format: int 0/1 from `MBF_RELAY_STATE` bit at `MBF_PAR_LIGHTING_GPIO-1`
- JSON key from Tasmota core: `D_JSON_LIGHT` = `"Light"`

### `NeoPool.Relay` object (lines 2252–2287)

| JSON path | Source | Format |
|---|---|---|
| `Relay.State` (array) | `MBF_RELAY_STATE` bits 0..`NEOPOOL_RELAY_MAX-1` | array of 0/1 |
| `Relay.Aux` (array) | `MBF_RELAY_STATE` bits 3.. | array of 0/1 (AUX1..AUX4) |
| `Relay.Acid` | bit at `MBF_PAR_PH_ACID_RELAY_GPIO-1` | 0/1 (only if mapped) |
| `Relay.Base` | bit at `MBF_PAR_PH_BASE_RELAY_GPIO-1` | 0/1 |
| `Relay.Redox` | bit at `MBF_PAR_RX_RELAY_GPIO-1` | 0/1 |
| `Relay.Chlorine` | bit at `MBF_PAR_CL_RELAY_GPIO-1` | 0/1 |
| `Relay.Conductivity` | bit at `MBF_PAR_CD_RELAY_GPIO-1` | 0/1 — the brine pump |
| `Relay.Heating` | bit at `MBF_PAR_HEATING_GPIO-1` | 0/1 |
| `Relay.UV` | bit at `MBF_PAR_UV_RELAY_GPIO-1` | 0/1 |
| `Relay.Valve` | bit at `MBF_PAR_FILTVALVE_GPIO-1` | 0/1 |

⚠ **`Relay.State[n]` mapping is not portable.** Indices 0..2 (often
quoted as pH/Filtration/Light) are **per-installation** — they
depend on the physical wiring. Indices 3..6 are stable as
AUX1..AUX4 by Sugar Valley convention. The named-relay keys
(`Relay.Acid`, `Relay.Base`, `Relay.Redox`, `Relay.Chlorine`,
`Relay.Conductivity`, `Relay.Heating`, `Relay.UV`, `Relay.Valve`)
are the correct **functional** state source — Tasmota emits each
only when its `MBF_PAR_*_RELAY_GPIO` is assigned to a physical
relay, so the entity becomes `unavailable` automatically when not
wired.

### `NeoPool.Connection` object (lines 2290–2302, only if `NEOPOOL_CONNSTAT` build flag + `flags.conn_stat`)

Sub-keys (all int): `Time`, `MBRequests`, `MBNoError`,
`MBIllegalFunc`, `MBIllegalDataAddr`, `MBIllegalDataValue`,
`MBSlaveError`, `MBAck`, `MBSlaveBusy`, `MBNotEnoughData`,
`MBMemParityErr`, `MBCRCErr`, `MBGWPath`, `MBGWTarget`, `MBRegErr`,
`MBRegData`, `MBTooManyReg`, `MBUnknownErr`, `MBNoResponse`,
`DataOutOfRange`.

## NP\* command table

### Command name table (driver lines 1301–1340)

```c
const char kNPCommands[] PROGMEM = D_PRFX_NEOPOOL "|"  // Prefix "NP"
  D_CMND_NP_RESULT "|"             // NPResult
  D_CMND_NP_READ "|"               // NPRead
  D_CMND_NP_READL "|"              // NPReadL
  D_CMND_NP_READLSB "|"            // NPReadLSB
  D_CMND_NP_READMSB "|"            // NPReadMSB
  D_CMND_NP_WRITE "|"              // NPWrite
  D_CMND_NP_WRITEL "|"             // NPWriteL
  D_CMND_NP_WRITELSB "|"           // NPWriteLSB
  D_CMND_NP_WRITEMSB "|"           // NPWriteMSB
  D_CMND_NP_BIT "|"                // NPBit
  D_CMND_NP_BITL "|"               // NPBitL
  D_CMND_NP_FILTRATION "|"         // NPFiltration
  D_CMND_NP_FILTRATIONMODE "|"     // NPFiltrationmode
  D_CMND_NP_FILTRATIONSPEED "|"    // NPFiltrationspeed
  D_CMND_NP_BOOST "|"              // NPBoost
  D_CMND_NP_TIME "|"               // NPTime
  D_CMND_NP_LIGHT "|"              // NPLight
  D_CMND_NP_PHMIN "|"              // NPpHMin
  D_CMND_NP_PHMAX "|"              // NPpHMax
  D_CMND_NP_PH "|"                 // NPpH (alias for NPpHMax)
  D_CMND_NP_REDOX "|"              // NPRedox
  D_CMND_NP_HYDROLYSIS "|"         // NPHydrolysis
  D_CMND_NP_IONIZATION "|"         // NPIonization
  D_CMND_NP_CHLORINE "|"           // NPChlorine
  D_CMND_NP_CONTROL "|"            // NPControl
  D_CMND_NP_TELEPERIOD "|"         // NPTelePeriod
  D_CMND_NP_SAVE "|"               // NPSave
  D_CMND_NP_EXEC "|"               // NPExec
  D_CMND_NP_ESCAPE "|"             // NPEscape
  D_CMND_NP_ONERROR "|"            // NPOnError
  D_CMND_NP_PHRES "|"              // NPPHRes
  D_CMND_NP_CLRES "|"              // NPCLRes
  D_CMND_NP_IONRES "|"             // NPIONRes
  D_CMND_NP_SETOPTION "|"          // NPSetOption
  D_CMND_NP_SO                     // NPSO (alias for NPSetOption)
#ifdef NEOPOOL_EMULATE_GPERH
  "|" D_CMND_NP_GPERH              // NPgPerh
#endif
;
```

### Resolved command list

| Command | Handler | Writes / does |
|---|---|---|
| `NPResult` | `CmndNeopoolResult` | sets response format |
| `NPRead` | `CmndNeopoolReadReg` | arbitrary register read |
| `NPReadL` | `CmndNeopoolReadReg` | 32-bit (LSB+MSB) read |
| `NPReadLSB` / `NPReadMSB` | `CmndNeopoolReadReg` | low/high-word read |
| `NPWrite` | `CmndNeopoolWriteReg` | arbitrary register write |
| `NPWriteL` | `CmndNeopoolWriteReg` | 32-bit write |
| `NPWriteLSB` / `NPWriteMSB` | `CmndNeopoolWriteReg` | low/high-word write |
| `NPBit` | `CmndNeopoolBit` | set/clear single bit |
| `NPBitL` | `CmndNeopoolBit` | set/clear bit in 32-bit value |
| `NPFiltration` | `CmndNeopoolFiltration` | filtration on/off → `MBF_PAR_FILT_MANUAL_STATE` |
| `NPFiltrationmode` | `CmndNeopoolFiltrationMode` | `MBF_PAR_FILT_MODE` |
| `NPFiltrationspeed` | `CmndNeopoolFiltrationSpeed` | sets speed via `MBF_PAR_FILTRATION_CONF` mask |
| `NPBoost` | `CmndNeopoolBoost` | `MBF_CELL_BOOST` (hydrolysis shock) |
| `NPTime` | `CmndNeopoolTime` | `MBF_PAR_TIME_LOW/HIGH` |
| `NPLight` | `CmndNeopoolLight` | light relay state |
| `NPpHMin` | `CmndNeopoolpHMin` | `MBF_PAR_PH2` low setpoint |
| `NPpHMax` | `CmndNeopoolpHMax` | `MBF_PAR_PH1` high setpoint |
| `NPpH` | `CmndNeopoolpHMax` (shared) | high setpoint (alias) |
| `NPRedox` | `CmndNeopoolRedox` | `MBF_PAR_RX1` setpoint |
| `NPHydrolysis` | `CmndNeopoolHydrolysis` | `MBF_PAR_HIDRO` setpoint |
| `NPIonization` | `CmndNeopoolIonization` | `MBF_PAR_ION` setpoint |
| `NPChlorine` | `CmndNeopoolChlorine` | `MBF_PAR_CL1` setpoint |
| `NPControl` | `CmndNeopoolControl` | dumps all relay/GPIO mappings |
| `NPTelePeriod` | `CmndNeopoolTelePeriod` | driver-local telemetry interval |
| `NPSave` | `CmndNeopoolSave` | writes `MBF_SAVE_TO_EEPROM` |
| `NPExec` | `CmndNeopoolExec` | writes `MBF_EXEC` |
| `NPEscape` | `CmndNeopoolEscape` | writes `MBF_ESCAPE` (clear error) |
| `NPOnError` | `CmndNeopoolOnError` | driver-local error handling |
| `NPPHRes` | `CmndNeopoolPHRes` | driver-local pH display resolution |
| `NPCLRes` | `CmndNeopoolCLRes` | driver-local chlorine resolution |
| `NPIONRes` | `CmndNeopoolIONRes` | driver-local ionization resolution |
| `NPSetOption` / `NPSO` | `CmndNeopoolSetOption` | driver-local flag bits |
| `NPgPerh` | `CmndNeopoolgPerh` | (build-time `NEOPOOL_EMULATE_GPERH` only) |

**No `NPConductivity` command exists.** Conductivity has no setpoint
register on the NeoPool hardware. The only writable CD register is
`MBF_PAR_CD_RELAY_GPIO` (0x040E), which is the relay-GPIO assignment
for the brine pump.

**No `NPAux<n>` commands exist either.** AUX relays cannot be toggled
through a dedicated NeoPool command — see Known firmware quirks below.

## Conductivity deep dive

Why this gets its own section: the driver emits Conductivity in a
**different shape** than pH/Redox/Chlorine, and HA templates that
assume the object shape will silently produce empty entities.

### Registers (verbatim)

```c
MBF_MEASURE_CONDUCTIVITY,   // 0x0105*  %     Conductivity level measured in %
MBF_CD_STATUS,              // 0x010A*  mask  Status of the Conductivity-module

MBF_PAR_SAL_AMPS = 0x030A,  // Current command in regulation for which we measure voltage
MBF_PAR_SAL_CELLK,          // 0x030B  Relationship between measured resistance and g/L
MBF_PAR_SAL_TCOMP,          // 0x030C  Temperature deviation from the conductivity

MBF_PAR_CD_RELAY_GPIO,      // 0x040E*  Relay assigned to brine pump (CD module only)
```

`MBF_PAR_SAL_*` are **factory calibration parameters** for the
salinity extrapolation from hydrolysis voltage — not the CD
measurement itself.

### Status bits (driver lines 343–347)

```c
// MBF_CD_STATUS                              // bit
MBMSK_CD_STATUS_RX_PUMP_ACTIVE   = 0x1000,    // 12  Conductivity pump relay on
MBMSK_CD_STATUS_CTRL_ACTIVE      = 0x2000,    // 13  Active CD control module
MBMSK_CD_STATUS_MEASURE_ACTIVE   = 0x4000,    // 14  Active CD measurement module
MBMSK_CD_STATUS_MODULE_PRESENT   = 0x8000,    // 15  CD measurement module detected
```

Of these, the driver **only uses** `MEASURE_ACTIVE` (in
`NeoPoolIsConductivity()`, line 2006). The other three bits exist in
the hardware register but are not emitted to JSON. They can be read
via `NPRead 0x010A` if needed.

### Driver helper

```c
bool NeoPoolIsConductivity(void) {
  return (NeoPoolGetData(MBF_CD_STATUS) & MBMSK_CD_STATUS_MEASURE_ACTIVE);
}
```

### Emit line — verbatim (lines 2162–2165)

```c
// Conductivity
if (NeoPoolIsConductivity()) {
  ResponseAppend_P(PSTR(",\""  D_NEOPOOL_CONDUCTIVITY  "\":" NEOPOOL_FMT_CD),
                   NeoPoolGetData(MBF_MEASURE_CONDUCTIVITY));
}
```

### Critical caveats

1. **Flat scalar, NOT an object.** Output is `,"Conductivity":<int>`.
   Templates expecting `value_json.NeoPool.Conductivity.Data` will
   silently return `None`. Read it as
   `value_json.NeoPool.Conductivity` directly.
2. **Unit is percent (0–100).** Source comment at register table:
   `"Conductivity level measured in %"`. Format is `NEOPOOL_FMT_CD = "%d"`.
3. **i18n key.** The JSON key string `"Conductivity"` resolves from
   `D_NEOPOOL_CONDUCTIVITY` — a **Tasmota i18n macro** defined in the
   language headers (`tasmota/language/*.h`), NOT a driver-local
   `D_NEOPOOL_JSON_*` macro. English builds emit `"Conductivity"`;
   non-English Tasmota firmware may emit a translated string (German
   `"Leitfähigkeit"`, etc.). Stock firmware is English; HA users on
   non-English Tasmota are rare but exist.
4. **No setpoint, no min/max, no tank, no state.** Read-only telemetry.
5. **Key absent when inactive.** When `MBF_CD_STATUS &
   MBMSK_CD_STATUS_MEASURE_ACTIVE == 0`, the key is **omitted from
   JSON**, not present-with-zero. HA availability templates should
   guard on `is defined`.
6. **CD relay state lives separately.** The brine pump's on/off state
   is at `NeoPool.Relay.Conductivity` (0/1) — emitted only if
   `MBF_PAR_CD_RELAY_GPIO != 0`. Same string key as the measurement
   but at a different JSON path, so they coexist.

## Driver-local JSON key macros

These are the `D_NEOPOOL_JSON_*` `#define`s that name the JSON keys
emitted by this driver. Verbatim, driver lines ~830–903:

```c
#define D_NEOPOOL_JSON_FILTRATION_NONE        ""
#define D_NEOPOOL_JSON_FILTRATION_MANUAL      "Manual"
#define D_NEOPOOL_JSON_FILTRATION_AUTO        "Auto"
#define D_NEOPOOL_JSON_FILTRATION_HEATING     "Heating"
#define D_NEOPOOL_JSON_FILTRATION_SMART       "Smart"
#define D_NEOPOOL_JSON_FILTRATION_INTELLIGENT "Intelligent"
#define D_NEOPOOL_JSON_FILTRATION_BACKWASH    "Backwash"
#define D_NEOPOOL_JSON_MODULES                "Modules"
#define D_NEOPOOL_JSON_POWERUNIT              "Powerunit"
#define D_NEOPOOL_JSON_CHLORINE               "Chlorine"
#define D_NEOPOOL_JSON_CONDUCTIVITY           "Conductivity"
#define D_NEOPOOL_JSON_FILTRATION             "Filtration"
#define D_NEOPOOL_JSON_FILTRATION_MODE        "Mode"
#define D_NEOPOOL_JSON_FILTRATION_SPEED       "Speed"
#define D_NEOPOOL_JSON_HYDROLYSIS             "Hydrolysis"
#define D_NEOPOOL_JSON_PERCENT                "Percent"
#define D_NEOPOOL_JSON_CELL_RUNTIME           "Runtime"
#define D_NEOPOOL_JSON_CELL_RUNTIME_TOTAL     "Total"
#define D_NEOPOOL_JSON_CELL_RUNTIME_PART      "Part"
#define D_NEOPOOL_JSON_CELL_RUNTIME_POL1      "Pol1"
#define D_NEOPOOL_JSON_CELL_RUNTIME_POL2      "Pol2"
#define D_NEOPOOL_JSON_CELL_RUNTIME_CHANGES   "Changes"
#define D_NEOPOOL_JSON_IONIZATION             "Ionization"
#define D_NEOPOOL_JSON_LIGHT                  "Light"
#define D_NEOPOOL_JSON_LIGHT_MODE             "Mode"
#define D_NEOPOOL_JSON_REDOX                  "Redox"
#define D_NEOPOOL_JSON_RELAY                  "Relay"
#define D_NEOPOOL_JSON_RELAY_PH_ACID          "Acid"
#define D_NEOPOOL_JSON_RELAY_PH_BASE          "Base"
#define D_NEOPOOL_JSON_RELAY_RX               "Redox"
#define D_NEOPOOL_JSON_RELAY_CL               "Chlorine"
#define D_NEOPOOL_JSON_RELAY_CD               "Conductivity"
#define D_NEOPOOL_JSON_RELAY_HEATING          "Heating"
#define D_NEOPOOL_JSON_RELAY_UV               "UV"
#define D_NEOPOOL_JSON_RELAY_FILTVALVE        "Valve"
#define D_NEOPOOL_JSON_AUX                    "Aux"
#define D_NEOPOOL_JSON_STATE                  "State"
#define D_NEOPOOL_JSON_TYPE                   "Type"
#define D_NEOPOOL_JSON_UNIT                   "Unit"
#define D_NEOPOOL_JSON_COVER                  "Cover"
#define D_NEOPOOL_JSON_SHOCK                  "Boost"
#define D_NEOPOOL_JSON_OFF                    "OFF"
#define D_NEOPOOL_JSON_ON                     "ON"
#define D_NEOPOOL_JSON_LOW                    "Low"
#define D_NEOPOOL_JSON_SETPOINT               "Setpoint"
#define D_NEOPOOL_JSON_MIN                    "Min"
#define D_NEOPOOL_JSON_MAX                    "Max"
#define D_NEOPOOL_JSON_PHPUMP                 "Pump"
#define D_NEOPOOL_JSON_FLOW1                  "FL1"
#define D_NEOPOOL_JSON_TANK                   "Tank"
#define D_NEOPOOL_JSON_BIT                    "Bit"
#define D_NEOPOOL_JSON_NODE_ID                "NodeID"
```

Note: `D_NEOPOOL_JSON_CONDUCTIVITY = "Conductivity"` is used inside
`NeoPool.Modules.Conductivity` and `NeoPool.Relay.Conductivity` — but
the **top-level** `NeoPool.Conductivity` measurement key uses the
Tasmota-core i18n macro `D_NEOPOOL_CONDUCTIVITY` (no `JSON` in the
name), which is a different macro from a different header. See the
Conductivity caveat above.

Optional NEOPOOL_CONNSTAT block adds:

```c
#define D_NEOPOOL_JSON_CONNSTAT               "Connection"
#define D_NEOPOOL_JSON_CONNSTAT_MB_REQUESTS   "MBRequests"
// ... 17 more counter sub-keys
#define D_NEOPOOL_JSON_CONNSTAT_DATA_OOR      "DataOutOfRange"
```

Root key (line 828): `#define D_NEOPOOL_NAME "NeoPool"`.

## MBMSK bit tables

### pH status — `MBF_PH_STATUS`

`MBMSK_PH_STATUS_ALARM` (low nibble, enum 0..6 — see
`kNeoPoolpHAlarms`); `MBMSK_PH_STATUS_CTRL_BY_FL` (flow-switch
present); `MBMSK_PH_STATUS_CTRL_ACTIVE` (control active);
`MBMSK_PH_STATUS_ACID_PUMP_ACTIVE` / `_BASE_PUMP_ACTIVE`;
`MBV_PH_ACID_BASE_ALARM6` = tank-empty alarm.

### Redox status — `MBF_RX_STATUS`

`MBMSK_RX_STATUS_ALARM`; `MBV_RX_ALARM6` = tank-empty alarm.

### Chlorine status — `MBF_CL_STATUS`

`MBMSK_CL_STATUS_MEASURE_ACTIVE` (bit 14, drives
`NeoPool.Modules.Chlorine`).

### Conductivity status — `MBF_CD_STATUS`

See "Conductivity deep dive" above. Only `MEASURE_ACTIVE` is
exposed to JSON.

### Hydrolysis status — `MBF_HIDRO_STATUS`

`MBMSK_HIDRO_STATUS_COVER`, `_SHOCK_ENABLED`, `_LOW`, `_FL1`,
`_REDOX_ENABLED`, plus polarity bits driving the State string.

## Known firmware quirks

1. **`NPAux<n>` commands do not exist** in `kNPCommands`. HA
   integrations that send `cmnd/SmartPool/NPAux1..4` will see state
   reads work (from `NeoPool.Relay.Aux[]`) but the commands are
   silently ignored by firmware. Correct mechanism: send
   `cmnd/SmartPool/Power<gpio>` (Tasmota core relay control) or use
   `NPWrite` to set the bit in `MBF_RELAY_STATE` directly.

2. **`NPConductivity` does not exist.** There is no setpoint
   register, so no setter command. Conductivity is read-only
   telemetry. The brine pump can be enabled/disabled by writing the
   GPIO assignment in `MBF_PAR_CD_RELAY_GPIO` (0x040E) via `NPWrite`,
   but that's hardware configuration, not runtime control.

3. **Conductivity key is i18n-keyed.** Stock English firmware emits
   `"Conductivity"`; non-English Tasmota localized builds emit a
   translated string. Stock builds account for >99% of installs in
   practice — HA templates should assume English unless a user
   reports otherwise.

4. **`Relay.State[n]` indices are not portable.** Indices 0..2 in
   particular are commonly assumed to map to pH/Filtration/Light but
   are configurable per controller. Use the named relay keys for
   functional state. Indices 3..6 are stable as AUX1..AUX4 by
   convention.

5. **Chlorine has no `Tank` field** despite `MBMSK_CL_STATUS_*` bits
   existing in hardware. Asymmetry vs. pH and Redox.

6. **`Hydrolysis.State`** is a **string** (`"OFF"`, `"FLOW"`,
   `"POL1"`, `"POL2"`), not an int — unlike pH.State and other state
   fields. Templates must compare strings.

7. **CD JSON key absent when CD module inactive** — not zero, not
   null. Same for pH/Redox/Chlorine/Ionization/Hydrolysis subtrees:
   they are entirely omitted when the corresponding module is not
   present. Use `availability_template` checking `is defined`.

## See also

- User-facing reference (companion integration repo):
  [`TASMOTA_NEOPOOL_MQTT_REFERENCE.md`](https://github.com/alexdelprete/ha-sugar-valley-neopool/blob/main/docs/TASMOTA_NEOPOOL_MQTT_REFERENCE.md)
- Quick-reference cheat sheet:
  [`MQTT_TOPIC_QUICK_REFERENCE.md`](https://github.com/alexdelprete/ha-sugar-valley-neopool/blob/main/docs/MQTT_TOPIC_QUICK_REFERENCE.md)
- Driver source (definitive truth):
  [`xsns_83_neopool.ino`](https://github.com/arendst/Tasmota/blob/development/tasmota/tasmota_xsns_sensor/xsns_83_neopool.ino)
- Parity tracking with the Python integration:
  [`PARITY_WITH_INTEGRATION.md`](./PARITY_WITH_INTEGRATION.md) (this repo only)
