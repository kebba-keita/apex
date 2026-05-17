"""
AI ANALYSIS & MARKET SENTIMENT GUIDE
"""

# ============================================================================
# MARKET SENTIMENT ANALYSIS
# ============================================================================

The AI Analysis Engine provides real-time market sentiment scores from multiple sources.

**Sentiment Range: -2 to +2**
- -2: Extremely bearish (strong sell signals)
- -1: Moderately bearish
- 0: Neutral (wait for clarity)
- +1: Moderately bullish
- +2: Extremely bullish (strong buy signals)

# ============================================================================
# DATA SOURCES & CALCULATIONS
# ============================================================================

## 1. TECHNICAL INDICATORS (40% Weight)

### RSI (Relative Strength Index)
```
Calculation: RSI = 100 - (100 / (1 + RS))
where RS = Average Gains / Average Losses (over 14 periods)

Interpretation:
- RSI < 30: Oversold → BUY signal (+1.0)
- RSI 30-70: Neutral → HOLD (0)
- RSI > 70: Overbought → SELL signal (-1.0)
```

**Python Code:**
```python
def calculate_rsi(prices, period=14):
    changes = [prices[i] - prices[i-1] for i in range(1, len(prices))]
    gains = sum(max(c, 0) for c in changes[-period:]) / period
    losses = -sum(min(c, 0) for c in changes[-period:]) / period
    
    if losses == 0:
        return 100
    rs = gains / losses
    return 100 - (100 / (1 + rs))
```

### MACD (Moving Average Convergence Divergence)
```
Calculation:
- MACD Line = EMA(12) - EMA(26)
- Signal Line = EMA(9) of MACD
- Histogram = MACD - Signal

Interpretation:
- Histogram > 0: Bullish → +0.5
- Histogram < 0: Bearish → -0.5
```

### Bollinger Bands
```
Calculation:
- Middle = SMA(20)
- Upper = Middle + (2 × Std Dev)
- Lower = Middle - (2 × Std Dev)
- Position = (Price - Lower) / (Upper - Lower)

Interpretation:
- Position < 0.2: Near lower band → BUY (+0.5)
- Position > 0.8: Near upper band → SELL (-0.5)
- 0.2-0.8: Normal movement → HOLD (0)
```

### Volume Analysis
```
Calculation:
- Volume Ratio = Current 5-bar volume / 20-bar average

Interpretation:
- If Volume Ratio > 1.5: High activity (amplifies signal)
- Applies 0.5x multiplier to sentiment
```

**Technical Sentiment Example:**
```python
from apex.ai.analysis import SentimentAnalyzer

analyzer = SentimentAnalyzer()

tech_sentiment = analyzer.analyze_technical({
    "rsi": 28,              # Oversold: +1.0
    "macd_histogram": 0.05, # Bullish: +0.5
    "bb_position": 0.15,    # Lower band: +0.5
    "volume_ratio": 1.8     # High volume: amplify by 0.5x
})

print(tech_sentiment)
# Output: {"sentiment": 1.3, "indicators": [...], "strength": 1.3}
```

---

## 2. ON-CHAIN METRICS (30% Weight)

On-chain data comes from blockchain analysis.

### Whale Movements
```
Data: Large wallet transactions (>$1M)

Interpretation:
- Net Flow = (Buying Volume - Selling Volume) / Total Volume
- Whale Buying > 1%: Accumulation → +1.0
- Whale Selling < -1%: Distribution → -1.0

Why It Matters:
- Whales have capital for large moves
- Their moves often precede price moves
```

**How to Get Data:**
- Glassnode API: `glassnode.com`
- On-chain module in trading bots
- Alert services: Whale Alert, Nansen

### Exchange Inflow/Outflow
```
Interpretation:
- High Exchange Inflow (>70%): Selling pressure → -0.5
- Low Exchange Inflow (<30%): Accumulation → +0.5
- Why: Money on exchange = preparing to sell

Calculation:
Exchange Inflow % = Coins moved to exchange / Daily volume
```

### Large Transaction Count
```
Interpretation:
- High activity (>10 large tx in 24h): → +0.3
- Shows accumulation or liquidation activity
- Indicates volatility opportunity
```

**On-Chain Analysis Example:**
```python
analyzer = SentimentAnalyzer()

onchain_sentiment = analyzer.analyze_onchain({
    "whale_net_flow": 1.5,           # Whales buying: +1.0
    "exchange_inflow": 0.25,         # Low inflow: +0.5
    "large_tx_in_24h": 15            # High activity: +0.3
})

print(onchain_sentiment)
# Output: {"sentiment": 1.8, "metrics": [...], "strength": 1.8}
```

---

## 3. SOCIAL SENTIMENT (30% Weight)

Track what people are saying and discussing.

### Twitter Sentiment
```
Measurement: NLP analysis of tweets
- Sentiment score 0-1
- 0.7+: Bullish tweets → +1.0
- 0.3-0.7: Mixed
- <0.3: Bearish tweets → -1.0

Sources:
- Track hashtags: #Bitcoin, #ETH, $BTC
- Influencer mentions
- News account mentions
```

### Fear & Greed Index
```
Range: 0-100
- 0-25: Extreme Fear → BUY signal (+1.0)
- 25-45: Fear
- 45-55: Neutral
- 55-75: Greed
- 75-100: Extreme Greed → SELL signal (-1.0)

Calculation: Weighted average of:
- Market momentum (25%)
- Stock market strength (25%)
- Social media buzz (15%)
- Market dominance (10%)
- Trends (10%)
- Volatility (15%)

Get it: alternative.me/fear-greed-index
```

### Mention Count
```
Measurement: How often mentioned
- 2-3x normal mentions: Trending → +0.3
- >3x normal mentions: Viral → +0.5
```

**Social Sentiment Example:**
```python
analyzer = SentimentAnalyzer()

social_sentiment = analyzer.analyze_social({
    "twitter_sentiment": 0.78,       # Bullish: +1.0
    "fear_greed_index": 22,          # Extreme fear: +1.0
    "mention_count_vs_avg": 2.8      # Viral: +0.5
})

print(social_sentiment)
# Output: {"sentiment": 1.5, "sources": [...], "strength": 1.5}
```

---

# ============================================================================
# COMBINED SENTIMENT CALCULATION
# ============================================================================

## Weighted Average
```
Combined Sentiment = (Tech * 0.40) + (OnChain * 0.30) + (Social * 0.30)

Example:
- Technical: +1.3 × 0.40 = +0.52
- On-Chain: +1.8 × 0.30 = +0.54
- Social: +1.5 × 0.30 = +0.45
- Combined = +1.51 → STRONG BUY
```

## Full Example
```python
from apex.ai.analysis import SignalGenerator

generator = SignalGenerator()

# Collect data
market_data = {
    "rsi": 25,
    "macd_histogram": 0.08,
    "bb_position": 0.1,
    "volume_ratio": 2.2
}

onchain_data = {
    "whale_net_flow": 2.5,
    "exchange_inflow": 0.2,
    "large_tx_in_24h": 18
}

social_data = {
    "twitter_sentiment": 0.82,
    "fear_greed_index": 18,
    "mention_count_vs_avg": 3.2
}

# Generate signal
signal = generator.generate_signal(
    symbol="BTCUSDT",
    market_data=market_data,
    onchain_data=onchain_data,
    social_data=social_data,
    entry_price=45000
)

print(signal.to_dict())
```

**Output:**
```json
{
    "symbol": "BTCUSDT",
    "signal": "strong_buy",
    "confidence": 0.95,
    "sentiment_score": 1.85,
    "risk_rating": "low",
    "entry_price": 45000,
    "stop_loss": 44100,
    "take_profit": 47250,
    "sources": ["technical", "onchain", "social"],
    "timestamp": "2024-01-15T10:30:00Z"
}
```

---

# ============================================================================
# WHALE TRACKING
# ============================================================================

## What to Monitor

### Large Purchases
```
Signal: Whale buying > $1M in 24h
Interpretation:
- Accumulation phase → Bullish
- Often precedes rallies by hours/days
- Check: How many whales? (1 whale = less reliable)
```

### Wallet Movements
```
Track:
- Exchanges to cold storage: Removing coins = Bullish
- Cold storage to exchange: Preparing to sell = Bearish
- Dormant wallets waking up: Major move coming
```

### Exchange Behavior
```
Inflow to exchange: Preparing to sell = Bearish
Outflow from exchange: Removing from sale = Bullish

Interpretation:
- If 80%+ exchange inflow: Heavy selling pressure
- If <20% exchange inflow: Strong accumulation
```

**Whale Tracking Example:**
```python
def track_whale_activity(whale_address, chain='ethereum'):
    """Track large wallet movements"""
    data = fetch_on_chain_data(whale_address)
    
    large_purchases = [tx for tx in data if tx['value'] > 1_000_000]
    recent_purchases = [tx for tx in large_purchases if is_recent(tx)]
    
    if len(recent_purchases) > 0:
        return {"status": "ACCUMULATING", "count": len(recent_purchases)}
    
    large_sales = [tx for tx in data if tx['value'] < -1_000_000]
    recent_sales = [tx for tx in large_sales if is_recent(tx)]
    
    if len(recent_sales) > 0:
        return {"status": "DISTRIBUTING", "count": len(recent_sales)}
    
    return {"status": "IDLE"}
```

---

# ============================================================================
# SENTIMENT-TO-SIGNAL CONVERSION
# ============================================================================

## Signal Types

| Sentiment | Signal | Confidence | Action |
|-----------|--------|------------|--------|
| +1.5 to +2.0 | STRONG BUY | 90% | Enter long |
| +0.5 to +1.5 | BUY | 70% | Enter long |
| -0.5 to +0.5 | NEUTRAL | 50% | Wait |
| -1.5 to -0.5 | SELL | 70% | Exit long |
| -2.0 to -1.5 | STRONG SELL | 90% | Exit + enter short |

## Risk Rating

| Combined Sentiment | Risk Rating | Position Size |
|--------------------|-------------|---------------|
| > +1.5 | Low | 2% portfolio |
| +0.5 to +1.5 | Medium | 1.5% portfolio |
| -0.5 to +0.5 | High | 1% portfolio |
| -1.5 to -0.5 | High | 1% portfolio |
| < -1.5 | Critical | 0.5% portfolio |

---

# ============================================================================
# REAL-TIME MONITORING
# ============================================================================

## Data Collection Frequency

| Data Type | Collection Frequency | Latency |
|-----------|---------------------|---------|
| Technical Indicators | Every 1 minute | Real-time |
| On-Chain Metrics | Every 5 minutes | Near real-time |
| Social Sentiment | Every 15 minutes | Slight delay |
| Fear & Greed Index | Every 1 hour | Daily update |

## Automation with Webhooks

```python
from apex.webhooks.listener import WebhookListener
from apex.ai.analysis import SignalGenerator

listener = WebhookListener()
generator = SignalGenerator()

@listener.register_signal_handler
async def analyze_and_alert(signal_data):
    """Automatically analyze and send alerts"""
    
    # Get latest data
    market = fetch_market_data(signal_data['symbol'])
    onchain = fetch_onchain_data(signal_data['symbol'])
    social = fetch_social_sentiment(signal_data['symbol'])
    
    # Generate AI signal
    ai_signal = generator.generate_signal(
        symbol=signal_data['symbol'],
        market_data=market,
        onchain_data=onchain,
        social_data=social,
        entry_price=market['price']
    )
    
    # Send alert if confidence > 75%
    if ai_signal.confidence > 0.75:
        await send_telegram_alert(ai_signal)
        await send_discord_alert(ai_signal)
```

---

# ============================================================================
# COMMON SIGNAL SCENARIOS
# ============================================================================

### Scenario 1: Bottom Fishing (RSI Oversold + Whale Buying)
```
Signals:
✓ RSI < 25 (extreme oversold)
✓ Whale net inflow > 2%
✓ Fear & Greed < 25 (extreme fear)
✓ High volume

Action: STRONG BUY with tight stop loss
Position: 2% of portfolio
SL: -2%, TP: +5-8%
```

### Scenario 2: Top Exhaustion (RSI Overbought + Whale Selling)
```
Signals:
✓ RSI > 75 (extreme overbought)
✓ Whale net outflow < -2%
✓ Fear & Greed > 75 (extreme greed)
✓ Declining volume

Action: STRONG SELL or reduce longs
Position: Scale out 50%
SL: +2%, TP: -5-8%
```

### Scenario 3: Breakout + Confirmation
```
Signals:
✓ Price above MA(50)
✓ Twitter sentiment bullish (>0.7)
✓ High exchange outflow (<30%)
✓ Volume spike

Action: BUY on confirmation
Position: 1.5% of portfolio
SL: MA(50) - 1%, TP: +5-10%
```

### Scenario 4: Uncertainty (Mixed Signals)
```
Signals:
✓ RSI neutral (40-60)
✓ MACD mixed
✓ Social sentiment split
✗ Whale activity unclear

Action: WAIT for clarity
Position: No entry
Wait for 2 confirmed signals
```

---

# ============================================================================
# BACKTESTING AI SIGNALS
# ============================================================================

## Key Metrics

```python
def backtest_signals(signals, prices, trades):
    """Evaluate AI signal quality"""
    
    correct = sum(1 for s in signals if s['actual_direction'] == s['predicted_direction'])
    win_rate = correct / len(signals)
    
    avg_confidence_winners = mean([s['confidence'] for s in winning_signals])
    avg_confidence_losers = mean([s['confidence'] for s in losing_signals])
    
    print(f"Win Rate: {win_rate:.1%}")
    print(f"Confidence of Winners: {avg_confidence_winners:.2f}")
    print(f"Confidence of Losers: {avg_confidence_losers:.2f}")
    print(f"Sharpe Ratio: {calculate_sharpe(returns):.2f}")
```

---

# ============================================================================
# INTEGRATION WITH TRADING BOT
# ============================================================================

```python
from apex.ai.analysis import SignalGenerator
from apex.automation.workflow import WorkflowEngine

# Setup
generator = SignalGenerator()
engine = WorkflowEngine()

# Register workflow
workflow = Workflow(
    name="AI Sentiment Trading",
    triggers=[AISignalTrigger(min_confidence=0.75)],
    conditions=[RiskValidationCondition()],
    actions=[ExecuteTradeAction(), TelegramAlertAction()]
)

engine.register_workflow(workflow)

# Main loop
async def trading_loop():
    while True:
        # Get data
        market = get_latest_market_data()
        onchain = get_onchain_metrics()
        social = get_social_sentiment()
        
        # Generate signal
        signal = generator.generate_signal(
            "BTCUSDT",
            market,
            onchain,
            social,
            market['price']
        )
        
        # Execute workflow
        if signal.confidence > 0.70:
            state = await engine.execute_workflow(
                "AI Sentiment Trading",
                signal.to_dict()
            )
            
            if state.status == WorkflowStatus.SUCCESS:
                print(f"✅ Trade executed: {signal.signal_type.value}")
        
        await asyncio.sleep(60)  # Check every minute
```

---

# ============================================================================
# DATA SOURCES & APIS
# ============================================================================

| Data Type | Provider | Cost | Latency |
|-----------|----------|------|---------|
| Technical | CCXT | Free | Real-time |
| On-Chain | Glassnode | $500/mo | 5 min |
| On-Chain | Santiment | $500/mo | Real-time |
| Social | LunarCrush | $300/mo | Real-time |
| Fear & Greed | Crypto F&G | Free | Daily |
| Whale Alerts | Whale Alert | Free/API | Real-time |

---

# ============================================================================
# BEST PRACTICES
# ============================================================================

✅ **DO:**
- Combine multiple data sources
- Update sentiment regularly (every 15 min)
- Use confidence scores to filter
- Backtest on historical data
- Monitor actual vs predicted signals
- Adjust weights based on performance

❌ **DON'T:**
- Overweight social sentiment alone
- Trade on one signal source
- Ignore risk management
- Neglect whale activity
- Trade during low liquidity
- Ignore market regimes (bull/bear/choppy)

---

# ============================================================================
# NEXT STEPS
# ============================================================================

1. Connect to at least 2 data sources
2. Implement sentiment calculation
3. Backtest on 6 months of data
4. Paper trade for 2 weeks
5. Start with small positions (0.5%)
6. Monitor accuracy daily
7. Adjust weights weekly
8. Scale up when >60% win rate

For implementation:
- See `apex/ai/analysis.py`
- See `apex/webhooks/listener.py` for real-time integration
- See `AUTOMATION_GUIDE.md` for deployment

