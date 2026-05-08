# Bearded Trading BB-SD

An advanced, state-gated institutional supply and demand trading strategy written in Pine Script v6 for TradingView. This engine uses a sequential integer state machine to identify high-probability zone retests while remaining entirely immune to trend-expansion noise, consolidation spam, and visual desynchronization artifacts.

---

## Technical Architecture Overview

Unlike basic binary indicators (Active / Dead), this engine structures its parallel matrix data array around a four-part progression. A zone cannot trade simply because price sits inside it; it must systematically transition through each gate by satisfying precise structural candlestick conditions.

[State 0: Freshly Born] ──► (Price Detaches Cleanly) ──► [State 1: Armed & Ready]
│
[State 3: Fired / Locked] ◄── (Tiered Confluence Match) ◄── [State 2: Touch Detected]
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
* **The Overlap Prevention Filter:** Before structural allocation, the new zone is processed through the `checkOverlap()` layout grid scan. If the top and bottom coordinates fall completely within the boundaries of an already existing active box, it is classified as a duplicate and discarded.

#### B. Risk Management & Visual Unit
Upon breakout confirmation, risk management metrics are generated:
* **Stop Loss Line Placement:** Rendered as a fixed 5-bar line anchor.
    * **Demand SL:** Placed below the box floor line ($bBot - \text{tickBuffer}$).
    * **Supply SL:** Placed above the box top ceiling line ($bTop + \text{tickBuffer}$).
    * **Adaptive Protection (`useDynamicSL`):** The engine scans the historical swing landscape back to the previous zone origin to snap protection paths beyond key structural swing pivots.
* **Take Profit Line Placement:**
    * **Adaptive Target (`useDynamicTP`):** The engine scans a cluster of candles around zone creation to identify structural extremes and anchors the `tpBuffer` beyond that pivot.
    * **Static Target:** Uses the manual `tpBuffer` value relative to the zone boundaries.

---

### 🛡️ State 1: Detached & Armed (The Anti-Spam Shield)

A zone enters State 1 when price action explicitly breaks away and separates from the zone coordinates. This acts as a protective shield ensuring the engine ignores follow-through momentum bars on the initial breakout trend expansion run.

#### A. Transition Prerequisites
* **Demand Zone Arming:** The zone switches to State 1 only when a candle close or a candle low prints completely above the zone top line (`close > bTop` or `low > bTop`).
* **Supply Zone Arming:** The zone switches to State 1 only when a candle close or a candle high prints completely below the zone bottom line (`close < bBot` or `high < bBot`).

---

### 🏹 State 2: Touch Detected (Retest Validation & Pullback Tracking)

State 2 is achieved when the market experiences a true counter-trend pullback, bringing price action back into the active historical boundaries.

#### A. RETEST Criteria Rules
* **Demand Retest Boundary:** While in State 1, a candle wick successfully penetrates the active zone boundaries (`low <= bTop` and `high >= bBot`).
* **Supply Retest Boundary:** While in State 1, a candle wick successfully penetrates the active zone boundaries (`high >= bBot` and `low <= bTop`).

---

### 🔒 State 3: Fired & Locked (Tiered Confluence Matrix)

State 3 represents trade confirmation. Upon a confirmed reversal candle body (`close > open` for longs, `close < open` for shorts), the engine evaluates a tiered confluence scoring system.

| Signal Tier | Requirements | Visual Style |
| :--- | :--- | :--- |
| **A++ (Elite Momentum)** | **BB Touch/Pierce** within lookback **AND** **EMA Slope** agrees with trade direction | Solid Vivid Hue |
| **A+ (Premium Rejection)** | **BB Touch/Pierce** within lookback (represents volatility rejection) | Intermediate Shade |
| **Standard** | Core Retest Engine fired without a recent Bollinger Band touch | Lighter Shade |

#### A. Confluence Filters & Exclusion Rules
* **BB Volatility Rejection:** For A+/A++ tiers, the price must have touched or pierced the respective Bollinger Band within the `bbLookback` window.
* **EMA Trend Alignment (A++):** For the Elite Momentum rating, the slope of the EMA basis line must align with the trade (sloping up for BUY, sloping down for SELL).
* **Opposing Band Exclusion:** Any setup that has touched the **opposing** Bollinger Band within the lookback window is disqualified from A+/A++ status to avoid trading into over-extension.

#### B. Execution & Graphic Output
* **Real-Time Visuals:** The strategy uses `calc_on_every_tick=true` to ensure Bollinger Bands and EMA basis lines update live on the forming candle.
* **Confirmed Execution:** To prevent premature signals during tick updates, orders (`strategy.entry`) are gated by `barstate.isconfirmed`, ensuring execution only occurs upon candle close.
* **Broker Execution:** Executes a `strategy.entry()` order using the static y-positions of the original anchor lines.
* **Signal Marker Mapping:** Prints bold triangle labels with tiered suffixes (e.g., BUY A++).
* **The Infinite Retest Loop Reset:** After a signal fires, the zone becomes inactive (State 3). It refuses to fire again until price completely leaves the level, resetting the sequence back to **State 1**.

---

## 🛑 Global State Exceptions: Zone Invalidation & Garbage Collection

Active zones are evaluated on every bar close. If a zone violates baseline parameters, it is subjected to an immediate Dynamic Garbage Collection Routine.

### Invalidation Triggers
* **Raw Boundary Breach (Hard Breach):** A candle body explicitly closes beyond the defensive wall line (`close < bBot` for Demand; `close > bTop` for Supply).
* **Proximity Limit Breach:** The current market price drifts too far away from the zone boundaries, exceeding the `proximityPct` threshold.

---

## Core Matrix Arrays Reference

| Array Variable Name | Functional Application |
| :--- | :--- |
| `demandBoxes` / `supplyBoxes` | Visual zone block coordinates rendering |
| `demandTradeLabels` / `supplyTradeLabels` | Bolded Tiered execution indicators (A++, A+, BUY/SELL) |
| `demandState` / `supplyState` | Controls the 4-stage anti-spam engine gates |
| `demandSLLines` / `supplySLLines` | Visual Stop Loss fixed 5-bar target markers |
| `demandTPLines` / `supplyTPLines` | Visual Take Profit target markers |

---

## License

This project is proprietary and intended for private trading terminal application. Modifications and logic extractions should preserve array alignment parameters to prevent pointer leakage errors in TradingView runtime processors.