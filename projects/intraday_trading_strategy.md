---
layout: single
title: ""
permalink: /projects/intraday_trading_strategy/
author_profile: false
classes: wide-project-layout
mathjax: true
---

## Introduction & Motivation

The main drive behind this project was to implement an advanced quantitative trading system that combines academic research from <em>mathematics for finance: an introduction to financial engineering</em> by capinski  with practical intraday trading strategies. I was studying Polytechnic University of Turin research paper and discovered the BEST (BEst STock Finder) framework—a sophisticated weighted sequence mining algorithm for asset selection. Simultaneously, I learned about  statistical indicators used in high-frequency trading. This led me to develop an end-to-end system that bridges stock selection with intelligent execution through multiple statistical filters. (I colloborated with my friend who was also interested in this project).

The implementation of the whole project can be found here : [Github Repository](https://github.com/luka612-max/intraday_trading_strategy)

## Project Overview

Our implementation included the following:
- Phase 1: Stock Selection using Weighted Sequence Mining (W-SEQ) algorithm
- Phase 2: Execution Engine with multi-factor statistical indicators (VWAP, Kalman Filter, GARCH, Hurst Exponent, Supertrend)
- Four distinct trading strategies (Momentum Long/Short, Reversal Long/Short)
- Conflict resolution logic to filter contradictory signals
- Daily recommendation system with comprehensive backtesting across NSE India indices
- Modular Python architecture with pandas-based data pipeline
- CLI integration for parameter configuration

## Trading Strategies & Selection Methodology

### Phase 1: BEST Framework - Stock Selection

Following is the high-level architecture for stock selection using Weighted Sequence Mining:

```python
# Configuration Parameters
ACTIVE_STRATEGIES = ['MOMENTUM_LONG', 'MOMENTUM_SHORT', 'REVERSAL_LONG', 'REVERSAL_SHORT']
TRAINING_PERIOD_DAYS = 15
Q_VALUE = 3  # Top/bottom Q stocks as triggers
K_VALUE = 1  # Top K sequences to recommend
MIN_SUPPORT = 2  # Minimum sequence occurrences
```
The core innovation is treating daily stock returns as "weighted items" and mining for recurrent profitable sequences across consecutive trading days. The system identifies patterns like: "If Stock A rises today → Stock B is a likely candidate for tomorrow."

**Stock Selection Class and Mining Logic**

We used a functional programming approach with dictionary-based data structures for efficient sequence mining:

```python
def create_transaction_sequences(daily_returns: pd.DataFrame, transaction_mode: str) -> Dict[pd.Timestamp, Dict[str, float]]:
    """Creates daily transactions based on positive (transaction_mode='UP') or negative (transaction_mode='DOWN') returns."""
    if transaction_mode == 'UP':
        target_returns = daily_returns[daily_returns > 0]
    elif transaction_mode == 'DOWN':
        target_returns = daily_returns[daily_returns < 0].abs()
    else:
        return {}

    transactions = {}
    for date in target_returns.index:
        day_data = target_returns.loc[date].dropna().to_dict()
        if day_data: 
            transactions[date] = day_data
    return transactions
```

**Weighted Sequence Mining Algorithm**

The core mining function uses efficient pair-counting and profit aggregation:

```python
def mine_profitable_sequences(transactions: Dict, min_support: int) -> Dict[Tuple[str, str], float]:
    """Mines profitable 2-item sequences from historical transaction data."""
    item_counts = defaultdict(int)
    for date in transactions:
        for item in transactions[date]:
            item_counts[item] += 1

    frequent_items = {item for item, count in item_counts.items() if count >= min_support}
    if not frequent_items: 
        return {}

    pair_counts = defaultdict(int)
    sequence_returns = defaultdict(list)
    sorted_dates = sorted(transactions.keys())

    for i in range(len(sorted_dates) - 1):
        prev_day_date, curr_day_date = sorted_dates[i], sorted_dates[i+1]
        prev_day_items = set(transactions[prev_day_date].keys()).intersection(frequent_items)
        curr_day_items = set(transactions[curr_day_date].keys()).intersection(frequent_items)

        for item_a in prev_day_items:
            for item_b in curr_day_items:
                if item_a != item_b:
                    sequence = (item_a, item_b)
                    pair_counts[sequence] += 1
                    profit = min(transactions[prev_day_date][item_a], transactions[curr_day_date][item_b])
                    sequence_returns[sequence].append(profit)

    profitable_sequences = {}
    for seq, count in pair_counts.items():
        if count >= min_support:
            avg_profit = sum(sequence_returns[seq]) / len(sequence_returns[seq])
            profitable_sequences[seq] = avg_profit

    return profitable_sequences
```

**Four Trading Strategies**

We implemented four distinct strategies combining trigger modes and transaction modes:

```python
STRATEGY_CONFIGS = {
    'MOMENTUM_LONG':  {'trigger_mode': 'UP',   'transaction_mode': 'UP',   'signal': 'BUY'},
    'MOMENTUM_SHORT': {'trigger_mode': 'DOWN', 'transaction_mode': 'DOWN', 'signal': 'SHORT-SELL'},
    'REVERSAL_LONG':  {'trigger_mode': 'DOWN', 'transaction_mode': 'UP',   'signal': 'BUY'},
    'REVERSAL_SHORT': {'trigger_mode': 'UP',   'transaction_mode': 'DOWN', 'signal': 'SHORT-SELL'},
}
```

### Phase 2: Execution Engine & Statistical Indicators

Once the BEST framework selects a stock, the execution engine applies multiple statistical filters to time entries and manage risk:

**1. Volume Weighted Average Price (VWAP):**
Acts as the "Fair Value" anchor. Trades are initiated when price deviates significantly from the session VWAP. This prevents entering positions at extreme valuations.

**2. Kalman Filter (Slope Analysis):**
Unlike simple moving averages, the Kalman Filter adapts to market noise in real-time:

- Dynamic Smoothing: Continuously updates state estimates as new price data arrives
- Trend Detection: Kalman Slope identifies the rate of change in price
- Risk Management: Ensures we don't "catch a falling knife" during mean reversion trades

**3. GARCH (Generalized Autoregressive Conditional Heteroskedasticity):**
Forecasts current market volatility through conditional variance modeling:

- Volatility Clustering: Captures periods of high and low volatility
- Dynamic Risk: Widens or narrows standard deviation bands based on predicted volatility
- Adaptive Stop-Losses: Adjusts exit levels proportionally to market regime

**4. Hurst Exponent (H) - Market Regime Identification**
Determines whether the market is mean-reverting or trending:

- H < 0.5: Mean-reverting (Ideal for the strategy)
- H > 0.5: Trending/Persistent (Signal to stay out of mean-reversion trades)
- H = 0.5: Random walk

**5. Dynamic Stop Loss: Supertrend**
Provides volatility-adjusted trailing stop losses:

- Calculation: Uses Average True Range (ATR) with multiplier (standard 10, 3) to set dynamic support/resistance
- Trailing Exit:
Long Positions: Stop follows the green indicator line below price
Short Positions: Stop follows the red indicator line above price
- Trend Flip: If price crosses the Supertrend line, position is immediately liquidated to preserve capital

## Recommendation Generation & Conflict Resolution

The system generates recommendations by combining all four strategies and resolving conflicts:

```python
def generate_recommendations(triggers: Dict, profitable_sequences: Dict, k: int) -> Set[str]:
    """Generates stock recommendations based on triggers and mined sequences."""
    recommendations = set()
    if not triggers or not profitable_sequences: 
        return recommendations

    for trigger_stock in triggers.keys():
        relevant_sequences = {seq: profit for seq, profit in profitable_sequences.items() if seq[0] == trigger_stock}
        if not relevant_sequences: 
            continue

        sorted_sequences = sorted(relevant_sequences.items(), key=lambda item: item[1], reverse=True)
        top_k_sequences = sorted_sequences[:k]

        for seq, profit in top_k_sequences:
            recommendations.add(seq[1])

    return recommendations
```

## Conflict Resolution Logic

Conflicting signals are intelligently filtered out:

```python
buy_signals = raw_recs['BUY']
short_signals = raw_recs['SHORT-SELL']
conflicting_signals = buy_signals.intersection(short_signals)

actionable_buys = buy_signals - conflicting_signals
actionable_shorts = short_signals - conflicting_signals
```

## Daily Recommendation System

The main execution loop runs daily from August 6, 2024 to August 6, 2025:

```python
def run_daily_recommendation_system():
    """Main function to run the W-SEQ daily stock recommendation system."""
    print(f"--- Starting W-SEQ System with Active Strategies: {', '.join(ACTIVE_STRATEGIES)} ---")

    all_daily_returns = prepare_daily_returns(combined_df.copy())
    if all_daily_returns.empty: 
        return

    current_date = START_TRADING_DATE
    while current_date <= END_TRADING_DATE:
        if (current_date - timedelta(days=1)) not in all_daily_returns.index:
            current_date += timedelta(days=1)
            continue

        print(f"\n{'='*51}\n--- Generating Recommendations for: {current_date.strftime('%Y-%m-%d')} ---\n{'='*51}")

        raw_recs = defaultdict(set)
        for strategy_name in ACTIVE_STRATEGIES:
            recommendations = run_strategy(all_daily_returns, current_date, strategy_name)
            signal_type = STRATEGY_CONFIGS[strategy_name]['signal']
            raw_recs[signal_type].update(recommendations)

        # Conflict Resolution
        buy_signals = raw_recs['BUY']
        short_signals = raw_recs['SHORT-SELL']
        conflicting_signals = buy_signals.intersection(short_signals)

        actionable_buys = buy_signals - conflicting_signals
        actionable_shorts = short_signals - conflicting_signals

        # Display Results
        print("📈 BUY Recommendations:", sorted(list(actionable_buys)))
        print("📉 SHORT-SELL Recommendations:", sorted(list(actionable_shorts)))

        current_date += timedelta(days=1)
```

## Sample Output

The system generates daily actionable recommendations:

```python
===================================================
--- Generating Recommendations for: 2025-01-18 ---
===================================================

--- Running Strategy: MOMENTUM_LONG ---
--- Running Strategy: MOMENTUM_SHORT ---
--- Running Strategy: REVERSAL_LONG ---
--- Running Strategy: REVERSAL_SHORT ---

----------------- FINAL ACTIONABLE RECOMMENDATIONS -----------------
⚠️ Conflicting Signals Found (No Action Taken): AXISBANK
📈 BUY Recommendations:
   -> APOLLOHOSP
   -> BPCL
   -> ETERNAL
   -> LT
📉 SHORT-SELL Recommendations:
   -> GAIL
   -> HCLTECH
   -> IOC
   -> LTIM
--------------------------------------------------------------------
```

## Backtesting Results

The system was backtested on NSE India indices from August 6, 2024 to August 6, 2025:

- Trading days analyzed: ~250 business days
- Average daily signals: 4-8 per category (BUY/SHORT)
- Conflict resolution rate: ~15-20% of raw signals filtered
- Consistent recommendation generation across market regimes

## Conclusion and Learnings

This project was an excellent deep-dive into quantitative trading system design, combining academic research with practical execution challenges. I learned not only about weighted sequence mining and statistical indicators but also about:

- Risk management through conflict resolution
- Multi-factor execution models for entry timing
- Volatility regimes and market regime identification
- Data pipeline design for real-time trading systems
- Balancing model complexity with interpretability
The framework demonstrates that profitable trading patterns can be discovered through systematic sequence analysis while maintaining interpretability—a critical advantage over black-box machine learning approaches. Overall a very positive learning outcome :)