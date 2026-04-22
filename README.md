# PrimeTradeAI
Trading Bot
#!/usr/bin/env python3
# bot.py
# Binance Futures Testnet Simplified Trading Bot
# IMPORTANT: Replace YOUR_API_KEY and YOUR_API_SECRET with your actual testnet credentials
# Get testnet API keys from: https://testnet.binancefuture.com
# Run Example:
# python bot.py --symbol BTCUSDT --side BUY --type MARKET --quantity 0.001
# python bot.py --symbol BTCUSDT --side SELL --type LIMIT --quantity 0.001 --price 50000

import argparse
import time
import hmac
import hashlib
import requests
import logging
from urllib.parse import urlencode

# ==================================================
# CONFIGURATION - REPLACE WITH YOUR TESTNET CREDENTIALS
# ==================================================
API_KEY = "YOUR_API_KEY"  # Get from https://testnet.binancefuture.com
API_SECRET = "YOUR_API_SECRET"  # Get from https://testnet.binancefuture.com
BASE_URL = "https://testnet.binancefuture.com"

# ==================================================
# LOGGING SETUP
# ==================================================
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.FileHandler("bot.log"),
        logging.StreamHandler()  # Also log to console
    ]
)

# ==================================================
# BINANCE CLIENT
# ==================================================
class BinanceFuturesBot:
    def __init__(self, api_key, api_secret, base_url):
        self.api_key = api_key
        self.api_secret = api_secret
        self.base_url = base_url

    def _generate_signature(self, params):
        """Generate HMAC SHA256 signature for Binance API"""
        query_string = urlencode(sorted(params.items()))
        signature = hmac.new(
            self.api_secret.encode('utf-8'),
            query_string.encode('utf-8'),
            hashlib.sha256
        ).hexdigest()
        return signature

    def _make_request(self, method, endpoint, params=None):
        """Make authenticated API request"""
        if params is None:
            params = {}
        
        params['timestamp'] = int(time.time() * 1000)
        params['signature'] = self._generate_signature(params)

        headers = {
            "X-MBX-APIKEY": self.api_key
        }

        try:
            if method.upper() == 'GET':
                response = requests.get(
                    self.base_url + endpoint,
                    headers=headers,
                    params=params,
                    timeout=10
                )
            else:  # POST
                response = requests.post(
                    self.base_url + endpoint,
                    headers=headers,
                    params=params,
                    timeout=10
                )
            
            response.raise_for_status()
            data = response.json()
            
            if 'code' in data and data['code'] != 200:
                raise Exception(f"API Error {data.get('code', 'Unknown')}: {data.get('msg', 'No message')}")
            
            return data
            
        except requests.exceptions.RequestException as e:
            logging.error(f"Network Error: {e}")
            raise Exception(f"Network Failure: {str(e)}")
        except Exception as e:
            logging.error(f"Request Error: {e}")
            raise

    def place_order(self, symbol, side, order_type, quantity, price=None):
        """Place a futures order"""
        endpoint = "/fapi/v1/order"
        
        params = {
            "symbol": symbol.upper(),
            "side": side.upper(),
            "type": order_type.upper(),
            "quantity": f"{quantity:.8f}".rstrip('0').rstrip('.'),
        }
        
        if order_type.upper() == "LIMIT":
            if price is None:
                raise ValueError("Price is required for LIMIT orders")
            params["price"] = f"{price:.2f}"
            params["timeInForce"] = "GTC"

        logging.info(f"Placing order: {params}")

        result = self._make_request('POST', endpoint, params)
        logging.info(f"Order result: {result}")
        return result

    def get_account_info(self):
        """Get account information (for testing API connection)"""
        endpoint = "/fapi/v2/account"
        return self._make_request('GET', endpoint)

# ==================================================
# INPUT VALIDATION
# ==================================================
def validate_inputs(args):
    """Validate command line arguments"""
    if args.side.upper() not in ["BUY", "SELL"]:
        raise ValueError("Side must be BUY or SELL")

    if args.type.upper() not in ["MARKET", "LIMIT"]:
        raise ValueError("Type must be MARKET or LIMIT")

    if args.quantity <= 0:
        raise ValueError("Quantity must be greater than 0")

    if args.type.upper() == "LIMIT" and (args.price is None or args.price <= 0):
        raise ValueError("Price must be greater than 0 for LIMIT orders")

    if len(args.symbol) < 6:
        raise ValueError("Symbol must be valid (e.g., BTCUSDT)")

# ==================================================
# MAIN
# ==================================================
def main():
    parser = argparse.ArgumentParser(
        description="Binance Futures Testnet Trading Bot",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog="""
Examples:
  python bot.py --symbol BTCUSDT --side BUY --type MARKET --quantity 0.001
  python bot.py --symbol BTCUSDT --side SELL --type LIMIT --quantity 0.001 --price 50000
        """
    )

    parser.add_argument("--symbol", required=True, help="Trading pair (e.g., BTCUSDT)")
    parser.add_argument("--side", required=True, choices=["BUY", "SELL"], help="Order side")
    parser.add_argument("--type", required=True, choices=["MARKET", "LIMIT"], help="Order type")
    parser.add_argument("--quantity", type=float, required=True, help="Order quantity")
    parser.add_argument("--price", type=float, help="Order price (required for LIMIT orders)")

    args = parser.parse_args()

    # Check if API credentials are set
    if API_KEY == "YOUR_API_KEY" or API_SECRET == "YOUR_API_SECRET":
        print(" ERROR: Please set your Binance Testnet API credentials in the CONFIGURATION section")
        print("   1. Go to https://testnet.binancefuture.com")
        print("   2. Create account and generate API keys")
        print("   3. Replace YOUR_API_KEY and YOUR_API_SECRET")
        return

    try:
        validate_inputs(args)

        print("\n" + "="*50)
        print(" BINANCE FUTURES TESTNET TRADING BOT")
        print("="*50)
        print(f"Symbol   : {args.symbol.upper()}")
        print(f"Side     : {args.side.upper()}")
        print(f"Type     : {args.type.upper()}")
        print(f"Quantity : {args.quantity}")
        if args.price:
            print(f"Price    : ${args.price:,.2f}")
        print("="*50)

        # Initialize bot
        bot = BinanceFuturesBot(API_KEY, API_SECRET, BASE_URL)

        # Test API connection
        print(" Testing API connection...")
        account_info = bot.get_account_info()
        print(" API connection successful!")

        # Place order
        result = bot.place_order(
            symbol=args.symbol,
            side=args.side,
            order_type=args.type,
            quantity=args.quantity,
            price=args.price
        )

        print("\n ORDER PLACED SUCCESSFULLY!")
        print("\n ORDER DETAILS:")
        print("-" * 30)
        print(f"Order ID     : {result.get('orderId', 'N/A')}")
        print(f"Symbol       : {result.get('symbol', 'N/A')}")
        print(f"Side         : {result.get('side', 'N/A')}")
        print(f"Type         : {result.get('type', 'N/A')}")
        print(f"Status       : {result.get('status', 'N/A')}")
        print(f"Quantity     : {result.get('origQty', 'N/A')}")
        print(f"Executed Qty : {result.get('executedQty', 'N/A')}")
        print(f"Avg Price    : {result.get('avgPrice', 'N/A')}")
        print(f"Price        : {result.get('price', 'N/A')}")

    except KeyboardInterrupt:
        print("\n\n  Bot stopped by user")
    except Exception as e:
        print(f"\n FAILED: {str(e)}")
        logging.error(f"Main execution error: {e}")
        print("\n Check bot.log for detailed logs")
        return 1

if __name__ == "__main__":
    import sys
    sys.exit(main() or 0)
