# Sofar Remote Power Control (RWV registers)

Sofar G3 protocol section *4.2 Remote control (0x1100-0x12FF)* contains one block of registers,
**0x1105-0x110C**, whose read/write attribute is `RWV` instead of the usual `RW`. The protocol defines
`V` as:

> **Volatile** - Variable registers, support frequent write operations

These eight registers are the **only** ones in the entire Sofar G3 protocol carrying that flag. Every
other control register - feed-in limitation (0x1023/0x1024), passive mode (0x1187-0x118C), time-of-use
and timed charging (which the protocol explicitly documents as persisting to EEPROM) - is plain `RW`
with no such guarantee. If you drive your inverter from an automation that recalculates a setpoint every
few seconds, this block is the only place Sofar sanctions that.

Everything below marked *measured* comes from a **HYD 20KTL-3PH** running firmware as shipped in 2026,
driven from this integration for 45 minutes across export limits of 0-100 %.

## What these registers can and cannot do

0x1106 and 0x1107 are *limits*, not setpoints. The protocol calls them "Output/Input **maximum active
power percentage**" - a cap on the inverter's own AC active output (0x0485), **not** on the power crossing
your meter. The inverter's own energy management still decides how to operate within those caps.

| What you want | Register | Verdict |
|---|---|---|
| Dynamic curtailment on a fast signal (negative prices, grid limit), with a ramp | 0x1106 + 0x110A | **Yes** - this is what the block is for, and the only wear-free way to do it |
| Cap grid draw / peak shaving | 0x1107 Active Power Import Limit | **Yes** - new capability |
| Reactive power / power factor | 0x1108 / 0x1109 | **Yes** |
| Reduce export while the battery still has charge headroom | 0x1106 | Yes, but passive `Desired Grid Power` = 0 already does this |
| **Zero feed-in when the battery is full and PV exceeds the load** | - | **No** - see below. Use `FeedIn: Maximum Power` (0x1024) |
| Desired grid power setpoint (`Passive: Desired Grid Power`) | - | **No** |
| Battery charge/discharge window (`Passive: Minimum/Maximum Battery Power`) | - | **No** |

There is no register in the RWV block that sets a grid power target or a battery power window. Forcing
the battery to charge from the grid still requires **Passive mode** (0x1187-0x118C) or TOU/timed
charging. The passive-mode entities are deliberately left untouched by this feature and remain the
supported fallback: if the RWV path misbehaves on your inverter, set `Remote: Power Control Mode` to
`Disabled` and carry on using `Passive: Update Battery Charge/Discharge` exactly as before.

### Why a 0 % export limit is not 0 W at the meter

*Measured.* With the limit at 0 % the inverter kept exporting 0.9-3.1 kW. It honours the cap by pushing
power into the battery, and it was **never observed curtailing PV**. The behaviour across the whole
session fits one law:

```
inverter AC output  =  (PV - battery charge acceptance)  +  limit
grid export         =  inverter AC output - house load
```

So the residual export at a 0 % limit is `PV - house load - whatever the battery will take`:

| Time | PV | Battery | House load | Limit | Grid |
|---|---|---|---|---|---|
| 07:38 | 13.5 kW | +8.7 kW | 1.7 kW | 0 % | **3.1 kW export** |
| 07:44-07:47 | 13.4 kW | +8.6 kW | 3.8 kW | 0 % | **0.9-1.1 kW export** |
| 07:55-08:00 | 13.8 kW | +8.9 kW | 5.7 kW | 0 % | **0.85 kW import** |
| 08:06 | 14.3 kW | -0.5 kW | 0.8 kW | 50 % | 13.2 kW export (4.8 kW residual + 10 kW limit) |

The third row is the same limit doing exactly what you would expect - the house load happened to consume
the residual. The limit itself is accurate: the response is **1 % of rated power per 1 % written**,
verified for 0-30 % and at 50 % on a 20 kW machine.

For genuine zero feed-in you need the **anti-reflux** function, which the protocol describes against the
grid connection point - *"VDE4105 safety regulation grid-connected power limit"*, measured at the PCC
(0x0488). That is `FeedIn: Limitation Mode` (0x1023) plus `FeedIn: Maximum Power` (0x1024) and the
`FeedIn: Update` button. Those are plain `RW`, so use them for state changes, not as a setpoint you
rewrite every minute.

### Don't run two controllers at once

*Measured.* The session above ran with passive mode commanding `Desired Grid Power` = -15500 W (export as
hard as you can) the whole time. Every time the remote autorepeat window expired, export snapped straight
back to -15.5 kW, because the release restores both limits to 100 % while passive mode keeps commanding.
The inverter arbitrates between the two, and what you get is neither setting.

Before using the remote limits, release the other controller: set `Passive: Desired Grid Power` to 0 (and
press `Passive: Update Battery Charge/Discharge`), or put `Energy Storage Mode` back to `Self Use`. The
`Remote: Control Conflict` sensor reports when the inverter is being driven from more than one place.

Note also that `Passive: Update Battery Charge/Discharge` has **no autorepeat**, and with
`Passive: Timeout` disabled (0x1184 = 0) the inverter holds the last setpoint indefinitely. Nothing
requires you to rewrite the passive registers on a timer - writing them only when the target value
actually changes cuts EEPROM writes by more than an order of magnitude.

## Entities

Direct, one-register-per-write entities (write immediately when changed, useful for manual testing):

| Entity | Register |
|---|---|
| `Power Control (bitmask)` (disabled by default) | 0x1105 |
| `Active Power Export Limit` | 0x1106 |
| `Active Power Import Limit` | 0x1107 |
| `Reactive Power Setting` | 0x1108 |
| `Power Factor Setting` | 0x1109 |
| `Active Power Change Rate` | 0x110A |
| `Reactive Power Response Time` | 0x110B |

These numbers also read their register back, so `Active Power Export Limit` is the reliable way to confirm
the inverter accepted a limit. 0x110C (`SVG Fixed Reactive Power`) is documented as "Not used, readable"
and is exposed as a read-back sensor only.

The **Remote Power** feature, which adds the heartbeat and the fail-safe on top:

| Entity | Purpose |
|---|---|
| `Remote: Power Control Mode` | Builds the 0x1105 enable bitmask: `Disabled`, `Active Power Control` (bit0), `Active + Reactive Power` (bit0+bit1), `Active + Reactive (Power Factor)` (bit0+bit1+bit2) |
| `Remote: Export Limit Percent` | Export cap in **%** of rated power, in 0.1 % steps - the recommended entity |
| `Remote: Import Limit Percent` | Import cap in **%** of rated power |
| `Remote: Export Power Limit` | Export cap in **W**. `-1` means "not used, take the percent entity". `0` is a valid hard-zero export limit. Needs a rated power, see below |
| `Remote: Import Power Limit` | Import cap in **W**, same `-1` convention |
| `Remote: Rated Power Override` | Rated power in W for the W -> % conversion. `0` = take it from the `Inverter power rating` sensor (0x06ED) |
| `Remote: Power Limit Change Rate` | 0x110A value, only sent in `Full` write mode (disabled by default) |
| `Remote: Autorepeat Duration` | How long the limits stay applied, in seconds |
| `Remote: Update Power Limits` | Button - applies the values and opens the autorepeat window |
| `Remote: Autorepeat Remaining` | Seconds left before the fail-safe release |
| `Remote: Applied Export Limit` / `... Percent` | What the last write actually sent. `unknown` until the button has been pressed once |
| `Remote: Limit Source` | Which entity the applied limit came from: `W entity`, `percent entity`, or `percent entity (W ignored: rated power unknown)` |
| `Remote: Control Conflict` | Other controllers commanding the inverter while these limits are applied |
| `Remote: Power Control Write Mode` | `Short (0x1105-0x1107)` or `Full (0x1105-0x110C)` (disabled by default, see below) |

### Watts or percent: which one wins

Export and import each have two entities and exactly one of them is in effect at a time. The **Watt entity
decides**, and you never have to park the percent one:

| `Remote: Export Power Limit` | What gets written | `Remote: Export Limit Percent` |
|---|---|---|
| `-1` (any negative) | the percent entity | in effect |
| `0` | a hard zero export limit | ignored |
| e.g. `5000` | 5000 W, converted to 25.0 % on a 20 kW inverter | ignored |

So `0` is a real limit, not "unused" - only a negative value hands control back to the percent entity. The
import pair works the same way and is decided independently, so you can run export from Watts while the
import cap stays on its percent entity. `Remote: Limit Source` always reports which one was used.

### The Watt entities need a rated power

The register is a percentage, so a Watt value has to be divided by the inverter's rated power. That comes
from the `Inverter power rating` sensor (0x06ED), which **reads 0 on some models** - confirmed on the
HYD 20KTL-3PH. When that happens the W entity cannot be used: set `Remote: Rated Power Override` to your
inverter's rated power in W (e.g. `20000`) and the W entities work with the register's full 0.1 %
resolution. Without a rated power the W value is ignored, the percent entity is used instead,
`Remote: Limit Source` says so, `Remote: Applied Export Limit` reads `unknown`, and a warning is logged
once.

If you would rather not depend on the rating at all, use `Remote: Export Limit Percent` - it accepts
0.1 % steps, which is one register step, i.e. 20 W on a 20 kW inverter.

## The fail-safe, and why you need it

Passive mode has an inverter-side watchdog: `Passive: Timeout` (0x1184) and `Passive: Timeout Action`
(0x1185) make the inverter revert on its own if communication stops. **The RWV block has no equivalent.**
An export limit of 0 % written before Home Assistant crashes or the RS485 link dies would stay applied
indefinitely.

`Remote: Update Power Limits` therefore uses the same autorepeat machinery as the SolaX remote control:

1. Pressing the button writes the limits and opens a window of `Remote: Autorepeat Duration` seconds.
2. On every polling cycle *while the window is open*, the current entity values are re-written. So once
   the window is open an automation can simply change the W or % number and the next cycle picks it up -
   no button press per update. Once the window has closed, changing a number does nothing until the
   button is pressed again.
3. When the window expires, one final write clears 0x1105 to `0` and restores both limits to `100 %`.

So keep `Remote: Autorepeat Duration` at a value you are comfortable having the limit persist for
without supervision, and have your automation press the button (or re-press it) as it updates the
setpoint. Setting `Remote: Power Control Mode` to `Disabled` and pressing the button releases everything
immediately and stops the repeat.

Do not use the direct `Active Power Export Limit` / `Active Power Import Limit` numbers at the same time
as the heartbeat - they write the same registers and the heartbeat will overwrite them on the next cycle.

## Prerequisites

1. **0x1105 enable bits.** 0x1106/0x1107 have no effect until bit0 (active power enable) is set. That is
   what `Remote: Power Control Mode` does, and it is the only enable that was needed on the test unit.
2. **0x0900 `Remote Config` (installer level).** The protocol's *"Only when the enable bit is turned on
   can the function of the corresponding register be reflected"* note belongs to the 0x09xx block -
   0x0900 bit0 gates 0x0901/0x0902, the installer-level twins of 0x1106/0x110A - not to 0x110x. It is
   unknown whether it also gates this block; on the test unit the limits worked without touching it. The
   `Remote Config` number is disabled by default; enable it in the entity registry to read the value.
3. **Confirmation is model-dependent.** `Derating Status` (0x0477) bit 7 reads *"Remote active control"*
   and `Remote Control Status` (0x0478) reports the reactive/PF equivalents, but on the HYD 20KTL-3PH
   0x0477 reads 0 even while a limit is demonstrably clamping - as does 0x06ED. `Derating Enable Status`
   (0x047C, disabled by default) carries the corresponding *enable* flags and is likely to behave the
   same way. Use the 0x1106 read-back plus `Remote: Applied Export Limit` instead.

## Ramp rate

*Measured.* A limit change is followed as a ramp - about 2 % of rated power per second on the test unit,
so a change from 100 % to 10 % took roughly 35 seconds. That rate is 0x110A, which the default `Short`
write does not send, so the inverter keeps whatever value it has stored. If you need a different rate,
write it once with the direct `Active Power Change Rate` number. It is deliberately not part of the Short
write: the write is contiguous, so including 0x110A would also command reactive power and power factor as
a side effect of every curtailment write.

## Short vs Full write mode

The protocol note for this block reads: *"When writing, the first address is fixed to any address within
this range, and the length is the length of the range"*, which is ambiguous about whether a partial write
is accepted. The default `Short` mode writes only 0x1105-0x1107 (three registers), which is the least
intrusive and is confirmed working on the HYD 20KTL-3PH. If your inverter rejects that write (Modbus
exception, or the read-back values never change), enable the `Remote: Power Control Write Mode` entity and
switch it to `Full`: the whole 0x1105-0x110C block is then written in one operation, with 0x1108-0x110C
re-sent from their current read-back values so your reactive-power and power-factor settings are preserved
unchanged. If any of those read-backs is unavailable, the write falls back to Short rather than commanding
reactive power 0 and power factor 0.

## Automation example

```yaml
# Curtail export during negative price hours, wear-free
- alias: Sofar curtail export on negative price
  triggers:
    - trigger: numeric_state
      entity_id: sensor.electricity_price
      below: 0
  actions:
    # release the other controller first - see "Don't run two controllers at once"
    - action: number.set_value
      target:
        entity_id: number.sofar_passive_mode_grid_power
      data:
        value: 0
    - action: button.press
      target:
        entity_id: button.sofar_passive_mode_battery_charge_discharge
    - action: select.select_option
      target:
        entity_id: select.solax_remote_power_control_mode
      data:
        option: Active Power Control
    - action: number.set_value
      target:
        entity_id: number.solax_remote_export_limit_percent
      data:
        value: 0
    - action: number.set_value
      target:
        entity_id: number.solax_remote_autorepeat_duration
      data:
        value: 3600
    - action: button.press
      target:
        entity_id: button.solax_remote_update_power_limits
```

Adjust the entity ids to your hub name. To release early, set `Remote: Power Control Mode` to `Disabled`
and press the button again. If the battery may be full while PV exceeds the house load, pair this with
`FeedIn: Maximum Power` - the RWV limit alone will not stop the residual export.

## Known gaps

- Whether 0x1106 ever curtails PV is still open. The measurements above were all taken with the battery
  accepting charge; the decisive test is a 0 % limit with the battery at 100 % SOC and PV above the house
  load. If PV then drops to match the load, this block *can* replace `FeedIn: Maximum Power` = 0.
- Whether 0x0900 bit0 gates this block (prerequisite 2).
- Which models populate 0x06ED, 0x0477/0x0478 and 0x047C. On the HYD 20KTL-3PH they read 0.
- Unrelated but adjacent: peak shaving 0x1132/0x1133 are true grid-side sell/buy caps in Watts, but only
  in peak-shaving mode, and the integration writes 0x1132 alone although the protocol note demands first
  address 0x1130 with length 4. Anti-reflux mode 3 ("Phase power mode") is not offered by the select.
