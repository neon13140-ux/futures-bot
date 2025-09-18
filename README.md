# futures-bot
Binance Futures trading bot with Telegram integration
import telebot
from binance.client import Client
from binance.enums import *
import os

# ====== ENV VARIABLES ======
TELEGRAM_TOKEN = os.getenv("TELEGRAM_TOKEN")
CHAT_ID = os.getenv("TELEGRAM_CHAT_ID")
API_KEY = os.getenv("BINANCE_API_KEY")
API_SECRET = os.getenv("BINANCE_API_SECRET")
USE_TESTNET = os.getenv("USE_TESTNET", "true").lower() == "true"

# ====== INIT ======
bot = telebot.TeleBot(TELEGRAM_TOKEN)
client = Client(API_KEY, API_SECRET, testnet=USE_TESTNET)

print("✅ Bot started... waiting for signals")

# ====== HELPERS ======
def parse_signal(text: str):
    """
    Example:
    BUY BTCUSDT 0.01 LEV=10 SL=64000 TP1=66000 TP2=67000
    SELL ETHUSDT 0.5 LEV=20 SL=2500 TP1=2450
    """
    parts = text.strip().split()
    if len(parts) < 3:
        return None

    side = parts[0].upper()
    symbol = parts[1].upper()
    qty = float(parts[2])

    params = {"side": side, "symbol": symbol, "qty": qty, "lev": 10, "sl": None, "tp": []}

    for p in parts[3:]:
        if p.startswith("LEV="):
            params["lev"] = int(p.split("=")[1])
        elif p.startswith("SL="):
            params["sl"] = float(p.split("=")[1])
        elif p.startswith("TP"):
            params["tp"].append(float(p.split("=")[1]))

    return params


def place_futures_order(params):
    symbol = params["symbol"]
    side = SIDE_BUY if params["side"] == "BUY" else SIDE_SELL

    # set leverage
    client.futures_change_leverage(symbol=symbol, leverage=params["lev"])

    # open position
    order = client.futures_create_order(
        symbol=symbol,
        side=side,
        type=FUTURE_ORDER_TYPE_MARKET,
        quantity=params["qty"]
    )

    # set SL and TP if provided
    if params["sl"]:
        client.futures_create_order(
            symbol=symbol,
            side=SIDE_SELL if side == SIDE_BUY else SIDE_BUY,
            type=FUTURE_ORDER_TYPE_STOP_MARKET,
            stopPrice=params["sl"],
            closePosition=True
        )

    for tp in params["tp"]:
        client.futures_create_order(
            symbol=symbol,
            side=SIDE_SELL if side == SIDE_BUY else SIDE_BUY,
            type=FUTURE_ORDER_TYPE_LIMIT,
            price=tp,
            timeInForce=TIME_IN_FORCE_GTC,
            closePosition=True
        )

    return order


# ====== TELEGRAM HANDLER ======
@bot.message_handler(func=lambda message: True)
def handle_signal(message):
    if str(message.chat.id) != str(CHAT_ID):
        bot.reply_to(message, "⛔ Unauthorized")
        return

    params = parse_signal(message.text)
    if not params:
        bot.reply_to(message, "⚠️ Wrong signal format")
        return

    try:
        order = place_futures_order(params)
        bot.reply_to(message, f"✅ Order placed: {order}")
    except Exception as e:
        bot.reply_to(message, f"❌ Error: {e}")


# ====== RUN ======
bot.polling(none_stop=True)
