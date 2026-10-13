# Chargy

A Home Assistant integration that plans when a home battery should charge from
the grid and when it may discharge, using electricity prices, a PV forecast and
the household's learned consumption.

## Language

### Time and horizons

**Slot**: A five-minute interval of time, always labelled and entered in the
local timezone.

**Neighbor slot**: A slot a whole number of five-minute steps before or after
another slot, crossing midnight like any other time.

**Day**: A span of exactly 24 hours: its start is always 24 hours before its
end, regardless of daylight saving time.

**History window**: The configured number of days ending now, today so far
included, over which consumption averages, their deviation, the battery's
efficiency and the standby power estimate are measured. _Avoid_: Window
(unqualified), Training window (Shady's term; it excludes the reference day)

**Baseline window**: The configured number of days ending now over which the
consumption baseline is measured. _Avoid_: Window (unqualified)

**Price horizon**: How far ahead the price provider has published prices.
_Avoid_: Horizon (unqualified)

**PV forecast horizon**: How far ahead the PV forecast reaches. _Avoid_: Horizon
(unqualified), forecast horizon

**Planning horizon**: The slots from now up to the earlier of the price horizon
and the PV forecast horizon: the slots for which every input exists. _Avoid_:
Horizon (unqualified)

### Battery and BMS

**BMS**: The battery system's own controller as exposed through Home Assistant:
the one device Chargy reads state from, and which regulates its own output.
_Avoid_: Inverter, storage, the system

**Battery**: The energy store of the battery system, with its state of charge,
capacity and stored energy. _Avoid_: BMS (for the energy store)

**State of charge (SoC)**: The battery's stored energy as a percentage of its
capacity.

**Max SoC**: The highest SoC the battery is charged to. A limit of the BMS, not
something Chargy decides.

**Min SoC**: The lowest SoC the battery is discharged to when covering load. A
limit of the BMS, not something Chargy decides.

**Soft target SoC**: The SoC, set by the user and never above Max SoC, up to
which Chargy plans grid charging. PV may still charge the battery beyond it.
_Avoid_: Soft target, Target SoC, target margin, SoC bounds

**Capacity**: The energy the battery holds at 100 % SoC, as the BMS reports it.
It can change over time. _Avoid_: Max capacity

**Stored energy**: The energy the battery currently holds: SoC times capacity,
including any energy below Min SoC.

**Max charge power**: The highest power the battery can be charged with. A limit
of the BMS that Chargy treats as fixed.

**Max discharge power**: The highest power the battery can be discharged with. A
limit of the BMS that Chargy treats as fixed.

**Standby power**: The power the battery system itself draws to keep running, in
every mode and even below Min SoC. _Avoid_: Standby drain

**Round-trip efficiency**: The share of the energy put into the battery that is
later delivered out of it, measured from the installation's own history.
_Avoid_: Efficiency (unqualified)

**Idle run**: A stretch of consecutive slots in which the BMS neither draws from
nor feeds the grid on any phase and the SoC never rises. The battery's drain
during an idle run is standby power.

**SoC valley**: A low point of the SoC inside a valley search window, used as a
measurement anchor for the round-trip efficiency.

**Valley search window**: A 24-hour span searched for an SoC valley. _Avoid_:
Window (unqualified)

### Consumption and the grid

**Consumption**: The household's gross electrical load. Never the BMS's own grid
flows. _Avoid_: Grid consumed

**Grid meter reading**: The signed net power at the house's grid connection,
import positive. _Avoid_: Shelly reading

**Grid draw**: Power the BMS takes from the grid. _Avoid_: Grid consumed

**Grid feed**: Power the BMS gives to the grid. _Avoid_: Grid fed, export

**Phase**: One of the three mains phases of the house, labelled L1, L2 and L3.
_Avoid_: Phase A/B/C, phase 1/2/3

### Consumption profile

**Baseline**: A phase's idle load floor: the mean of the lowest cluster of its
consumption readings over the baseline window. The whole-house baseline is the
sum over the phases.

**Bin**: One fixed-width wattage range of the histogram from which the baseline
is found. _Avoid_: Bucket

**Falloff region**: The stretch of bins after a candidate peak of the histogram
in which the counts keep declining. _Avoid_: Valley region

**Smoothing radius**: How many neighbor slots on each side of a slot are blended
into it. _Avoid_: Smoothing window

**Bucket**: One weekday-and-slot cell of the consumption profile, per phase or
combined for the whole house. It is keyed by local weekday and local time of
day. _Avoid_: Bin (for a histogram range)

**Expected consumption**: A slot's weighted mean consumption for its weekday and
time of day, for the whole house. In holiday mode it is the baseline times the
holiday baseline factor.

**Confidence**: How stable a bucket's consumption is: 1 / (1 + relative
variability).

**Consumption band**: A slot's expected consumption with a higher and a lower
figure around it. The margin shrinks as confidence rises.

**Scenario**: Which figure of the consumption band a simulation assumes: high,
mean or low.

### Holiday mode

**Holiday switch**: The user's on/off input that turns holiday mode on and off.

**Holiday mode**: The state in which the holiday switch is on. Expected
consumption then follows the baseline instead of the profile.

**Holiday flag**: A slot's mark of holiday or not: holiday if the holiday switch
was on at any moment within it.

**Holiday baseline factor**: The multiplier applied to the whole-house baseline
in holiday mode. _Avoid_: Baseline buffer

### Energy balance

**Trajectory**: The slot-by-slot course of stored energy over the planning
horizon, with spill and shortfall per slot.

**Shortfall**: Load the battery cannot cover in a discharge slot, counted as
battery-side energy. Load imported in a hold slot or charge slot is planned, not
a shortfall.

**Gap**: The grid-side energy needed to remove every shortfall over the planning
horizon, assuming higher consumption and before any charging is planned. A
requirement, not a promise. The user-facing sensor labels it "grid energy
needed".

**Unmet gap**: The part of the gap the charge planner could not place.

**Spill**: PV energy that cannot be stored because the battery is full. _Avoid_:
Overflow, curtailment

**Spill-relief slot**: A discharge slot added outside the discharge-eligible
tiers solely to make room for PV that would otherwise be spill. _Avoid_:
PV-overflow extra

### Prices and tiers

**Price provider**: The component that reads one tariff's prices from Home
Assistant and returns a price curve. Exactly one is active. _Avoid_: Tariff (for
the provider)

**Price curve**: The price per slot over the price horizon, in €/kWh after the
price unit factor is applied.

**Price unit factor**: The multiplier that converts a provider's native price
unit into €/kWh: 1 for €/kWh, 1/100 for cent/kWh. Chosen by the user from a
dropdown.

**Tier**: A ranked class of prices; rank 0 is the cheapest.

**Charge-eligible tier**: One of the lowest N tiers, where N is set by the user.
The highest tier never is. _Avoid_: Cheap tier

**Discharge-eligible tier**: One of the highest M tiers, where M is set by the
user. The lowest tier never is. _Avoid_: Expensive tier, expensive window

**Tier strategy**: A method that classifies a price curve into tiers.

**Tier span**: A contiguous span of time in which one tier applies. _Avoid_:
Tier window

**Tier timeline**: The ordered sequence of tier spans over the price horizon.
_Avoid_: Tier plan

**Percentile window**: The span of prices a rolling tier strategy classifies
against, with a backward and a forward length. _Avoid_: Rolling window, window
(unqualified)

### Planning

**Slot mode**: What the plan allows in a slot: charge, discharge or hold.

**Charge slot**: A slot in which grid charging is planned.

**Discharge slot**: A slot in which the battery is allowed to cover the load, up
to its maximum discharge power. How much it delivers is the BMS's decision.

**Hold slot**: A slot in which the battery neither charges from the grid nor
discharges. PV surplus still charges it.

**Unassigned mode**: The slot mode of slots no planner has claimed: discharge
until a discharge planner has run, hold afterwards.

**Block**: A run of consecutive slots with the same slot mode in the plan, such
as a charge block or a discharge block. _Avoid_: Charge window, discharge
window, window (for a run of slots)

**Claim**: A planner fixing a slot's mode so that no later planner changes it.

**Plan**: The slot modes, with the grid energy for each charge slot, once the
planners have run. _Avoid_: Schedule

**Planner order**: Whether the discharge planner (_discharge first_, the
default) or the charge planner (_charge first_) runs first.

**Replan**: Re-deriving the gap and the plan from the current inputs. _Avoid_:
Recompute (Shady's term for something else)

**Charge planner**: The planner that claims charge slots in charge-eligible
tiers to cover the gap.

**Discharge planner**: The planner that claims discharge slots in
discharge-eligible tiers, plus spill-relief slots.
