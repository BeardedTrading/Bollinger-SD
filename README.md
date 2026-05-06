# Bearded Trading Supply/Demand Engine (v6)

An advanced, state-gated institutional supply and demand trading strategy written in Pine Script v6 for TradingView. This engine uses a sequential integer state machine to identify high-probability zone retests while remaining entirely immune to trend-expansion noise, consolidation spam, and visual desynchronization artifacts.

---

## Technical Architecture Overview

Unlike basic binary indicators (Active / Dead), this engine structures its parallel matrix data array around a four-part progression. A zone cannot trade simply because price sits inside it; it must systematically transition through each gate by satisfying precise structural candlestick conditions.

[State 0: Freshly Born] ──► (Price Detaches Cleanly) ──► [State 1: Armed & Ready]
│
[State 3: Fired / Locked] ◄── (Bull/Bear Candle Reversal) ◄── [State 2: Touch Detected]
│
└──► (Price Low/High Detaches Completely Again) ──► Resets to State 1

---

## The Gated Lifecycle Specifications

### 📊 State 0: Freshly Born (Zone Discovery & Initialization)

Every zone begins its existence in State 0. It serves as an unconfirmed structure and is strictly forbidden from evaluating entries or firing signals.

#### A. Zone Discovery & Filtering Rules
* **Base Candle Identification:** * **Demand Base Candidate:** Any bearish candle close (`close < open`). The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$).
  * **Supply Base Candidate:** Any bullish candle close (`close > open`). The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$).
* **Breakout Confirmation:** A candidate transitions from a cached state to an official active zone box when a subsequent candle aggressively breaches the base extremes:
  * **Demand Zone:** A candle high breaks strictly above base high (`high > potentialDemandBaseHigh`).
  * **Supply Zone:** A candle low breaks strictly below base low (`low < potentialSupplyBaseLow`).
* **The Overlap Prevention Filter:** Before structural allocation, the new zone is processed through the `checkOverlap()` layout grid scan. If the top and bottom coordinates fall completely within the boundaries of an already existing active box in the array matrix, it is classified as a duplicate transaction and discarded instantly to preserve chart clarity.

#### B. Risk Management Instantiation
The moment the breakout occurs and the box is born, risk management metrics are generated **exactly once**:
* **Stop Loss Line Placement:** Rendered as a fixed 5-bar line anchor from the origin bar.
  * **Demand SL:** Placed below the box floor line ($bBot - \text{tickBuffer}$).
  * **Supply SL:** Placed above the box top ceiling line ($bTop + \text{tickBuffer}$).
  * **Adaptive Protection (`useDynamicSL`):** The engine automatically scans the historical swing landscape back to the previous zone origin to snap protection paths beyond key structural swing pivots.
* **Take Profit Line Placement:** Calculated dynamically based on the configuration of target inputs (`tpBuffer`).
* *Note: These visual line paths are never stretched, dragged, or duplicated by loops. They act as constant boundaries for order execution references.*

---

### 🛡️ State 1: Detached & Armed (The Anti-Spam Shield)

A zone enters State 1 when price action explicitly breaks away and separates from the zone coordinates. This acts as a protective shield ensuring the engine ignores follow-through momentum bars on the initial breakout trend expansion run.

#### A. Transition Prerequisites
* **Demand Zone Arming:** The zone switches from State 0 (or resets from State 3) to State 1 only when a candle close or a candle low prints completely above the zone top line (`close > bTop` or `low > bTop`).
* **Supply Zone Arming:** The zone switches from State 0 (or resets from State 3) to State 1 only when a candle close or a candle high prints completely below the zone bottom line (`close < bBot` or `high < bBot`).
* **Engine Status:** The zone is now "Armed and Ready". It begins monitoring live price feeds for pullback vectors.

---

### 🏹 State 2: Touch Detected (Retest Validation & Pullback Tracking)

State 2 is achieved when the market experiences a true counter-trend pullback, bringing price action back down or up into the active historical boundaries.

#### A. RETEST Criteria Rules
* **Demand Retest Boundary:** While in State 1, a candle wick successfully penetrates the active zone boundaries (`low <= bTop` and `high >= bBot`).
* **Supply Retest Boundary:** While in State 1, a candle wick successfully penetrates the active zone boundaries (`high >= bBot` and `low <= bTop`).
* **Visual Status:** The `demandTouched` / `supplyTouched` internal array indices flip to `true`.

---

### 🔒 State 3: Fired & Locked (Signal Generation & Gating Lockout)

State 3 represents trade confirmation and immediate risk isolation. 

#### A. Opportunity Reversal Requirements
While the zone is held in State 2, the engine looks for a structural candle body reversal on the bar close:
* **BUY Signal Formula:** The current candle must close green (`close > open`), its closing price must be above the zone top (`close > bTop`), and the prior bar's close must have successfully held above the zone floor (`close[1] > bBot`).
* **SELL Signal Formula:** The current candle must close red (`close < open`), its closing price must be below the zone bottom (`close < bBot`), and the prior bar's close must have successfully held below the zone ceiling (`close[1] < bTop`).

#### B. Execution & Graphic Output
Upon matching the formula, the strategy dispatches instructions instantly on the bar close:
* **Broker Execution:** Executes a `strategy.entry()` long or short order, using the static y-positions of the original anchor lines (`line.get_y1()`) to calculate the exact risk distribution.
* **Signal Marker Mapping:** Prints a clean blue **BUY** triangle or red **SELL** inverted triangle at the execution bar index.
* **Consolidated Label Overwrite Gating:** To prevent stacking overlapping markers during tight retest consolidation, the script maintains precisely **one** active trade label slot pointer per array index. When a new retest occurs, the engine deletes the *previous* visual triangle marker on the chart interface (`if not na(tLbl) label.delete(tLbl)`) and replaces it with the newest signal.

#### C. The Infinite Retest Loop Reset
After a signal fires, the zone switches directly to State 3. **It becomes completely inactive.** This freezes out consecutive consolidation bars while price sits inside the box. It will refuse to fire again until price completely leaves the level, resetting the sequence back to **State 1**.

---

## 🛑 Global State Exceptions: Zone Invalidation & Garbage Collection

Active zones are evaluated on every bar close across all states. If a zone violates hard baseline parameters, it is bypassed by the state machine and subjected to an immediate **Dynamic Garbage Collection Routine**.

### Invalidation Triggers
* **Raw Boundary Breach (Hard Breach):** A candle body explicitly closes beyond the defensive wall line (`close < bBot` for Demand; `close > bTop` for Supply).
* **Proximity Limit Breach:** The current market price drifts too far away from the zone boundaries, exceeding the threshold defined by the `proximityPct` parameter.

### The Cleanup Sequence
When an invalidation trigger is met, all parallel entries sitting at loop column index position `i` are synchronously deleted to maintain a 1:1 data matrix alignment:

[Invalidation Event]
│
├──► 1. box.delete(b) ───────► Wipes Graphic Box Object from Chart View
├──► 2. line.delete(slL) ────► Removes Stop Loss Anchor from Memory
├──► 3. line.delete(tpL) ────► Removes Take Profit Anchor from Memory
├──► 4. label.delete(tLbl) ──► Retroactive Historical Purge of Signals
└──► 5. array.remove() ──────► Slices and shifts core data matrix index columns

---

## 🔔 Alert Dispatch Framework

Alert notifications are evaluated and processed strictly on the close of each candle bar using the `alert.freq_once_per_bar_close` parameter.

### System Alert Discrepancy Matrix
A common technical discrepancy occurs when an execution alert fires to an external webhook/terminal, but no signal arrow appears on the chart interface. This behavior is intentional within this architecture and is driven by the **Retroactive Historical Purge**:

* If a zone catches a valid retest, it dispatches an execution alert to your webhooks and prints a visual triangle marker. 
* However, if price continues to slide against that setup on subsequent bars, forcing a hard invalidation close, the garbage collection routine immediately triggers. It deletes the box, the lines, and **wipes that triangle marker from chart history**. 
* Your terminal correctly received the alert while the zone was structurally alive, but the chart interface cleaned itself up retrospectively when the zone ultimately failed.

---

## Core Matrix Arrays Reference

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