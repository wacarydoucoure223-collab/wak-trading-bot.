# wak-trading-bot.
bot_wak.py
#!/usr/bin/env python3
# bot_wak.py
# Wak Trading Bot - version simple DCA / Paper trading by default
# Usage: python3 bot_wak.py

import os
import time
import logging
import math
from decimal import Decimal
from dotenv import load_dotenv

# Optional Binance client (only used if REAL=1)
try:
    from binance.client import Client as BinanceClient
    from binance.exceptions import BinanceAPIException
    BINANCE_AVAILABLE = True
except Exception:
    BINANCE_AVAILABLE = False

load_dotenv()

# --------------------
# Configuration (via .env)
# --------------------
SYMBOL = os.getenv("SYMBOL", "ETHUSDT")           # paire
BASE_ASSET = os.getenv("BASE_ASSET", "USDT")      # monnaie de base (ex: USDT)
INVESTMENT_PER_ORDER = float(os.getenv("INVESTMENT_PER_ORDER", "10"))  # montant en base asset par ordre (ex: 10 USDT)
DCA_INTERVAL_MIN = int(os.getenv("DCA_INTERVAL_MIN", "60"))  # intervalle entre checks (minutes)
MAX_ORDERS = int(os.getenv("MAX_ORDERS", "20"))  # nombre max d'ordres DCA
SLIPPAGE_PCT = float(os.getenv("SLIPPAGE_PCT", "0.5"))  # slippage estimé en %
REAL = int(os.getenv("REAL", "0"))  # 0 = paper, 1 = place real order (requires keys & python-binance)
USE_TESTNET = int(os.getenv("USE_TESTNET", "1"))  # si REAL=1, définir testnet d'abord
API_KEY = os.getenv("BINANCE_API_KEY", "")
API_SECRET = os.getenv("BINANCE_API_SECRET", "")

# Safety caps
MAX_INVESTMENT_CAP = float(os.getenv("MAX_INVESTMENT_CAP", "1000"))  # limite totale investie
LOG_LEVEL = os.getenv("LOG_LEVEL", "INFO")

# --------------------
# Logging
# --------------------
logging.basicConfig(
    level=getattr(logging, LOG_LEVEL.upper(), logging.INFO),
    format="%(asctime)s | %(levelname)s | %(message)s",
    datefmt="%Y-%m-%d %H:%M:%S"
)
log = logging.getLogger("bot_wak")

# --------------------
# Exchange adapter (Binance) - only used when REAL=1 and python-binance present
# --------------------
class BinanceAdapter:
    def __init__(self, api_key, api_secret, testnet=True):
        if not BINANCE_AVAILABLE:
            raise RuntimeError("python-binance non installé. Installe via pip install python-binance")
        self.client = BinanceClient(api_key, api_secret)
        if testnet:
            # redirect to testnet API (python-binance)
            self.client.API_URL = "https://testnet.binance.vision/api"
            log.info("Binance client in TESTNET mode.")
        else:
            log.info("Binance client in LIVE mode.")

    def get_symbol_ticker_price(self, symbol):
        r = self.client.get_symbol_ticker(symbol=symbol)
        return float(r['price'])

    def place_market_buy(self, symbol, quoteOrderQty):
        # place market buy by quote quantity (BNB etc)
        return self.client.create_order(symbol=symbol, side='BUY', type='MARKET', quoteOrderQty=str(quoteOrderQty))

# --------------------
# Simple paper trading engine
# --------------------
class PaperEngine:
    def __init__(self):
        self.orders = []  # stores executed orders

    def get_price(self, symbol):
        # fallback price: simple approach using Binance public client if available
        if BINANCE_AVAILABLE and API_KEY:
            try:
                tmp = BinanceAdapter(api_key="", api_secret="", testnet=True)
                return tmp.get_symbol_ticker_price(symbol)
            except Exception:
                pass
        # If no internet or client, raise
        raise RuntimeError("Impossible d'obtenir le prix. Installe python-binance ou active une API.")

    def place_market_buy(self, symbol, quoteAmount):
        price = self.get_price(symbol)
        qty = float(Decimal(quoteAmount) / Decimal(price))
        # round qty to 6 decimals
        qty = float(round(qty, 6))
        order = {
            "id": f"PAPER-{len(self.orders)+1}",
            "symbol": symbol,
            "side": "BUY",
            "type": "MARKET",
            "quoteQty": float(quoteAmount),
            "executedQty": qty,
            "avgPrice": price,
            "timestamp": int(time.time() * 1000)
        }
        self.orders.append(order)
        log.info(f"[PAPER] Bought {qty} {symbol.replace('USDT','')} for {quoteAmount} {BASE_ASSET} at price {price}")
        return order

# --------------------
# Bot core
# --------------------
def run_bot():
    log.info("Démarrage bot_wak.py")
    if REAL:
        if not BINANCE_AVAILABLE:
            log.error("REAL=1 mais python-binance non installé. Quitte.")
            return
        if not API_KEY or not API_SECRET:
            log.error("REAL=1 mais clés API manquantes dans .env")
            return
        adapter = BinanceAdapter(api_key=API_KEY, api_secret=API_SECRET, testnet=bool(USE_TESTNET))
        engine = adapter
        log.warning("MODE RÉEL (ou testnet réel connecté). Vérifie les clés et le mode.")
    else:
        engine = PaperEngine()
        log.info("MODE PAPER (simulation). Pas d'ordres réels envoyés.")

    total_invested = 0.0
    executed_orders = 0

    try:
        while executed_orders < MAX_ORDERS:
            try:
                # get current market price
                if REAL:
                    price = engine.get_symbol_ticker_price(SYMBOL)
                else:
                    price = engine.get_price(SYMBOL)
                log.info(f"Prix actuel {SYMBOL}: {price:.6f} {BASE_ASSET}")

            except Exception as e:
                log.error(f"Erreur récupération prix: {e}")
                time.sleep(10)
                continue

            # Safety check total invested
            if total_invested + INVESTMENT_PER_ORDER > MAX_INVESTMENT_CAP:
                log.warning("Capitale max atteint, arrêt des nouveaux ordres.")
                break

            # Simple condition: buy every interval (DCA)
            # You can extend: buy only if price down X% since last buy etc.
            try:
                # place buy
                result = engine.place_market_buy(SYMBOL, INVESTMENT_PER_ORDER)
                executed_orders += 1
                total_invested += INVESTMENT_PER_ORDER
                log.info(f"Ordre exécuté #{executed_orders} — investissement total: {total_invested} {BASE_ASSET}")
            except Exception as e:
                log.error(f"Erreur exécution ordre: {e}")

            # Wait next cycle
            log.info(f"Attente {DCA_INTERVAL_MIN} minutes avant prochain check.")
            for _ in range(DCA_INTERVAL_MIN * 6):  # sleep in 10s chunks to be responsive to KeyboardInterrupt
                time.sleep(10)

    except KeyboardInterrupt:
        log.info("Arrêt manuel demandé (KeyboardInterrupt).")
    except Exception as e:
        log.exception(f"Erreur inattendue: {e}")

    log.info("Bot terminé. Résumé:")
    log.info(f"Ordres exécutés: {executed_orders}, Total investi: {total_invested} {BASE_ASSET}")

if __name__ == "__main__":
    run_bot() # .env.example -> copier comme .env
SYMBOL=ETHUSDT
BASE_ASSET=USDT
INVESTMENT_PER_ORDER=10
DCA_INTERVAL_MIN=60
MAX_ORDERS=20
SLIPPAGE_PCT=0.5
MAX_INVESTMENT_CAP=1000
LOG_LEVEL=INFO

# REAL trading flags
REAL=0
USE_TESTNET=1

# Only needed if REAL=1
BINANCE_API_KEY=
BINANCE_API_SECRET= python-dotenv>=1.0.0
python-binance>=1.0.17 python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt # paper (par défaut REAL=0)
python3 bot_wak.py python3 bot_wak.py # avec screen
screen -S bot_wak
source venv/bin/activate
python3 bot_wak.py
# ctrl+a d pour détacher

# ou nohup (simple)
nohup python3 bot_wak.py > bot_wak.log 2>&1 & python3 bot_wak.py # Empêche de publier tes clés API et configs sensibles
.env
*.env

# Dossiers de l'environnement virtuel Python
venv/
env/
ENV/
.venv/
.pyenv/

# Fichiers temporaires Python
__pycache__/
*.py[cod]
*.pyo
*.pyd

# Logs
*.log
log/
logs/

# Fichiers système inutiles
.DS_Store
Thumbs.db 
