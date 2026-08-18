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

## What these registers can and cannot do

**Important:** 0x1106 and 0x1107 are *limits*, not setpoints. The protocol calls them "Output/Input
maximum active power percentage". They cap what the inverter is allowed to do; they do not command a
direction or a magnitude. The inverter's own energy-management algorithm still decides how to operate
within those caps.

| What you want | Register | Replaceable? |
|---|---|---|
| Dynamic export curtailment (e.g. negative prices) | 0x1106 Active Power Export Limit | **Yes** - fully replaces `FeedIn: Maximum Power` / `FeedIn: Limitation Mode`, without EEPROM writes |
| Cap grid draw / peak shaving | 0x1107 Active Power Import Limit | **Yes** - new capability |
| Ramp rate for limit changes | 0x110A Active Power Change Rate | **Yes** |
| Reactive power / power factor | 0x1108 / 0x1109 | **Yes** |
| Desired grid power setpoint (`Passive: Desired Grid Power`) | - | **No** |
| Battery charge/discharge window (`Passive: Minimum/Maximum Battery Power`) | - | **No** |

There is no register in the RWV block that sets a grid power target or a battery power window. Forcing
the battery to charge from the grid still requires **Passive mode** (0x1187-0x118C) or TOU/timed
charging. The passive-mode entities are deliberately left untouched by this feature and remain the
supported fallback: if the RWV path misbehaves on your inverter, set `Remote: Power Control Mode` to
`Disabled` and carry on using `Passive: Update Battery Charge/Discharge` exactly as before.

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

0x110C (`SVG Fixed Reactive Power`) is documented as "Not used, readable" and is exposed as a read-back
sensor only.

The **Remote Power** feature, which adds the heartbeat and the fail-safe on top:

| Entity | Purpose |
|---|---|
| `Remote: Power Control Mode` | Builds the 0x1105 enable bitmask: `Disabled`, `Active Power Control` (bit0), `Active + Reactive Power` (bit0+bit1), `Active + Reactive (Power Factor)` (bit0+bit1+bit2) |
| `Remote: Export Power Limit` | Export cap in **W**. `-1` means "not used, take the percent entity". `0` is a valid hard-zero export limit |
| `Remote: Import Power Limit` | Import cap in **W**, same `-1` convention |
| `Remote: Export Limit Percent` | Export cap in **%** of rated power, used when the W entity is `-1` |
| `Remote: Import Limit Percent` | Import cap in **%** of rated power |
| `Remote: Power Limit Change Rate` | 0x110A value, only sent in `Full` write mode (disabled by default) |
| `Remote: Autorepeat Duration` | How long the limits stay applied, in seconds |
| `Remote: Update Power Limits` | Button - applies the values and opens the autorepeat window |
| `Remote: Autorepeat Remaining` | Seconds left before the fail-safe release |
| `Remote: Power Control Write Mode` | `Short (0x1105-0x1107)` or `Full (0x1105-0x110C)` (disabled by default, see below) |

The W entities are converted to the register's 0.1 % units using the `Inverter power rating` sensor
(0x06ED), so they give the full 0.1 % resolution the register supports. The percent entities are whole
percent. If the inverter power rating is unavailable, the W entity is ignored and the percent entity is
used instead (a warning is logged).

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
   what `Remote: Power Control Mode` does.
2. **0x0900 `Remote Config` (installer level).** The protocol states: *"Only when the enable bit is
   turned on can the function of the corresponding register be reflected."* Bit0 is the active
   down-load enable. Read the `Remote Config` entity first; changing it is an installer-level setting.
3. **Confirm the inverter accepted control** via the `Derating Status` (0x0477) sensor - bit 7 reads
   *"Remote active control"* - and `Remote Control Status` (0x0478).

## Short vs Full write mode

The protocol note for this block reads: *"When writing, the first address is fixed to any address within
this range, and the length is the length of the range"*, which is ambiguous about whether a partial write
is accepted. The default `Short` mode writes only 0x1105-0x1107 (three registers), which is the least
intrusive. If your inverter rejects that write (Modbus exception, or the read-back values never change),
enable the `Remote: Power Control Write Mode` entity and switch it to `Full`: the whole 0x1105-0x110C
block is then written in one operation, with 0x1108-0x110C re-sent from their current read-back values so
your reactive-power and power-factor settings are preserved unchanged.

## Automation example

```yaml
# Zero export during negative price hours, wear-free
- alias: Sofar zero export on negative price
  triggers:
    - trigger: numeric_state
      entity_id: sensor.electricity_price
      below: 0
  actions:
    - action: select.select_option
      target:
        entity_id: select.solax_remote_power_control_mode
      data:
        option: Active Power Control
    - action: number.set_value
      target:
        entity_id: number.solax_remote_export_power_limit
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
and press the button again.
