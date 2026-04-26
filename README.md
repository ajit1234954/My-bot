from flask import Flask, request
import requests, time, hmac, hashlib, json

app = Flask(__name__)

API_KEY = "YOUR_API_KEY"
API_SECRET = "YOUR_SECRET"
BASE_URL = "https://api.delta.exchange"

ACCOUNT_BALANCE = 500
RISK_PER_TRADE = 0.02   # 2% risk = ₹10

# Symbol mapping (Delta product_id check manually once)
SYMBOL_MAP = {
    "BTCUSDT": 27,
    "TAOUSDT": 123
}

def calculate_size(price, sl_percent):
    risk_amount = ACCOUNT_BALANCE * RISK_PER_TRADE
    sl_distance = price * (sl_percent / 100)
    size = risk_amount / sl_distance
    return round(size, 2)

def sign_request(timestamp, method, path, body=""):
    return hmac.new(
        API_SECRET.encode(),
        (timestamp + method + path + body).encode(),
        hashlib.sha256
    ).hexdigest()

def place_order(symbol, side, sl_percent, tp_percent):
    product_id = SYMBOL_MAP[symbol]

    # Dummy price fetch (better: use live price API)
    price = 100  

    size = calculate_size(price, float(sl_percent))

    order = {
        "order_type": "market",
        "size": size,
        "side": side,
        "product_id": product_id
    }

    body = json.dumps(order)
    timestamp = str(int(time.time()))

    signature = sign_request(timestamp, "POST", "/v2/orders", body)

    headers = {
        "api-key": API_KEY,
        "timestamp": timestamp,
        "signature": signature,
        "Content-Type": "application/json"
    }

    r = requests.post(BASE_URL + "/v2/orders", headers=headers, data=body)
    print("Order:", r.json())


@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json

    symbol = data["symbol"]
    signal = data["signal"]
    sl = float(data["sl"].replace("%",""))
    tp = float(data["tp"].replace("%",""))

    place_order(symbol, signal, sl, tp)

    return "trade executed"

app.run(port=5000)
