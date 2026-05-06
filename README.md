# Bearded Trading Supply/Demand Engine (v6)

An advanced, state-gated institutional supply and demand trading strategy written in Pine Script v6 for TradingView. This engine uses a multi-layered integer state machine to identify high-probability zone retests while remaining entirely immune to trend-expansion noise, consolidation spam, and visual desynchronization artifacts.

---

## Technical Architecture Overview

Unlike basic binary indicators (Active / Dead), this engine maps supply and demand structures using a synchronized parallel array matrix paired with a **4-Stage Gating Detachment Matrix**. This architectural framework ensures that trades are only signaled when price action cleanly breaks away from a zone and returns on a validated pullback.

[State 0: Freshly Born] ──► (Price Detaches Cleanly) ──► [State 1: Armed & Ready]
│
[State 3: Fired / Locked] ◄── (Bull/Bear Candle Reversal) ◄── [State 2: Touch Detected]
│
└──► (Price Low/High Detaches Completely Again) ──► Resets to State 1

---

## Structural Engine Rules

### 1. Zone Creation Mechanics (The Base-and-Breakout Engine)
Zones are dynamically initialized using a two-phase structural candle relationship:
* **Base Candle Identification:** * **Demand Candidate:** Any bearish candle close (`close < open`). The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$).
  * **Supply Candidate:** Any bullish candle close (`close > open`). The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$).
* **Breakout Trigger Evaluation:** A cached base candle candidate is converted into an active zone box when a subsequent candle breaks outside the base extremes:
  * **Demand Zone Confirmation:** Triggered when a subsequent candle's high breaks strictly above the base high (`high > potentialDemandBaseHigh`).
  * **Supply Zone Confirmation:** Triggered when a subsequent candle's low breaks strictly below the base low (`low < potentialSupplyBaseLow`).
* **Overlap Prevention Filter:** Before array storage, the proposed zone must clear the `checkOverlap()` layout grid scan. If the coordinates fall completely within the boundaries of an already existing active box, it is classified as a duplicate transaction and discarded instantly to preserve chart clarity.

### 2. Risk Management Visuals (Static Anchor Lines)
To preserve visual performance and eliminate tracking lag, stop loss and take profit targets are rendered **exactly once** at the zone’s birth:
* **Stop Loss Line Placement:** Fixed at zone birth extending exactly 5 bars from origin.
  * **Demand SL:** Projected below the box bottom floor line ($bBot - \text{tickBuffer}$).
  * **Supply SL:** Projected above the box top ceiling line ($bTop + \text{tickBuffer}$).
  * **Adaptive Protection (`useDynamicSL`):** When enabled, the engine automatically scans historical structural swing pivots back to the previous zone origin to snap protection lines beyond major market extremes.
* **Take Profit Line Placement:** Calculated dynamically based on the configuration of your target inputs (`tpBuffer`).
* *Note: These visual line paths are never stretched, dragged, or duplicated by loops. They act as constant, unchangeable boundaries for order execution references.*

### 3. Multi-Retest State Matrix
To allow infinite independent retests without creating overlapping execution markers or double-trigger loops, the zone cycle maps through four distinct integer states:
* **State 0: Freshly Born (Locked):** Zone is dormant. It cannot trade. This blocks breakout follow-through bars from triggering false multi-signals during momentum expansions.
* **State 1: Detached & Armed:** Armed when price explicitly breaks away and separates from the zone bounds. 
  * **Demand Reset:** `close > bTop` or `low > bTop`
  * **Supply Reset:** `close < bBot` or `high < bBot`
* **State 2: Pullback Touch Detected:** True retest condition met when a candle wick successfully penetrates the active zone coordinates (`low <= bTop` and `high >= bBot`).
* **State 3: Signal Fired & Frozen:** Triggered when a confirmed reversal candle body closes inside or emerging out of State 2 (`close > open` for longs, `close < open` for shorts). The zone instantly locks into State 3 to freeze out consecutive consolidation ticks. It will refuse to fire again until price completely leaves the level, resetting the sequence back to **State 1**.

### 4. Zone Invalidation & Garbage Collection
Active zones are evaluated on every bar close. A zone is permanently deleted if it encounters:
* **A Hard Breach:** A candle body explicitly closes beyond the structural zone wall (`close < bBot` for Demand; `close > bTop` for Supply).
* **A Proximity Breach:** The current market price drifts past the boundaries defined by `proximityPct`.

When triggered, a complete array garbage collection cycle occurs synchronously:

IF (Close cuts through Floor/Ceiling) OR (Price Drifts Outside Proximity Limits)
└──► 1. Wipe Graphic Box Object from Chart Memory
└──► 2. Wipe Graphic SL/TP Line Anchors from Chart Memory
└──► 3. Erase Associated BUY/SELL Signal Label Pointers
└──► 4. Cleanly Splice and Column-Shift Core Matrix Arrays

---

## Alert Dispatch Routing System

Alert notifications are evaluated and processed strictly on the close of each candle bar using the `alert.freq_once_per_bar_close` parameter.

### System Alert Discrepancy Matrix
A common technical discrepancy in advanced algorithmic scripts occurs when an alert is fired, but no signal marker appears on the chart interface. This behavior is intentional within this framework and is driven by two design mechanics:

1. **The Retroactive Historical Purge:** If a long-term zone catches a valid retest, it immediately dispatches an execution alert to your webhooks or terminal and prints a visual triangle marker. However, if price continues to slide against that setup on subsequent bars, forcing a hard invalidation close, the garbage collection routine immediately triggers. It deletes the box, the lines, and **wipes that triangle marker from chart history**. Your terminal correctly received the alert while the zone was structurally alive, but the chart interface cleaned itself up retrospectively when the zone ultimately failed.
2. **Consolidated Label Overwrite Gating:** To prevent your interface from stacking multiple overlapping triangle labels during tight retest cycles, the script maintains precisely **one** active trade label slot pointer per array index. When a subsequent valid retest occurs, the engine fires a new alert notification but runs a memory sweep (`if not na(tLbl) label.delete(tLbl)`), replacing the older visual marker with the newest signal. You will receive multiple real-time alerts over time, but the chart interface will keep itself visually optimized by displaying only the most recent retest vector.

---

## Core Matrix Arrays

Data integrity is handled across a multi-array column grid tracker. When an index position modifications occurs, it is applied simultaneously down the column row list to maintain strict 1:1 row alignment:

| Array Variable Name | Data Type Stored | Functional Application |
| :--- | :--- | :--- |
| `demandBoxes` / `supplyBoxes` | `box` object pointer | Visual zone block coordinates rendering |
| `demandCreationBar` / `supplyCreationBar` | `int` value | Marks the initial breakout baseline anchor point |
| `demandTradeLabels` / `supplyTradeLabels` | `label` object pointer | Visual execution arrow indicators |
| `demandState` / `supplyState` | `int` coordinate | Controls the 4-stage anti-spam engine gates |
| `demandMinLow` / `supplyMaxHigh` | `float` currency value | Tracks extreme historical wick parameters since birth |
| `demandTouched` / `supplyTouched` | `bool` flag | Registers initial boundary zone validation touches |
| `demandSLLines` / `supplySLLines` | `line` object pointer | Visual Stop Loss fixed 5-bar target markers |
| `demandTPLines` / `supplyTPLines` | `line` object pointer | Visual Take Profit fixed 5-bar target markers |

---

## License

This project is proprietary and intended for private trading terminal application. Modifications and logic extractions should preserve array alignment parameters to prevent pointer leakage errors in TradingView runtime processors.