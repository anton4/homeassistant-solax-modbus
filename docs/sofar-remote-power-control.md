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
| Trim an export you are **actively commanding** through passive mode, on a fast signal, with a ramp | 0x1106 + 0x110A | **Yes** - measured 1:1 against the commanded part, and the only wear-free way to do it |
| Cap grid draw / peak shaving | 0x1107 Active Power Import Limit | Untested as a feature - but see the warning below, a low value here changes behaviour dramatically |
| Reactive power / power factor | 0x1108 / 0x1109 | Untested |
| Reduce export when you are *not* commanding any | 0x1106 | **No** - measured: a 2 kW cap with 4.8 kW of output. See below |
| **Zero feed-in when PV exceeds load plus what the battery accepts** | - | **No** - use `FeedIn: Maximum Power` (0x1024) |
| Desired grid power setpoint (`Passive: Desired Grid Power`) | - | **No** |
| Battery charge/discharge window (`Passive: Minimum/Maximum Battery Power`) | - | **No** |

There is no register in the RWV block that sets a grid power target or a battery power window. Forcing
the battery to charge from the grid still requires **Passive mode** (0x1187-0x118C) or TOU/timed
charging. The passive-mode entities are deliberately left untouched by this feature and remain the
supported fallback: if the RWV path misbehaves on your inverter, set `Remote: Power Control Mode` to
`Disabled` and carry on using `Passive: Update Battery Charge/Discharge` exactly as before.

### What 0x1106 caps: commanded export, not total output

This is the model that fits every measurement across five hardware sessions, and it is not what the
register's name suggests. Split your export into two parts:

- **Involuntary export** - PV the inverter cannot store or consume: `PV - house load - battery charge
  acceptance`. 0x1106 has **no authority over this at all**.
- **Commanded export** - the part passive mode asks for via `Passive: Desired Grid Power`. 0x1106 caps
  *this*.

```
grid export  =  (PV - house load - battery charge acceptance)  +  min(commanded export, 0x1106 limit)
```

With passive mode commanding -15500 W, the limit tracked beautifully - involuntary residual 3.1 kW, then a
2 kW limit gave 4.85 kW of export, 4 kW gave 7.3 kW, 6 kW gave 8.7 kW. With `Desired Grid Power` = 0,
nothing is commanded, and the limit does nothing whatsoever. Read-back-verified, so there is no doubt about
what the inverter was holding:

| stored 0x1106 | PV | battery | BMS charge limit | AC output | grid |
|---|---|---|---|---|---|
| 100 % | 16.4 kW | +12.8 kW | 12.74 kW | 3.6 kW | -2.2 kW |
| 50 % | 16.2 | +12.5 | 12.54 | 3.7 | -2.3 |
| 10 % (= 2 kW cap) | 15.7 | +10.9 | 10.90 | **4.8 kW** | -3.4 |
| 0 % | 15.9 | +10.5 | 11.80 | **5.4 kW** | -4.1 |

A 2 kW cap with 4.8 kW of output, and a 0 % cap with 5.4 kW - because all of that output was involuntary.

**So this block cannot give you zero export.** For that you need the **anti-reflux** function, which the
protocol describes against the grid connection point - *"VDE4105 safety regulation grid-connected power
limit"*, measured at the PCC (0x0488). That is `FeedIn: Limitation Mode` (0x1023) plus `FeedIn: Maximum
Power` (0x1024) and the `FeedIn: Update` button. Those are plain `RW`, so use them for state changes, not as
a setpoint you rewrite every minute.

### Watch the battery's charge limit, not just its power

The involuntary residual moves with what the battery will *accept*, which is not a constant. On the test
system the BMS-reported charge limit tapered from **15.7 kW at 92 % SOC to 10.3 kW at 96 %**, and the
battery charged at exactly that value whenever nothing interfered - within 100 W. Every kilowatt the BMS
withdraws appears at the grid instead. If your inverter exposes a max-charge-power sensor, put it on the
same chart as PV, battery power and grid power; without it, the residual export looks inexplicable.

### Engaging the block can make things worse, and not because of the export limit

*Measured, and the most important warning here.* A later session started from a clean baseline - passive
`Desired Grid Power` = 0, PV 20 kW, the battery absorbing 18 kW, house load 2 kW, grid **+120 W, i.e. zero
export**. `Remote: Export Limit Percent` was set to 100 (a limit that restricts nothing) and the button
pressed. **48 seconds later**, with nothing else touched, the inverter ramped its output from 1.9 kW to
~6.5 kW at the configured ramp rate and began exporting **5 kW**. Battery charging fell from 18.3 kW to
12.3 kW - the inverter gave up charging and pushed the surplus to the grid.

The export limit had nothing to do with it. Stepping it 50 % -> 25 % -> 1 % -> 0 % left export at
-4.8 to -5.3 kW, unchanged. Setting it back to **100 %** did not stop it either. Only
`Remote: Power Control Mode` = `Disabled` recovered the baseline, within ~20 s.

**The cause was the import limit** (0x1107), which had been left at **4.2 %** from an earlier test.
Confirmed by repeating the identical press with 0x1107 at 100 %: export stayed at its pre-press value for
the full two and a half minutes, no ramp, no jump. So:

- **Leave `Remote: Import Limit Percent` at 100** unless you are deliberately testing an import cap. Check
  `Remote: Applied Import Limit Percent`, or the `0x1107=` field of `Remote: Register Read-back`, before
  every test - a stale import limit will invalidate everything you conclude about the export limit.
- If you already have zero export from passive `Desired Grid Power` = 0, engaging this block may well be a
  step backwards on this firmware. Verify on your own unit before automating it.

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
| `Remote: Register Read-back` | What the inverter itself holds in 0x1105-0x110C, e.g. `0x1105=1 0x1106=10.0% 0x1107=100.0% ...` |
| `Remote: Read-back Matches` | `yes` / `no: 0x1106 holds 100.0 %, sent 10.0 %` / `pending read-back` / `unknown` |
| `Remote: Full-Scale Ramp Time` | Seconds the stored 0x110A rate needs for a 0 -> 100 % swing |
| `Remote: Ramp Rate` | The same rate in W/s (needs a rated power) |
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
   what `Remote: Power Control Mode` does, and it is the only enable that was needed on the test unit -
   verified by read-back: bit0 latches when the button is pressed and clears again on `Disabled`.
2. **0x0900 `Remote Config` (installer level).** The protocol's *"Only when the enable bit is turned on
   can the function of the corresponding register be reflected"* note belongs to the 0x09xx block -
   0x0900 bit0 gates 0x0901/0x0902, the installer-level twins of 0x1106/0x110A - not to 0x110x. It is
   unknown whether it also gates this block; on the test unit the limits worked without touching it. The
   `Remote Config` number is disabled by default; enable it in the entity registry to read the value.
3. **Don't trust the status words to confirm anything; use the read-back.** `Derating Status` (0x0477) bit 7
   reads *"Remote active control"* and `Remote Control Status` (0x0478) reports the reactive/PF equivalents,
   but on the HYD 20KTL-3PH 0x0477 reads 0 even while a limit is demonstrably in force - as does 0x06ED.
   `Derating Enable Status` (0x047C, disabled by default) carries the corresponding *enable* flags and is
   likely to behave the same way. `Remote: Register Read-back` and `Remote: Read-back Matches` are the
   reliable confirmation: on that unit they showed the whole block storing exactly what was written
   (0x1105 bit0 latched, 0x1106 following 100 -> 50 -> 25 -> 1 -> 0 -> 100 %) in `Short` write mode.

## Troubleshooting: the limit does nothing

Work down this list in order - each entity rules out one layer.

1. `Remote: Autorepeat Remaining` - if it is 0 the window has closed and nothing is being written. Press
   `Remote: Update Power Limits`.
2. `Remote: Limit Source` - tells you whether the W or the percent entity is in effect. Changes to the
   *other* one are ignored by design.
3. `Remote: Applied Export Limit` / `... Percent` - what the integration computed and sent. If this is not
   what you expect, the problem is on this side, and the source entity above says why.
4. `Remote: Read-back Matches` - whether the inverter actually **stored** it. `no: 0x1105 holds 0, sent 1`
   means the enable bits are not latching, so the limits cannot apply; `no: 0x1106 holds ...` means the
   write is being rejected or overwritten - try `Short` write mode (see below). `pending read-back` is
   normal for up to one scan interval after a change; `unknown` means the button has never been pressed or
   the block read is failing.
   While you are there, read the **`0x1107=`** field of `Remote: Register Read-back` too. A stale import
   limit changes the inverter's behaviour on its own, and it will make the export limit look guilty of
   things it did not do (see "Engaging the block can make things worse" above).
5. `Remote: Full-Scale Ramp Time` - how long the inverter takes to walk to a new limit. A stored 0x110A of
   10 means **nine minutes** for a 90-point change, which looks exactly like "nothing happened".
6. `Remote: Control Conflict` - another controller is commanding the inverter and the two are being
   arbitrated. Release it (see above).
7. If all of the above are clean and the export still will not fall: check that the surplus has somewhere
   to go. `Passive: Maximum Battery Power` = 0 blocks battery charging entirely, and since the inverter
   does not curtail PV, the un-storable surplus goes to the grid regardless of any limit. Measured: with
   that at 0 and a 0 % export limit, a 17.7 kW array exported 15.6 kW.

## Ramp rate

*Measured.* A limit change is followed as a ramp - a change from 100 % to 10 % took roughly 35 seconds on
the test unit (~500 W/s on a 20 kW machine). That rate is 0x110A. The protocol lists its unit as a plain
`%`, but every sibling rate register is explicitly `%Pn/min` (0x0902 `ActiveOutputDownSpeed`, 0x0906,
0x0914, 0x0915, 0x0917), and only that reading fits the measurement - 35 s for a 90-point step implies a
stored value near 150. On a 20 kW inverter:

| stored 0x110A | rate | time for a 100 % -> 10 % change |
|---|---|---|
| 1 | 3.3 W/s | 90 min |
| 10 | 33 W/s | **9 min** |
| 100 | 333 W/s | 54 s |
| 150 | 500 W/s | 36 s |

`Remote: Full-Scale Ramp Time` and `Remote: Ramp Rate` show the stored value in both forms - check them
first whenever a limit change appears to do nothing. The default `Short` write does not send 0x110A, so the
inverter keeps whatever it holds; write it once with the direct `Active Power Change Rate` number if you
need a different rate. It is deliberately not part of the Short write, because the write is contiguous and
including 0x110A would also command reactive power and power factor as a side effect of every curtailment
write. Note that selecting `Full` write mode *does* send it, from the `Remote: Power Limit Change Rate`
local whose default is 100 - which can be slower than what your inverter already had.

## Short vs Full write mode

The protocol note for this block reads: *"When writing, the first address is fixed to any address within
this range, and the length is the length of the range"*, which is ambiguous about whether a partial write
is accepted. The default `Short` mode writes only 0x1105-0x1107 (three registers), which is the least
intrusive and is **confirmed accepted on the HYD 20KTL-3PH** - every register read back exactly what was
written.

**`Full` is rejected outright on the HYD 20KTL-3PH.** Read-back proof: after selecting it, an export limit
of 1 % left 0x1106 sitting at the previous 10 %, and the subsequent release left 0x1105 at 1 - i.e. the
inverter refused an eight-register write at 0x1105 and kept refusing every write after it. `Full` re-sends
0x110C, which the protocol itself calls *"not used, readable"*, which is the likely reason.

**Keep `Short` unless you have proven it is rejected** - i.e. `Remote: Read-back Matches` reports `no:` for
0x1106 while `Short` is selected. Note that the hub does not check Modbus write responses, so a refused
write produces no log line of its own: **`Remote: Read-back Matches` is the only place it shows up.**

The plugin now protects itself, so a rejected `Full` cannot strand you:

- After **two consecutive read-backs** that disagree with what was sent, it logs an error and starts sending
  the three-register `Short` payload instead. `Remote: Read-back Matches` appends
  `(Short fallback: Full was rejected)` while that is in force. Selecting `Short` yourself clears the state,
  so re-selecting `Full` later gets a fresh chance.
- The autorepeat **release** only uses `Full` if a `Full` write has been *seen to land* on your inverter.
  Otherwise it goes out as `Short`. The release is the one write that must succeed - if it is rejected, the
  limits stay applied with no heartbeat left to retry them.

**If you are stuck with a limit applied** (`Remote: Register Read-back` shows `0x1105=1` and a limit you
cannot clear): set `Remote: Power Control Write Mode` to `Short`, press `Remote: Update Power Limits`, and
confirm the read-back reads `0x1105=0 0x1106=100.0% 0x1107=100.0%`. Failing that, the direct
`Active Power Export Limit` number and the `Power Control (bitmask)` number (disabled by default) write
single registers through a different code path and can be used by hand.

What `Full` does when it works: writes the whole 0x1105-0x110C block in one operation, with 0x1108-0x110C
re-sent from their current read-back values so your reactive-power and power-factor settings are preserved
unchanged. It also sends 0x110A from `Remote: Power Limit Change Rate`, which can be slower than the rate
your inverter already holds. If any of the read-backs is unavailable, the write falls back to Short rather
than commanding reactive power 0 and power factor 0.

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
- **A 0 % export limit perversely increases export.** Reproducible in two sessions: at exactly 0 % the
  inverter charged 1.3-1.4 kW *below* the BMS charge limit and exported that instead, where 10 % and 50 %
  left charging at the limit. Cause unknown; avoid 0 % and use 1 % if you need a near-zero commanded cap.
- Whether 0x1105 bit0 latches, whether `Short` writes are accepted, and whether `Full` is accepted are all
  **answered** by read-back on a HYD 20KTL-3PH: yes, yes, and no respectively.
- Untested, one observation each: `FeedIn: Maximum Power` = 0 with mode `Enabled - 3-phase limit` did not
  appear to stop a ~3 kW involuntary export either, but the feed-in mode and the passive battery window were
  both changed during that window, so it needs a clean test of its own. The 0x1107 import limit has never
  been exercised as an actual peak-shaving feature - only observed wrecking an export test.
- Whether 0x0900 bit0 gates this block (prerequisite 2).
- Which models populate 0x06ED, 0x0477/0x0478 and 0x047C. On the HYD 20KTL-3PH they read 0.
- Unrelated but adjacent: peak shaving 0x1132/0x1133 are true grid-side sell/buy caps in Watts, but only
  in peak-shaving mode, and the integration writes 0x1132 alone although the protocol note demands first
  address 0x1130 with length 4. Anti-reflux mode 3 ("Phase power mode") is not offered by the select.
