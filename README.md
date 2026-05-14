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
* **Structure Preservation:** Every confirmed structural breakout creates a unique zone. The engine permits overlapping zones to persist, ensuring historical support and resistance levels remain visible even if they reside within newer structures.

#### B. Risk Management & Visual Unit
Risk management price levels are no longer anchored to the zone discovery phase; instead, the calculation engine waits for the signal candle to define the trade's defensive and objective boundaries.

---

### 🛡️ State 1: Detached & Armed (The Anti-Spam Shield)

A zone enters State 1 immediately upon discovery and serves as the primary gating mechanism for retest validation.

#### A. Transition Prerequisites
* **Demand Zone Arming:** The zone is armed and ready for a retest once it is successfully drawn on the chart.
* **Supply Zone Arming:** The zone is armed and ready for a retest once it is successfully drawn on the chart.

---

### 🏹 State 2: Touch Detected (Retest Validation & Valid Wick Alerts)

State 2 is achieved when the market pulls back into the active historical boundaries with specific rejection characteristics.

#### A. RETEST Criteria Rules
* **Correct Color Wick Requirement:**
    * **Demand Retest:** Strictly requires a **Bearish (Red)** candle wick to penetrate the zone boundaries.
    * **Supply Retest:** Strictly requires a **Bullish (Green)** candle wick to penetrate the zone boundaries.
* **No-Body-Close Restriction:** If the retesting candle body closes **inside** the zone boundaries, the retest is disqualified. The zone remains on the chart and stays **Orange**, requiring a fresh, valid wick rejection to re-arm.
* **Retest Expiry:** The signal gate remains open for only **2 bars** following a valid wick touch. If a reversal confirmation does not occur within this window, the retest expires to ensure signals only trigger on immediate rejections.
* **Valid Wick Alerts:** The strategy issues a real-time notification upon the close of a valid wick retest (where the body remains outside). This serves as an early warning to prepare for a potential signal.

---

### 🔒 State 3: Fired & Locked (Tiered Confluence Matrix)

State 3 represents trade confirmation. The engine evaluates a tiered confluence scoring system based on market context and volatility.

| Signal Tier | Requirements | Visual Style |
| :--- | :--- | :--- |
| **A++ (Elite Momentum)** | **Triple EMA Stacked** (20>50>100 for Buys) **AND** **Bollinger Band** Touch/Pierce | Solid Vivid Hue |
| **A+ (Market Context)** | **Triple EMA Stacked** (Correct order for trade direction) | Intermediate Shade |
| **Standard** | Core Retest Engine fired without trend alignment or volatility rejection | Lighter Shade |

#### A. Signal Execution & Replacement
* **Exit Confirmation:** A BUY or SELL signal fires only after a valid retest has occurred AND a subsequent reversal candle (Green for BUY, Red for SELL) closes **completely outside** the zone boundaries.
* **Signal Replacement:** If a zone is retested multiple times, the engine automatically deletes previous signal objects (Labels, SL/TP lines) and anchors the new signal to the most recent valid rejection.

#### B. Execution & Graphic Output
* **Dynamic Risk Anchoring:** * **Stop Loss (SL):** Calculated relative to the **Signal Candle Low** (Demand) or **High** (Supply) plus the `tickBuffer`.
    * **Take Profit (TP):** Calculated relative to the **Signal Candle High** (Demand) or **Low** (Supply) plus the `tpBuffer`.
* **Visual Transitions:** Zone borders remain **Orange** during the retest phase and only transition to **Green** (Demand) or **Red** (Supply) once an entry signal is officially confirmed.

---

## 🛑 Global State Exceptions: Zone Invalidation & Garbage Collection

Active zones are evaluated on every bar close. If a zone violates baseline parameters, it is subjected to an immediate Dynamic Garbage Collection Routine.

### Invalidation Triggers
* **Hard Breach:** A candle body explicitly closes beyond the defensive outer wall line of the zone (`close < bBot` for Demand; `close > bTop` for Supply).
* **Proximity Limit Breach:** The current market price drifts too far away, exceeding the `proximityPct` threshold.

---

## Core Matrix Arrays Reference

| Array Variable Name | Functional Application |
| :--- | :--- |
| `demandBoxes` / `supplyBoxes` | Visual zone block coordinates rendering |
| `demandTradeLabels` / `supplyTradeLabels` | Tiered execution indicators (A++, A+, BUY/SELL) |
| `demandState` / `supplyState` | Controls the sequential signal engine gates |
| `demandSLLines` / `supplySLLines` | Signal-anchored Stop Loss markers |
| `demandTPLines` / `supplyTPLines` | Signal-anchored Take Profit markers |

---

## License

This project is proprietary and intended for private trading terminal application. Modifications and logic extractions should preserve array alignment parameters to prevent pointer leakage errors in TradingView runtime processors.