import pandas as pd
import numpy as np
import yfinance as yf
from flask import Flask, request, jsonify
import os
from dotenv import load_dotenv

# Load environment variables
load_dotenv()

app = Flask(__name__)

class PaperTradingAgent:
    def __init__(self, symbol, initial_balance):
        self.symbol = symbol
        self.initial_balance = initial_balance
        self.balance = initial_balance
        self.positions = {}
        self.trade_history = []

    def get_historical_data(self):
        data = yf.download(self.symbol, period='1y')
        return data

    def calculate_indicators(self, data):
        data['SMA_50'] = data['Close'].rolling(window=50).mean()
        data['SMA_200'] = data['Close'].rolling(window=200).mean()
        return data.dropna()  # Remove NaN values

    def generate_signals(self, data):
        data['Signal'] = np.where(data['SMA_50'] > data['SMA_200'], 1, -1)
        return data['Signal'].tolist()

    def execute_trade(self, action, close_price):
        if action == "BUY" and self.balance >= close_price:
            shares_to_buy = int(self.balance / close_price)
            if shares_to_buy > 0:
                self.positions[self.symbol] = shares_to_buy
                self.balance -= shares_to_buy * close_price
                self.trade_history.append(("BUY", shares_to_buy, close_price))
                print(f'Buy {shares_to_buy} shares of {self.symbol} at ${close_price:.2f}')
        elif action == "SELL" and self.symbol in self.positions:
            shares_to_sell = self.positions[self.symbol]
            self.balance += shares_to_sell * close_price
            del self.positions[self.symbol]
            self.trade_history.append(("SELL", shares_to_sell, close_price))
            print(f'Sell {shares_to_sell} shares of {self.symbol} at ${close_price:.2f}')

    def run_backtest(self):
        print(f'Running backtest for {self.symbol}')
        historical_data = self.get_historical_data()
        indicator_data = self.calculate_indicators(historical_data)
        signals = self.generate_signals(indicator_data)
        for i, signal in enumerate(signals):
            close_price = indicator_data['Close'].iloc[i].item()
            if signal == 1:
                self.execute_trade("BUY", close_price)
            elif signal == -1:
                self.execute_trade("SELL", close_price)
        print(f'Final Balance: ${self.balance:.2f}')

# Load configuration from environment variables
SYMBOL = os.getenv("SYMBOL", "BTC-USD")
INITIAL_BALANCE = float(os.getenv("INITIAL_BALANCE", 10000))

agent = PaperTradingAgent(SYMBOL, INITIAL_BALANCE)

@app.route('/webhook', methods=['POST'])
def webhook():
    data = request.json
    print("Received Webhook:", data)
    if "action" in data and "close_price" in data:
        action = data["action"].upper()
        close_price = float(data["close_price"])
        agent.execute_trade(action, close_price)
        return jsonify({"message": f"Executed {action} at {close_price}"}), 200
    return jsonify({"error": "Invalid data"}), 400

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=5000)
