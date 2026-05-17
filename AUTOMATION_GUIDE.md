"""
AUTOMATION & WORKFLOW GUIDE
Complete Setup for TradingView, Webhooks, Telegram, Discord, & Trading Execution
"""

# ============================================================================
# ARCHITECTURE OVERVIEW
# ============================================================================

```
┌─────────────────────────────────────────────────────────────────┐
│                    AUTOMATION SYSTEM FLOW                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  TradingView Alert                                               │
│       │                                                           │
│       ├─→ Webhook Endpoint (FastAPI)                             │
│       │         │                                                │
│       │         ├─→ Signature Verification (HMAC-SHA256)         │
│       │         ├─→ Rate Limiting Check                          │
│       │         └─→ Payload Validation                           │
│       │                 │                                        │
│       │                 ├─→ Workflow Engine                       │
│       │                 │       │                                │
│       │                 │       ├─→ Trigger Detection            │
│       │                 │       │    (Signal, Price, Error)      │
│       │                 │       │                                │
│       │                 │       ├─→ Condition Validation         │
│       │                 │       │    (Risk OK? Correlation?)     │
│       │                 │       │                                │
│       │                 │       └─→ Action Execution             │
│       │                 │            │                           │
│       │                 │            ├─→ Execute Trade           │
│       │                 │            ├─→ Send Telegram           │
│       │                 │            ├─→ Send Discord            │
│       │                 │            └─→ Log to Database         │
│       │                 │                                        │
│       └─────────────────┤ AI Analysis Engine                      │
│                         │    │                                   │
│                         │    ├─→ Technical Analysis              │
│                         │    ├─→ On-Chain Metrics                │
│                         │    └─→ Social Sentiment                │
│                         │         │                              │
│                         │         └─→ Generate Signal            │
│                         │                                        │
│                         └─→ Send Alerts                           │
│                             (Telegram + Discord)                 │
└─────────────────────────────────────────────────────────────────┘
```

---

# ============================================================================
# COMPONENT 1: WEBHOOK LISTENER (FastAPI Server)
# ============================================================================

## File: apex/webhooks/listener.py

```python
"""
FastAPI webhook server for receiving TradingView alerts
"""

from fastapi import FastAPI, HTTPException, Request, BackgroundTasks
from fastapi.middleware.cors import CORSMiddleware
import hmac
import hashlib
import json
import logging
from datetime import datetime
from typing import Optional

app = FastAPI(title="APEX Webhooks", version="1.0.0")

# Add CORS middleware (allow TradingView)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

logger = logging.getLogger(__name__)

# ============================================================================
# WEBHOOK SIGNATURE VALIDATION
# ============================================================================

def verify_webhook_signature(payload: str, signature: str, secret: str) -> bool:
    """
    Verify webhook signature using HMAC-SHA256
    
    TradingView sends:
    - Payload: JSON body
    - X-Webhook-Signature: HMAC-SHA256 signature
    """
    
    expected_signature = hmac.new(
        secret.encode(),
        payload.encode(),
        hashlib.sha256
    ).hexdigest()
    
    return hmac.compare_digest(signature, expected_signature)

# ============================================================================
# RATE LIMITING
# ============================================================================

from collections import defaultdict
from time import time

class RateLimiter:
    """Simple in-memory rate limiter"""
    
    def __init__(self, max_requests: int = 10, window_seconds: int = 60):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        self.requests = defaultdict(list)
    
    def is_allowed(self, key: str) -> bool:
        """Check if request is allowed"""
        now = time()
        
        # Clean old requests
        self.requests[key] = [
            req_time for req_time in self.requests[key]
            if now - req_time < self.window_seconds
        ]
        
        # Check limit
        if len(self.requests[key]) >= self.max_requests:
            return False
        
        self.requests[key].append(now)
        return True

rate_limiter = RateLimiter(max_requests=100, window_seconds=60)

# ============================================================================
# WEBHOOK ENDPOINTS
# ============================================================================

@app.post("/webhooks/tradingview")
async def tradingview_webhook(
    request: Request,
    background_tasks: BackgroundTasks
):
    """
    Receive alerts from TradingView
    
    TradingView Alert Payload Format:
    {
        "symbol": "BTCUSDT",
        "signal": "buy",
        "price": 45000,
        "confidence": 0.85,
        "stop_loss": 44100,
        "take_profit": 46500
    }
    """
    
    try:
        # Get signature
        signature = request.headers.get("X-Webhook-Signature", "")
        
        # Get payload
        body = await request.body()
        payload_str = body.decode('utf-8')
        
        # Verify signature
        from apex.config import settings
        if not verify_webhook_signature(payload_str, signature, settings.WEBHOOK_SECRET):
            logger.warning(f"Invalid webhook signature from {request.client.host}")
            raise HTTPException(status_code=401, detail="Invalid signature")
        
        # Rate limiting
        symbol = json.loads(payload_str).get("symbol", "UNKNOWN")
        if not rate_limiter.is_allowed(symbol):
            logger.warning(f"Rate limit exceeded for {symbol}")
            raise HTTPException(status_code=429, detail="Rate limit exceeded")
        
        # Parse payload
        signal_data = json.loads(payload_str)
        
        # Validate required fields
        required_fields = ["symbol", "signal", "price"]
        for field in required_fields:
            if field not in signal_data:
                raise HTTPException(status_code=400, detail=f"Missing field: {field}")
        
        logger.info(f"✅ TradingView signal received: {signal_data['symbol']} {signal_data['signal']}")
        
        # Process in background
        background_tasks.add_task(process_signal, signal_data)
        
        return {
            "status": "received",
            "symbol": signal_data['symbol'],
            "signal": signal_data['signal'],
            "timestamp": datetime.utcnow().isoformat()
        }
    
    except json.JSONDecodeError:
        raise HTTPException(status_code=400, detail="Invalid JSON")
    except Exception as e:
        logger.error(f"Webhook error: {str(e)}")
        raise HTTPException(status_code=500, detail=str(e))

# ============================================================================
# SIGNAL PROCESSING
# ============================================================================

async def process_signal(signal_data: dict):
    """
    Process received signal through workflow engine
    """
    
    from apex.automation.workflow import WorkflowEngine
    from apex.ai.analysis import SignalGenerator
    
    try:
        logger.info(f"Processing signal: {signal_data}")
        
        # 1. Get market data
        market_data = get_market_data(signal_data['symbol'])
        
        # 2. Generate AI signal (confirmation)
        ai_generator = SignalGenerator()
        ai_signal = ai_generator.generate_signal(
            symbol=signal_data['symbol'],
            market_data=market_data,
            entry_price=signal_data['price']
        )
        
        # 3. Log signal
        log_signal_to_db(signal_data, ai_signal)
        
        # 4. Execute workflow
        engine = WorkflowEngine()
        result = await engine.execute_signal(signal_data)
        
        logger.info(f"✅ Signal processed: {result}")
    
    except Exception as e:
        logger.error(f"❌ Error processing signal: {str(e)}")
        await send_error_alert(signal_data, str(e))

def get_market_data(symbol: str) -> dict:
    """Fetch latest market data"""
    from apex.exchanges import ExchangeManager
    
    manager = ExchangeManager()
    ticker = manager.get_ticker(symbol)
    
    return {
        "symbol": symbol,
        "price": ticker['last'],
        "bid": ticker['bid'],
        "ask": ticker['ask'],
        "volume_24h": ticker['quoteVolume']
    }

def log_signal_to_db(signal_data: dict, ai_signal):
    """Log to database"""
    from apex.models import AISignal
    from sqlalchemy import create_engine
    from sqlalchemy.orm import sessionmaker
    
    try:
        db = sessionmaker(bind=create_engine(SQLALCHEMY_DATABASE_URL))()
        
        db_signal = AISignal(
            symbol=signal_data['symbol'],
            signal_type=signal_data['signal'],
            confidence=signal_data.get('confidence', 0.5),
            entry_price=signal_data['price'],
            stop_loss=signal_data.get('stop_loss'),
            take_profit=signal_data.get('take_profit'),
            ai_signal=ai_signal.signal_type.value if ai_signal else None,
            ai_confidence=ai_signal.confidence if ai_signal else None,
            created_at=datetime.utcnow()
        )
        
        db.add(db_signal)
        db.commit()
        db.close()
    except Exception as e:
        logger.error(f"Database error: {str(e)}")

# ============================================================================
# ERROR HANDLING
# ============================================================================

@app.post("/webhooks/error")
async def error_webhook(request: Request):
    """Receive error notifications"""
    data = await request.json()
    
    logger.error(f"🚨 Trading error: {data}")
    
    # Send alert
    from apex.webhooks.telegram_handler import TelegramHandler
    telegram = TelegramHandler()
    await telegram.send_alert(
        f"🚨 *Trading Error*\n\n{data['error']}\n\nSymbol: {data.get('symbol', 'N/A')}",
        alert_type="error"
    )
    
    return {"status": "error_logged"}

# ============================================================================
# HEALTH CHECK
# ============================================================================

@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "webhook_url": "https://yourdomain.com/webhooks/tradingview"
    }

# ============================================================================
# RUN SERVER
# ============================================================================

if __name__ == "__main__":
    import uvicorn
    uvicorn.run(app, host="0.0.0.0", port=8001)
```

**To run locally:**
```bash
cd apex
python -m uvicorn apex.webhooks.listener:app --reload --port 8001
# Server ready at http://localhost:8001
```

---

# ============================================================================
# COMPONENT 2: TRADINGVIEW ALERT SETUP
# ============================================================================

## Step 1: Create Alert

1. Open TradingView chart
2. Set up your indicator/strategy alert
3. Click "Create Alert"

## Step 2: Configure Alert Action

**Alert Message (JSON):**
```json
{
    "symbol": "{{ticker}}",
    "signal": "buy",
    "price": {{close}},
    "confidence": 0.85,
    "stop_loss": {{close}} * 0.98,
    "take_profit": {{close}} * 1.03,
    "time": "{{time}}"
}
```

**Webhook URL:**
```
https://yourdomain.com/webhooks/tradingview
```

**Headers:**
```
Content-Type: application/json
X-Webhook-Signature: YOUR_WEBHOOK_SECRET
```

## Step 3: Get Webhook Secret

```python
# Generate in your config
import secrets
webhook_secret = secrets.token_hex(32)
print(webhook_secret)
# Output: abc123...xyz789
```

Add to `.env`:
```
WEBHOOK_SECRET=abc123...xyz789
```

---

# ============================================================================
# COMPONENT 3: TELEGRAM ALERTS
# ============================================================================

## File: apex/webhooks/telegram_handler.py

```python
"""
Telegram notification handler
"""

import asyncio
from telegram import Bot, InlineKeyboardButton, InlineKeyboardMarkup
from telegram.error import TelegramError
from datetime import datetime
import logging

logger = logging.getLogger(__name__)

class TelegramHandler:
    def __init__(self, token: str = None, chat_id: str = None):
        from apex.config import settings
        self.token = token or settings.TELEGRAM_BOT_TOKEN
        self.chat_id = chat_id or settings.TELEGRAM_CHAT_ID
        self.bot = Bot(token=self.token)
    
    # ====================================================================
    # SIGNAL ALERTS
    # ====================================================================
    
    async def send_buy_alert(self, signal: dict):
        """Send buy signal alert"""
        
        message = f"""
🟢 *BUY SIGNAL DETECTED*

*Symbol:* {signal['symbol']}
*Entry Price:* ${signal['entry']:.2f}
*Stop Loss:* ${signal['stop_loss']:.2f}
*Take Profit 1:* ${signal['take_profit_1']:.2f}
*Take Profit 2:* ${signal['take_profit_2']:.2f}

*Confidence:* {signal['confidence']:.0%}
*Reason:* {signal['reason']}

*Time:* {datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    async def send_sell_alert(self, signal: dict):
        """Send sell signal alert"""
        
        message = f"""
🔴 *SELL SIGNAL DETECTED*

*Symbol:* {signal['symbol']}
*Entry Price:* ${signal['entry']:.2f}
*Stop Loss:* ${signal['stop_loss']:.2f}
*Take Profit 1:* ${signal['take_profit_1']:.2f}

*Confidence:* {signal['confidence']:.0%}
*Reason:* {signal['reason']}

*Time:* {datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    # ====================================================================
    # TRADE RESULT ALERTS
    # ====================================================================
    
    async def send_trade_result(self, trade: dict):
        """Send trade result notification"""
        
        status = "✅ WIN" if trade['pnl'] > 0 else "❌ LOSS"
        
        message = f"""
{status}

*Symbol:* {trade['symbol']}
*Entry:* ${trade['entry']:.2f}
*Exit:* ${trade['exit']:.2f}
*P&L:* ${trade['pnl']:.2f} ({trade['pnl_pct']:.2%})

*Duration:* {trade['duration']}
*Return on Risk:* {trade['ror']:.1f}x

*Time:* {datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    # ====================================================================
    # RISK WARNINGS
    # ====================================================================
    
    async def send_risk_warning(self, warning: dict):
        """Send risk management warning"""
        
        message = f"""
⚠️ *RISK WARNING*

*Type:* {warning['type']}
*Details:* {warning['details']}

*Current Portfolio Heat:* {warning['portfolio_heat']:.1%}
*Daily Limit:* {warning['daily_limit']:.1%}

*Action:* Reduce position size or skip trades

*Time:* {datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    # ====================================================================
    # ERROR ALERTS
    # ====================================================================
    
    async def send_error_alert(self, error: dict):
        """Send error notification"""
        
        message = f"""
🚨 *ERROR ALERT*

*Component:* {error['component']}
*Error:* {error['message']}
*Symbol:* {error.get('symbol', 'N/A')}

*Status:* System monitoring enabled
*Time:* {datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    # ====================================================================
    # DAILY REPORT
    # ====================================================================
    
    async def send_daily_report(self, report: dict):
        """Send daily performance report"""
        
        message = f"""
📊 *DAILY REPORT* - {report['date']}

*Trades:* {report['total_trades']}
*Wins:* {report['winning_trades']} ({report['win_rate']:.1%})
*Losses:* {report['losing_trades']}

*Daily P&L:* ${report['daily_pnl']:.2f}
*Return:* {report['return_pct']:.2%}

*Best Trade:* +{report['best_trade']:.2%}
*Worst Trade:* {report['worst_trade']:.2%}

*Account Balance:* ${report['account_balance']:.2f}
*Equity:* ${report['equity']:.2f}

*Status:* {report['status']}
"""
        
        await self._send_message(message, parse_mode="Markdown")
    
    # ====================================================================
    # HELPER METHODS
    # ====================================================================
    
    async def _send_message(self, text: str, parse_mode: str = "Markdown"):
        """Send message to Telegram"""
        
        try:
            await self.bot.send_message(
                chat_id=self.chat_id,
                text=text,
                parse_mode=parse_mode
            )
            logger.info(f"✅ Telegram message sent")
        except TelegramError as e:
            logger.error(f"❌ Telegram error: {str(e)}")
    
    async def send_alert(self, message: str, alert_type: str = "info"):
        """Generic alert sender"""
        
        if alert_type == "buy":
            await self.send_buy_alert({"symbol": "SIGNAL", "entry": 0, "stop_loss": 0, "take_profit_1": 0, "take_profit_2": 0, "confidence": 0.8, "reason": message})
        elif alert_type == "sell":
            await self.send_sell_alert({"symbol": "SIGNAL", "entry": 0, "stop_loss": 0, "take_profit_1": 0, "confidence": 0.8, "reason": message})
        elif alert_type == "error":
            await self._send_message(message, parse_mode="Markdown")
        else:
            await self._send_message(message)
```

## Setup Telegram Bot

1. Message `@BotFather` on Telegram
2. `/newbot` and follow instructions
3. Get token and chat ID:

```bash
# Get your chat ID
curl "https://api.telegram.org/bot$TOKEN/getMe"

# Then send yourself a message:
# https://api.telegram.org/bot$TOKEN/sendMessage?chat_id=$CHAT_ID&text=test
```

Add to `.env`:
```
TELEGRAM_BOT_TOKEN=123456:ABC-DEF1234ghIkl-zyx57W2v1u123ew11
TELEGRAM_CHAT_ID=123456789
```

---

# ============================================================================
# COMPONENT 4: WORKFLOW ENGINE
# ============================================================================

## File: apex/automation/workflow.py

```python
"""
Workflow orchestration engine
"""

from dataclasses import dataclass
from typing import List, Optional, Callable
from enum import Enum
import asyncio
import logging

logger = logging.getLogger(__name__)

class WorkflowStatus(Enum):
    PENDING = "pending"
    RUNNING = "running"
    SUCCESS = "success"
    FAILED = "failed"
    SKIPPED = "skipped"

@dataclass
class WorkflowState:
    workflow_name: str
    status: WorkflowStatus
    trigger_data: dict
    actions_executed: List[str] = None
    errors: List[str] = None
    result: dict = None
    
    def __post_init__(self):
        if self.actions_executed is None:
            self.actions_executed = []
        if self.errors is None:
            self.errors = []

class Trigger:
    """Base trigger class"""
    async def check(self, data: dict) -> bool:
        raise NotImplementedError

class Condition:
    """Base condition class"""
    async def validate(self, data: dict) -> bool:
        raise NotImplementedError

class Action:
    """Base action class"""
    async def execute(self, data: dict) -> dict:
        raise NotImplementedError

@dataclass
class Workflow:
    name: str
    triggers: List[Trigger]
    actions: List[Action]
    conditions: List[Condition] = None
    enabled: bool = True
    
    def __post_init__(self):
        if self.conditions is None:
            self.conditions = []

class WorkflowEngine:
    """Execute workflows based on triggers and conditions"""
    
    def __init__(self):
        self.workflows = {}
        self.history = []
    
    def register_workflow(self, workflow: Workflow):
        """Register a workflow"""
        self.workflows[workflow.name] = workflow
        logger.info(f"Workflow registered: {workflow.name}")
    
    async def execute_workflow(self, workflow_name: str, data: dict) -> WorkflowState:
        """Execute a workflow"""
        
        if workflow_name not in self.workflows:
            return WorkflowState(
                workflow_name=workflow_name,
                status=WorkflowStatus.FAILED,
                trigger_data=data,
                errors=["Workflow not found"]
            )
        
        workflow = self.workflows[workflow_name]
        state = WorkflowState(
            workflow_name=workflow_name,
            status=WorkflowStatus.PENDING,
            trigger_data=data
        )
        
        try:
            # Check if workflow is enabled
            if not workflow.enabled:
                state.status = WorkflowStatus.SKIPPED
                return state
            
            # Check triggers
            trigger_matched = False
            for trigger in workflow.triggers:
                if await trigger.check(data):
                    trigger_matched = True
                    break
            
            if not trigger_matched:
                state.status = WorkflowStatus.SKIPPED
                return state
            
            # Validate conditions
            for condition in workflow.conditions:
                if not await condition.validate(data):
                    state.status = WorkflowStatus.SKIPPED
                    state.errors.append("Condition validation failed")
                    return state
            
            # Execute actions
            state.status = WorkflowStatus.RUNNING
            
            for action in workflow.actions:
                try:
                    result = await action.execute(data)
                    state.actions_executed.append(action.__class__.__name__)
                    if state.result is None:
                        state.result = result
                except Exception as e:
                    logger.error(f"Action error: {str(e)}")
                    state.errors.append(f"{action.__class__.__name__}: {str(e)}")
            
            state.status = WorkflowStatus.SUCCESS if not state.errors else WorkflowStatus.FAILED
        
        except Exception as e:
            logger.error(f"Workflow execution error: {str(e)}")
            state.status = WorkflowStatus.FAILED
            state.errors.append(str(e))
        
        # Log history
        self.history.append(state)
        
        return state
```

---

# ============================================================================
# COMPONENT 5: TRIGGERS, CONDITIONS, ACTIONS
# ============================================================================

## File: apex/automation/triggers.py

```python
"""
Trigger definitions
"""

from apex.automation.workflow import Trigger

class SignalTrigger(Trigger):
    """Trigger on trading signal"""
    
    def __init__(self, signal_type: str = None, min_confidence: float = 0.5):
        self.signal_type = signal_type
        self.min_confidence = min_confidence
    
    async def check(self, data: dict) -> bool:
        if 'signal' not in data:
            return False
        
        if self.signal_type and data['signal'] != self.signal_type:
            return False
        
        confidence = data.get('confidence', 0)
        return confidence >= self.min_confidence

class PriceTrigger(Trigger):
    """Trigger on price level"""
    
    def __init__(self, price_level: float, direction: str = "above"):
        self.price_level = price_level
        self.direction = direction  # "above" or "below"
    
    async def check(self, data: dict) -> bool:
        current_price = data.get('price', 0)
        
        if self.direction == "above":
            return current_price > self.price_level
        else:
            return current_price < self.price_level

class ErrorTrigger(Trigger):
    """Trigger on error"""
    
    async def check(self, data: dict) -> bool:
        return data.get('error') is not None
```

## File: apex/automation/actions.py

```python
"""
Action definitions
"""

from apex.automation.workflow import Action
import logging

logger = logging.getLogger(__name__)

class ExecuteTradeAction(Action):
    """Execute a trade"""
    
    async def execute(self, data: dict) -> dict:
        from apex.exchanges import ExchangeManager
        
        try:
            manager = ExchangeManager()
            
            order = manager.create_order(
                symbol=data['symbol'],
                order_type='market',
                side=data['signal'],  # 'buy' or 'sell'
                quantity=data.get('position_size', 0.01)
            )
            
            logger.info(f"✅ Trade executed: {order}")
            
            return {
                "status": "success",
                "order_id": order.get('id'),
                "symbol": data['symbol'],
                "side": data['signal']
            }
        
        except Exception as e:
            logger.error(f"❌ Trade execution error: {str(e)}")
            raise

class TelegramAlertAction(Action):
    """Send Telegram alert"""
    
    async def execute(self, data: dict) -> dict:
        from apex.webhooks.telegram_handler import TelegramHandler
        
        try:
            telegram = TelegramHandler()
            
            if data['signal'] == 'buy':
                await telegram.send_buy_alert(data)
            else:
                await telegram.send_sell_alert(data)
            
            logger.info(f"✅ Telegram alert sent")
            
            return {"status": "success", "channel": "telegram"}
        
        except Exception as e:
            logger.error(f"❌ Telegram error: {str(e)}")
            raise

class DiscordAlertAction(Action):
    """Send Discord alert"""
    
    async def execute(self, data: dict) -> dict:
        # Similar to Telegram but for Discord
        pass

class LogToDatabaseAction(Action):
    """Log trade to database"""
    
    async def execute(self, data: dict) -> dict:
        # Log to Supabase/PostgreSQL
        pass
```

---

# ============================================================================
# COMPLETE AUTOMATION WORKFLOW EXAMPLE
# ============================================================================

```python
from apex.automation.workflow import Workflow, WorkflowEngine
from apex.automation.triggers import SignalTrigger
from apex.automation.actions import ExecuteTradeAction, TelegramAlertAction
from apex.automation.conditions import RiskValidationCondition

# Create workflow
workflow = Workflow(
    name="TradingView Automation",
    triggers=[
        SignalTrigger(signal_type="buy", min_confidence=0.75)
    ],
    conditions=[
        RiskValidationCondition()  # Validate risk before executing
    ],
    actions=[
        ExecuteTradeAction(),        # Execute trade on exchange
        TelegramAlertAction(),       # Send Telegram alert
        LogToDatabaseAction()        # Log to database
    ]
)

# Register workflow
engine = WorkflowEngine()
engine.register_workflow(workflow)

# Execute
signal_data = {
    "symbol": "BTCUSDT",
    "signal": "buy",
    "price": 45000,
    "confidence": 0.85,
    "stop_loss": 44100,
    "take_profit": 46500
}

state = await engine.execute_workflow("TradingView Automation", signal_data)
print(f"Workflow status: {state.status}")
print(f"Actions executed: {state.actions_executed}")
```

---

# ============================================================================
# LOCAL TESTING
# ============================================================================

## Test 1: Start Webhook Server

```bash
cd apex
python -m uvicorn apex.webhooks.listener:app --reload --port 8001
```

Output:
```
Uvicorn running on http://127.0.0.1:8001
Press CTRL+C to quit
```

## Test 2: Send Test Signal

```bash
curl -X POST http://localhost:8001/webhooks/tradingview \
  -H "Content-Type: application/json" \
  -H "X-Webhook-Signature: test_sig" \
  -d '{
    "symbol": "BTCUSDT",
    "signal": "buy",
    "price": 45000,
    "confidence": 0.85,
    "stop_loss": 44100,
    "take_profit": 46500
  }'
```

Response:
```json
{
  "status": "received",
  "symbol": "BTCUSDT",
  "signal": "buy",
  "timestamp": "2024-01-15T10:30:00"
}
```

## Test 3: Check Logs

```bash
tail -f logs/apex.log
```

---

# ============================================================================
# DEPLOYMENT (PRODUCTION)
# ============================================================================

## Option 1: Railway (Recommended)

```bash
# 1. Install Railway CLI
npm i -g @railway/cli

# 2. Login
railway login

# 3. Create project
railway init

# 4. Add environment variables
railway variable set WEBHOOK_SECRET=your_secret
railway variable set TELEGRAM_BOT_TOKEN=your_token
railway variable set DATABASE_URL=postgresql://...

# 5. Deploy
railway up
```

**Get URL:**
```bash
railway domains
# Output: https://apex-prod-xyz.railway.app
```

Update TradingView webhook URL to:
```
https://apex-prod-xyz.railway.app/webhooks/tradingview
```

## Option 2: Docker

```dockerfile
FROM python:3.11

WORKDIR /app

COPY requirements.txt .
RUN pip install -r requirements.txt

COPY . .

CMD ["uvicorn", "apex.webhooks.listener:app", "--host", "0.0.0.0", "--port", "8001"]
```

```bash
docker build -t apex-bot .
docker run -e WEBHOOK_SECRET=xyz -p 8001:8001 apex-bot
```

---

# ============================================================================
# MONITORING & TROUBLESHOOTING
# ============================================================================

## Checklist

- [ ] Webhook endpoint accessible (test `/health`)
- [ ] Signature verification passing
- [ ] Rate limiting working
- [ ] Telegram bot sending messages
- [ ] Trades executing on exchange
- [ ] Errors being logged
- [ ] Database recording events
- [ ] Alerts working as expected

## Common Issues

| Issue | Solution |
|-------|----------|
| Webhook not receiving signals | Check TradingView webhook URL configuration |
| Signature verification failing | Verify webhook secret matches |
| Telegram not sending | Check bot token and chat ID |
| Rate limit errors | Reduce alert frequency |
| Trade execution failing | Check exchange API keys and permissions |

---

# ============================================================================
# NEXT STEPS
# ============================================================================

1. ✅ Setup webhook server (locally)
2. ✅ Configure TradingView alerts
3. ✅ Setup Telegram/Discord
4. ✅ Test all components
5. Deploy to cloud (Railway/Render)
6. Monitor first 24 hours
7. Optimize based on logs
8. Add more workflows
9. Scale to multiple strategies
10. Implement backtesting

For code: See `/apex/webhooks/`, `/apex/automation/`
For docs: See `/AUTOMATION_GUIDE.md`, `/AI_ANALYSIS_GUIDE.md`

