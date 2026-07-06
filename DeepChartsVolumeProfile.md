# DeepChartsVolumeProfile

A session Volume Profile indicator for NinjaTrader 8. One profile is drawn automatically for every
session in the loaded chart history, anchored directly over that session's own candles - not a single
fixed panel. Its signature mode splits each price row down the middle: real Bid/Ask **Delta** on the
left half, **Total Volume** on the right half (Deepchart's "Delta and Total Volume" mode).

## Requirements

- **Order Flow Volumetric data**: the indicator sources real Bid/Ask volume via NinjaTrader's built-in
  `AddVolumetric()` (not the up/down tick-rule approximation). This requires your data feed to actually
  support Bid/Ask volumetric data.
- **Tick Replay**: for the real Bid/Ask volume to reconstruct correctly on **historical** bars, enable
  Tick Replay:
  - Tools → Options → Market Data → check **Show Tick Replay**
  - On the chart's Data Series settings, enable **Tick Replay**
- If your feed can't supply this, the indicator prints a warning to the NinjaScript Output window
  ("a serie volumetrica nao pode ser carregada...") and won't draw anything.

## Installation

Paste `DeepChartsVolumeProfile.cs` into NinjaTrader's NinjaScript Editor (or place it in
`Documents\NinjaTrader 8\bin\Custom\Indicators\`) and compile with F5. Make sure no other script also
defines the global-namespace enums `DeltaTotalVPDisplayMode` / `LevelExtensionMode` (e.g. an old copy
under a previous name) - NinjaTrader compiles every script in the folder together, and duplicate
definitions will fail with CS0101.

## How it works

- **One profile per session.** "Session" is whatever time window you configure (Session Start/End Time
  + Time Zone) - e.g. RTH 9:30-16:00. Every occurrence of that window in the loaded chart history gets
  its own profile, positioned starting exactly at that session's first bar.
- **Real Bid/Ask, not tick-rule.** Volume per price level comes from NinjaTrader's native Order Flow
  Volumetric bars (`GetTotalVolumeForPrice` / `GetDeltaForPrice`), which classify trades against the
  actual best bid/ask at the time, not just an up/down-tick heuristic.
- **Live/developing profile.** The session currently in progress is drawn too, updating tick by tick,
  merging in the still-forming Volumetric bar on top of the already-committed ones.
- **Gaps are excluded automatically.** Only ticks that fall inside your configured session window are
  ever accumulated - time outside it (e.g. overnight, non-RTH hours) is never counted, whether you're
  looking at a single session or a manually merged multi-day one.

## Properties

### 1. Session

| Property | Default | Description |
|---|---|---|
| Session Start Time | 09:30 | Start of the session used to build each day's profile |
| Session End Time | 16:00 | End of the session - that day's profile freezes here |
| Time Zone | Local | Time zone the Start/End times above are expressed in (Local, EST, CST, MST, PST, UTC, GMT, CET, EET, JST, AEST) |

A session that crosses midnight (Start > End, e.g. 18:00 → 03:00) is supported.

### 2. Volumetric Data

| Property | Default | Description |
|---|---|---|
| Volumetric Bar Size (ticks) | 500 | Size, in ticks, of the underlying Order Flow Volumetric bars used to source Bid/Ask volume. Bigger = fewer bar objects to process (cheaper), smaller = finer intraday granularity for the live/developing profile. Doesn't affect the profile's own row height. |
| Ticks Per Row | 4 | How many ticks are grouped into a single profile row (price bucket). 1 = finest resolution. |

### 3. Display

| Property | Default | Description |
|---|---|---|
| Display Mode | Delta and Total | `Total` = single-sided bar, full width, classic profile look. `Delta` = single-sided bar sized by net buy/sell imbalance. `Delta and Total` = split row: Delta grows left from the session's first bar, Total grows right from the same point. |
| Max Width % of Session | 100 | How far the highest-volume (POC) row reaches across the session's own bar width. 100 = the POC row fills the whole session; lower values shrink everything proportionally so the profile doesn't fully cover the candles. |
| Row Opacity % | 85 | Opacity of the Delta/Total fill rectangles (not the POC/VAH/VAL/High/Low outline boxes, which are always fully opaque). |
| Show POC | on | Outlines the row with the single highest total volume. |
| Extend POC | None | `None` = outline only, bounded to the session. `Naked Extension` = a ray continues rightward past the session and freezes the instant price first touches that price *after* the session ends. `Infinity Extend` = the ray always reaches the right edge of the chart, ignoring touches. |
| Show Session High | on | Outlines the session's highest traded row. |
| Extend Session High | None | Same three modes as Extend POC, applied to the session high. |
| Show Session Low | on | Outlines the session's lowest traded row. |
| Extend Session Low | None | Same three modes as Extend POC, applied to the session low. |

### 4. Value Area

| Property | Default | Description |
|---|---|---|
| Value Area % | 70 | Percentage of the session's total volume that must fall inside the Value Area, expanding outward from the POC row (same algorithm as a standard VAH/VAL calculation). |
| Show VAH | on | Outlines the Value Area High row. |
| Extend VAH | None | Same three extension modes, applied to VAH. |
| Show VAL | on | Outlines the Value Area Low row. |
| Extend VAL | None | Same three extension modes, applied to VAL. |
| Shade Value Area | on | Fills rows between VAL and VAH using the separate "Value Area Fill Color" instead of the regular Total Volume color, so the value area is visually distinguishable from the rest of the profile. |
| Value Area Opacity % | 85 | Opacity of that shaded fill, independent from the regular Row Opacity. |

### 5. Colors

All colors are standard color pickers; each has a paired `...Serializable` property behind the
scenes so your chosen colors persist across saves/reloads (nothing you need to touch directly).

| Property | Default | Used for |
|---|---|---|
| Delta Buy Color | Lime Green | Delta rows/bars where net delta ≥ 0 (net buying) |
| Delta Sell Color | Crimson | Delta rows/bars where net delta < 0 (net selling) |
| Total Volume Color | Dodger Blue | Total volume bars **outside** the Value Area |
| Value Area Fill Color | Medium Purple | Total volume bars **inside** the Value Area (only when "Shade Value Area" is on) |
| POC Color | Yellow | POC outline box + POC extension ray |
| VAH Color | Deep Sky Blue | VAH outline box + VAH extension ray |
| VAL Color | Deep Sky Blue | VAL outline box + VAL extension ray |
| Session High Color | White | Session High outline box + extension ray |
| Session Low Color | White | Session Low outline box + extension ray |
| Merge Selection Color | Orange | Border drawn around whichever session is currently selected via Ctrl+Click |

### 6. Performance

| Property | Default | Description |
|---|---|---|
| Max Sessions Kept | 10 | Older sessions beyond this count stop being rendered and are dropped from memory (oldest first). Keep this reasonable on charts loaded with a lot of history. |

## Merging sessions (mouse only, no keyboard)

You can combine adjacent sessions into one profile - useful for looking at, say, two consecutive days
as a single combined range.

- **Ctrl+Click** a session's profile to select it (gets an orange outline - the Merge Selection Color).
- **Ctrl+Click a different, immediately adjacent session** while one is selected → the two merge into
  one profile, anchored at the earlier session's first bar and reaching to the later session's last bar.
  The combined volume is a straight sum - whatever gap sits between the two sessions was never
  accumulated in the first place, so it's automatically excluded.
- **Ctrl+Click empty space**, or **Ctrl+Click the selected session again** (when it's *not* a merged
  result) → deselects.
- **Ctrl+Click an already-merged profile again** → splits it back into its two immediate constituents.
  This works at any depth: merge A+B, then merge that result with C, and clicking the combined profile
  once splits it into `(A+B)` and `C`; the `(A+B)` piece stays selected, so clicking it again splits it
  into `A` and `B`. There's no "undo last action only" limitation - every merge remembers its own two
  original pieces, so you can always split back down to individual sessions.
- Clicking a session that **isn't adjacent** to the one selected just moves the selection there instead
  of merging (merging only ever makes sense between neighbors, since anything else would silently skip
  over data in between).
- Only finalized (completed) sessions can be selected or merged - the still-in-progress current session
  is excluded, since merging into data that's still changing wouldn't make sense.
- Merges only live in memory for the current chart session - closing/reloading the chart resets them.

## Troubleshooting

- **Nothing draws at all**: check the NinjaScript Output window for the Bid/Ask warning message - your
  feed or Tick Replay setup likely isn't supplying real Bid/Ask volumetric data.
- **Profile looks frozen / doesn't move when scrolling**: shouldn't happen with this version - if it
  does, you may be running an old cached copy; recompile and make sure only one copy of the script
  exists in your NinjaScript folder.
- **Ctrl+Click does nothing**: make sure you're clicking directly within a *finalized* session's bar
  range (not today's still-forming session, and not empty chart space to the right/left of any candles).
