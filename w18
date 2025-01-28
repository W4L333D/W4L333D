import os
import yaml
import asyncpg
import aiohttp
import asyncio
from telegram import Update, Bot
from telegram.ext import Application, CommandHandler, ContextTypes
from datetime import datetime, timedelta
from typing import Dict, List, Optional

class CryptoAlphaBot:
    def __init__(self):
        self.config = self.load_config()
        self.session = aiohttp.ClientSession()
        self.pool = None
        self.tg_bot = None
        
        # APIs
        self.dexscreener_api = "https://api.dexscreener.com/latest/dex"
        self.rugcheck_api = "https://rugcheck.xyz/api/v1/token"
        self.bonkbot_api = "https://api.bonkbot.com/trade"
        
        # Initialize Telegram
        self.tg_app = Application.builder().token(self.config['telegram']['token']).build()
        self.register_handlers()

    def load_config(self) -> Dict:
        """Load configuration from YAML file"""
        with open("config.yaml", 'r') as f:
            return yaml.safe_load(f)

    async def init_db(self):
        """Initialize database with all tables"""
        self.pool = await asyncpg.create_pool(**self.config['database'])
        async with self.pool.acquire() as conn:
            # Create all previous tables plus new trading tables
            await conn.execute('''
                CREATE TABLE IF NOT EXISTS trades (
                    trade_id SERIAL PRIMARY KEY,
                    chat_id BIGINT,
                    pair_address TEXT,
                    direction TEXT,
                    amount NUMERIC,
                    price NUMERIC,
                    status TEXT,
                    executed_at TIMESTAMP
                );
                
                CREATE TABLE IF NOT EXISTS user_portfolios (
                    user_id BIGINT PRIMARY KEY,
                    holdings JSONB,
                    risk_level TEXT,
                    last_updated TIMESTAMP
                );
                -- Include previous tables from earlier implementations
            ''')

    async def tg_send_message(self, chat_id: int, message: str):
        """Send Telegram message"""
        await self.tg_bot.send_message(chat_id=chat_id, text=message)

    async def execute_trade(self, pair: Dict, direction: str, amount: float, chat_id: int):
        """Execute trade through BonkBot"""
        headers = {
            "x-bonkbot-key": self.config['bonkbot']['api_key'],
            "x-chat-id": str(chat_id)
        }
        
        payload = {
            "pair_address": pair['pairAddress'],
            "direction": direction,
            "amount": amount,
            "slippage": self.config['trading']['slippage']
        }

        async with self.session.post(self.bonkbot_api, json=payload, headers=headers) as response:
            result = await response.json()
            if response.status == 200:
                await self.log_trade(chat_id, pair, direction, amount, result['price'])
                await self.tg_send_message(chat_id, 
                    f"✅ {direction.upper()} executed\n"
                    f"Token: {pair['baseToken']['symbol']}\n"
                    f"Amount: {amount:.4f}\n"
                    f"Price: ${result['price']:.6f}"
                )
            else:
                await self.tg_send_message(chat_id, 
                    f"❌ Trade failed\nReason: {result.get('error', 'Unknown error')}"
                )

    async def log_trade(self, chat_id: int, pair: Dict, direction: str, amount: float, price: float):
        """Record trade in database"""
        async with self.pool.acquire() as conn:
            await conn.execute('''
                INSERT INTO trades 
                (chat_id, pair_address, direction, amount, price, status, executed_at)
                VALUES ($1, $2, $3, $4, $5, $6, $7)
            ''', chat_id, pair['pairAddress'], direction, amount, price, 'filled', datetime.utcnow())

    # Previous security/filtering implementations
    # (RugCheck, fake volume detection, blacklisting, etc.)
    
    async def analyze_and_trade(self):
        """Main analysis loop with trading signals"""
        while True:
            pairs = await self.fetch_market_data()
            safe_pairs = await self.apply_security_checks(pairs)
            
            for pair in safe_pairs:
                if self.generate_signal(pair):
                    await self.send_trading_alert(pair)
            
            await asyncio.sleep(self.config['trading']['analysis_interval'])

    def generate_signal(self, pair: Dict) -> bool:
        """Generate trading signals based on analysis"""
        # Implement your trading strategy logic here
        return False  # Placeholder

    async def send_trading_alert(self, pair: Dict):
        """Send trading alert to subscribed users"""
        message = (
            f"🚨 Trading Signal Detected\n"
            f"Token: {pair['baseToken']['symbol']}\n"
            f"Price: ${pair['priceNative']:.6f}\n"
            f"Volume: ${pair['volume']['h24']:,.2f}"
        )
        
        async with self.pool.acquire() as conn:
            users = await conn.fetch("SELECT user_id FROM user_portfolios")
            for user in users:
                await self.tg_send_message(user['user_id'], message)

    # Telegram Command Handlers
    def register_handlers(self):
        self.tg_app.add_handler(CommandHandler("start", self.start_handler))
        self.tg_app.add_handler(CommandHandler("buy", self.buy_handler))
        self.tg_app.add_handler(CommandHandler("sell", self.sell_handler))
        self.tg_app.add_handler(CommandHandler("portfolio", self.portfolio_handler))

    async def start_handler(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        await update.message.reply_text(
            "🦍 Crypto Alpha Bot Ready\n\n"
            "Commands:\n"
            "/buy [token] [amount] - Execute buy order\n"
            "/sell [token] [amount] - Execute sell order\n"
            "/portfolio - View holdings\n"
            "/alerts [on/off] - Manage trading signals"
        )

    async def buy_handler(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id
        try:
            token_address = context.args[0]
            amount = float(context.args[1])
            
            pair = await self.verify_pair(token_address)
            if pair:
                await self.execute_trade(pair, 'buy', amount, chat_id)
            else:
                await update.message.reply_text("❌ Invalid or unsafe token")
                
        except Exception as e:
            await update.message.reply_text(f"Error: {str(e)}")

    async def sell_handler(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        # Similar to buy_handler
        pass

    async def portfolio_handler(self, update: Update, context: ContextTypes.DEFAULT_TYPE):
        chat_id = update.effective_chat.id
        async with self.pool.acquire() as conn:
            portfolio = await conn.fetchrow(
                "SELECT holdings FROM user_portfolios WHERE user_id = $1", chat_id
            )
        
        if portfolio:
            await update.message.reply_text(f"📊 Portfolio:\n{portfolio['holdings']}")
        else:
            await update.message.reply_text("No holdings found")

    async def run(self):
        """Launch the bot"""
        await self.init_db()
        await self.tg_app.initialize()
        await self.tg_app.start()
        await self.analyze_and_trade()

# config.yaml example
"""
telegram:
  token: YOUR_TELEGRAM_TOKEN
  chat_id: YOUR_CHAT_ID

bonkbot:
  api_key: YOUR_BONKBOT_KEY
  slippage: 1.5

trading:
  analysis_interval: 300
  max_position_size: 0.1
  take_profit: 0.15
  stop_loss: 0.1

# Previous configuration sections...
"""

if __name__ == "__main__":
    bot = CryptoAlphaBot()
    asyncio.run(bot.run())
