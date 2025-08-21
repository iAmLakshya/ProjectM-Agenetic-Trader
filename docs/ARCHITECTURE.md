# How ProjectM Works

This document explains the internal workings of the trading bot.

## The Main Loop

Everything centres around a repeating cycle. Each iteration:

```
┌─────────────────────────────────────────────────┐
│                                                 │
│   Fetch State → Build Context → Query LLM →     │
│   Validate → Execute → Log → Sleep → Repeat    │
│                                                 │
└─────────────────────────────────────────────────┘
```

The interval between cycles is configurable via command line arguments.

## Data Sources

**From Hyperliquid:**
- Current positions and their PnL
- Open orders waiting to fill
- Recent trade executions
- Live mid prices for each asset

**From TAAPI:**
- Moving averages (EMA at various periods)
- Momentum indicators (RSI, MACD)
- Volatility measures (ATR, Bollinger width)
- Any other indicator the LLM requests

## Decision Making

The LLM receives a structured prompt containing all gathered data. It responds with JSON specifying:

- Which assets to trade
- Direction (long, short, or flat)
- Position size as percentage of capital
- Stop loss and take profit levels
- Reasoning behind the decision

If the JSON is malformed, a secondary model attempts to fix it before giving up.

## Safety Checks

Before any order goes to the exchange:

1. **Margin check** ensures sufficient collateral exists
2. **Leverage check** confirms we stay within allowed limits
3. **Size validation** prevents obviously wrong position sizes

Orders that fail these checks are blocked and logged.

## State Reconciliation

The bot periodically compares what it thinks is happening versus what Hyperliquid reports. If they disagree, Hyperliquid wins. This catches edge cases like:

- Orders that filled while we were processing
- Positions closed by stop losses
- Network issues causing missed updates

## Logging and Debugging

Two log streams capture everything:

**Trade Diary (JSONL format)**
Each entry records the decision, execution result, and current portfolio state. Useful for performance analysis.

**LLM Logs**
Full prompts and responses from every model call. Useful for understanding why the bot made specific decisions.

Both are accessible via HTTP while the bot runs.

## Key Design Choices

**Why Hyperliquid?**
Decentralised, low fees, good API. Perpetuals allow both long and short positions.

**Why OpenRouter?**
Single API for multiple LLM providers. Easy to switch models without code changes.

**Why TAAPI?**
Reliable indicator calculations. Saves implementing TA logic ourselves.

**Why JSON responses?**
Structured output is easier to validate and parse than free text. The schema is enforced strictly.

## Error Handling

Network failures trigger automatic retries with backoff. Repeated failures pause trading until the next cycle. Nothing crashes the main loop except explicit shutdown signals.
