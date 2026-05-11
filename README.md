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
* **Base Candle Identification:** * **Demand Base Candidate:** Any bearish candle close (`close < open`). The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$). [cite: 59]
    * **Supply Base Candidate:** Any bullish candle close (`close > open`). [cite_start]The engine caches its `bar_index`, `high` ($bTop$), and `low` ($bBot$). [cite: 59]
* **Breakout Confirmation:** A candidate transitions from a cached state to an official active zone box when a subsequent candle aggressively breaches the base extremes:
    * [cite_start]**Demand Zone:** A candle high breaks strictly above base high (`high > potentialDemandBaseHigh`). [cite: 59]
    * [cite_start]**Supply Zone:** A candle low breaks strictly below base low (`low < potentialSupplyBaseLow`). [cite: 59]
* **Structure Preservation:** Every confirmed structural breakout creates a unique zone. [cite_start]The engine permits overlapping zones to persist, ensuring historical support and resistance levels remain visible even if they reside within newer structures. [cite: 59]

#### B. Risk Management & Visual Unit
[cite_start]Risk management price levels are no longer anchored to the zone discovery phase; instead, the calculation engine waits for the signal candle to define the trade's defensive and objective boundaries. [cite: 59]

---

### 🛡️ State 1: Detached & Armed (The Anti-Spam Shield)

[cite_start]A zone enters State 1 immediately upon discovery and serves as the primary gating mechanism for retest validation. [cite: 59]

#### A. Transition Prerequisites
* [cite_start]**Demand Zone Arming:** The zone is armed and ready for a retest once it is successfully drawn on the chart. [cite: 59]
* [cite_start]**Supply Zone Arming:** The zone is armed and ready for a retest once it is successfully drawn on the chart. [cite: 59]

---

### 🏹 State 2: Touch Detected (Retest Validation & Early Warning)

[cite_start]State 2 is achieved when the market pulls back into the active historical boundaries with specific candlestick characteristics. [cite: 59]

#### A. RETEST Criteria Rules
* **Correct Color Wick Requirement:**
    * [cite_start]**Demand Retest:** Strictly requires a **Bearish (Red)** candle wick to penetrate the zone boundaries. [cite: 59]
    * [cite_start]**Supply Retest:** Strictly requires a **Bullish (Green)** candle wick to penetrate the zone boundaries. [cite: 59]
* [cite_start]**No-Body-Close Restriction:** If the retesting candle body closes **inside** the zone boundaries, the retest is disqualified. [cite: 59] [cite_start]The zone remains on the chart and stays **Orange**, requiring a fresh, valid wick rejection to re-arm. [cite: 59]
* [cite_start]**Early Warning Alerts:** The strategy issues a real-time alert upon the close of a valid retest candle. [cite: 59] [cite_start]This allows the user to prepare for a potential entry before the reversal signal is confirmed. [cite: 59]

---

### 🔒 State 3: Fired & Locked (Tiered Confluence Matrix)

[cite_start]State 3 represents trade confirmation and execution. [cite: 59]

#### A. Signal Execution Gates
* [cite_start]**Exit Confirmation:** A BUY or SELL signal fires only after a valid retest has occurred AND a subsequent reversal candle (Green for BUY, Red for SELL) closes **completely outside** the zone boundaries. [cite: 59]
* [cite_start]**Signal Replacement logic:** If a zone is retested multiple times, the engine automatically deletes the previous signal objects (Labels, SL/TP lines) and anchors the new signal to the most recent valid retest candle. [cite: 59]

| Signal Tier | Requirements | Visual Style |
| :--- | :--- | :--- |
| **A++ (Elite Momentum)** | **BB Touch/Pierce** within lookback **AND** **EMA Slope** agrees with trade direction | [cite_start]Solid Vivid Hue [cite: 59] |
| **A+ (Premium Rejection)** | **BB Touch/Pierce** within lookback (represents volatility rejection) | [cite_start]Intermediate Shade [cite: 59] |
| **Standard** | Core Retest Engine fired without a recent Bollinger Band touch | [cite_start]Lighter Shade [cite: 59] |

#### B. Execution & Graphic Output
* [cite_start]**Real-Time Visuals:** The strategy uses `calc_on_every_tick=true` for real-time EMA and Bollinger Band tracking. [cite: 59]
* [cite_start]**Dynamic Risk Anchoring:** * **Stop Loss (SL):** Calculated relative to the **Signal Candle Low** (Demand) or **High** (Supply) plus the `tickBuffer`. [cite: 59]
    * [cite_start]**Take Profit (TP):** Calculated relative to the **Signal Candle High** (Demand) or **Low** (Supply) plus the `tpBuffer`. [cite: 59]
* [cite_start]**Visual Transitions:** Zone borders remain **Orange** during the retest phase and only transition to **Green** (Demand) or **Red** (Supply) once an entry signal is officially confirmed. [cite: 59]

---

## 🛑 Global State Exceptions: Zone Invalidation & Garbage Collection

Active zones are evaluated on every bar close. [cite_start]If a zone violates baseline parameters, it is subjected to an immediate Dynamic Garbage Collection Routine. [cite: 59]

### Invalidation Triggers
* [cite_start]**Hard Breach:** A candle body explicitly closes beyond the defensive outer wall line of the zone. [cite: 59]
* [cite_start]**Proximity Limit Breach:** The current market price drifts too far away, exceeding the `proximityPct` threshold. [cite: 59]

---

## Core Matrix Arrays Reference

| Array Variable Name | Functional Application |
| :--- | :--- |
| `demandBoxes` / `supplyBoxes` | [cite_start]Visual zone block coordinates rendering [cite: 59] |
| `demandTradeLabels` / `supplyTradeLabels` | [cite_start]Tiered execution indicators (A++, A+, BUY/SELL) [cite: 59] |
| `demandState` / `supplyState` | [cite_start]Controls the sequential signal engine gates [cite: 59] |
| `demandSLLines` / `supplySLLines` | [cite_start]Signal-anchored Stop Loss markers (updated to most recent retest) [cite: 59] |
| `demandTPLines` / `supplyTPLines` | [cite_start]Signal-anchored Take Profit markers (updated to most recent retest) [cite: 59] |

---

## License

This project is proprietary and intended for private trading terminal application. Modifications and logic extractions should preserve array alignment parameters to prevent pointer leakage errors in TradingView runtime processors.