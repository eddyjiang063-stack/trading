import requests
import time
import hmac
import hashlib
import json
from typing import List, Dict, Optional

# --- API Configuration ---
BASE_URL = "https://mock-api.roostoo.com"
API_KEY = "ttdLE0ZRfWAZeaBoaq8SyEhH6QSQ8yr6laZzTgKNTTYTkogK1cUWdT9P6r59Jd3S"
SECRET_KEY = "8mYHLPiVxrTDx8bTe5UUumVOmZ07DBAE1ZovuxgyJk7ny1oYrDECdWc1GbuMX1fN"


# ------------------------------
# Utility Functions (保持不變)
# ------------------------------

def _get_timestamp():
    """Return a 13-digit millisecond timestamp as string."""
    return str(int(time.time() * 1000))


def _get_signed_headers(payload: dict = {}):
    """
    Generate signed headers and totalParams for RCL_TopLevelCheck endpoints.
    """
    payload['timestamp'] = _get_timestamp()
    sorted_keys = sorted(payload.keys())
    total_params = "&".join(f"{k}={payload[k]}" for k in sorted_keys)

    signature = hmac.new(
        SECRET_KEY.encode('utf-8'),
        total_params.encode('utf-8'),
        hashlib.sha256
    ).hexdigest()

    headers = {
        'RST-API-KEY': API_KEY,
        'MSG-SIGNATURE': signature
    }

    return headers, payload, total_params


# ------------------------------
# API Functions (保持不變)
# ------------------------------

def get_ticker(pair=None):
    """Get ticker for one or all pairs."""
    url = f"{BASE_URL}/v3/ticker"
    params = {'timestamp': _get_timestamp()}
    if pair:
        params['pair'] = pair
    try:
        res = requests.get(url, params=params)
        res.raise_for_status()
        return res.json()
    except requests.exceptions.RequestException as e:
        print(f"Error getting ticker: {e}")
        return None


def get_balance():
    """Get wallet balances."""
    url = f"{BASE_URL}/v3/balance"
    headers, payload, _ = _get_signed_headers({})
    try:
        res = requests.get(url, headers=headers, params=payload)
        res.raise_for_status()
        return res.json()
    except requests.exceptions.RequestException as e:
        print(f"Error getting balance: {e}")
        return None


def place_order(pair_or_coin, side, quantity, price=None, order_type=None):
    """Place a LIMIT or MARKET order."""
    time.sleep(1)
    url = f"{BASE_URL}/v3/place_order"
    pair = f"{pair_or_coin}/USD" if "/" not in pair_or_coin else pair_or_coin

    if order_type is None:
        order_type = "LIMIT" if price is not None else "MARKET"

    if order_type == 'LIMIT' and price is None:
        print("Error: LIMIT orders require 'price'.")
        return None

    payload = {
        'pair': pair,
        'side': side.upper(),
        'type': order_type.upper(),
        'quantity': str(quantity)
    }
    if order_type == 'LIMIT':
        payload['price'] = str(price)

    headers, _, total_params = _get_signed_headers(payload)
    headers['Content-Type'] = 'application/x-www-form-urlencoded'

    try:
        res = requests.post(url, headers=headers, data=total_params)
        res.raise_for_status()
        return res.json()
    except requests.exceptions.RequestException as e:
        print(f"Error placing order: {e}")
        return None


def query_order(order_id=None, pair=None, pending_only=None):
    """Query order history or pending orders."""
    url = f"{BASE_URL}/v3/query_order"
    payload = {}
    if order_id:
        payload['order_id'] = str(order_id)
    elif pair:
        payload['pair'] = pair
        if pending_only is not None:
            payload['pending_only'] = 'TRUE' if pending_only else 'FALSE'

    headers, _, total_params = _get_signed_headers(payload)
    headers['Content-Type'] = 'application/x-www-form-urlencoded'

    try:
        res = requests.post(url, headers=headers, data=total_params)
        res.raise_for_status()
        return res.json()
    except requests.exceptions.RequestException as e:
        print(f"Error querying order: {e}")
        return None


# ------------------------------
# 趨勢跟隨交易策略類
# ------------------------------

class TrendFollowingStrategy:
    def __init__(self, pair: str = "BTC/USD", initial_balance: float = 1000):
        self.pair = pair
        self.base_currency = pair.split('/')[0]  # BTC
        self.quote_currency = pair.split('/')[1]  # USD

        # 策略參數
        self.fast_ma_period = 5  # 快速均線週期
        self.slow_ma_period = 20  # 慢速均線週期
        self.rsi_period = 10  # RSI週期
        self.take_profit = 0.03  # 止盈 3%
        self.stop_loss = 0.03  # 止損 3%

        # 交易參數
        self.trade_amount_usd = 100  # 每次交易金額 (USD)
        self.max_position = 0.1  # 最大持倉比例 10%

        # 狀態變量
        self.position = 0.0  # 當前持倉數量
        self.entry_price = 0.0  # 入場價格
        self.trade_history = []  # 交易記錄
        self.price_history = []  # 價格歷史
        self.balance_history = []  # 資金曲線

        print(f"初始化趨勢跟隨策略 - 交易對: {self.pair}")
        print(f"均線參數: 快線{self.fast_ma_period}期, 慢線{self.slow_ma_period}期")
        print(f"風險控制: 止盈{self.take_profit * 100}%, 止損{self.stop_loss * 100}%")

    def get_current_price(self) -> Optional[float]:
        """獲取當前價格"""
        ticker = get_ticker(self.pair)
        if ticker:
            return ticker.get('Data',{}).get('BTC/USD',{}).get('LastPrice')
        print(f"無法獲取 {self.pair} 的價格數據")
        return None

    def update_price_history(self, price: float):
        """更新價格歷史數據"""
        self.price_history.append(price)
        # 保持歷史數據長度
        max_history = max(self.slow_ma_period * 2, 100)
        if len(self.price_history) > max_history:
            self.price_history = self.price_history[-max_history:]

    def calculate_sma(self, period: int) -> Optional[float]:
        """計算簡單移動平均線"""
        if len(self.price_history) < period:
            return None
        return sum(self.price_history[-period:]) / period

    def calculate_ema(self, period: int) -> Optional[float]:
        """計算指數移動平均線"""
        if len(self.price_history) < period:
            return None

        prices = self.price_history[-period * 2:]  # 使用更多數據計算
        alpha = 2 / (period + 1)
        ema = prices[0]

        for price in prices[1:]:
            ema = alpha * price + (1 - alpha) * ema

        return ema

    def calculate_rsi(self, period: int) -> Optional[float]:
        """計算RSI指標"""
        if len(self.price_history) < period + 1:
            return None

        gains = []
        losses = []

        for i in range(len(self.price_history) - period, len(self.price_history) - 1):
            change = self.price_history[i + 1] - self.price_history[i]
            if change > 0:
                gains.append(change)
                losses.append(0)
            else:
                gains.append(0)
                losses.append(abs(change))

        avg_gain = sum(gains) / period
        avg_loss = sum(losses) / period

        if avg_loss == 0:
            return 100

        rs = avg_gain / avg_loss
        rsi = 100 - (100 / (1 + rs))
        return rsi

    def get_trading_signal(self) -> Dict[str, any]:
        """獲取交易信號"""
        if len(self.price_history) < self.slow_ma_period:
            return {"action": "HOLD", "reason": "數據不足"}

        # 計算技術指標
        fast_ma = self.calculate_ema(self.fast_ma_period)
        slow_ma = self.calculate_ema(self.slow_ma_period)
        rsi = self.calculate_rsi(self.rsi_period)
        current_price = self.price_history[-1]

        if not all([fast_ma, slow_ma, rsi]):
            return {"action": "HOLD", "reason": "指標計算失敗"}

        signal = {
            "current_price": current_price,
            "fast_ma": fast_ma,
            "slow_ma": slow_ma,
            "rsi": rsi,
            "action": "HOLD",
            "reason": ""
        }

        # 多頭信號：快線上穿慢線且RSI不是極度超買
        if (fast_ma > slow_ma and
                self.price_history[-2] <= slow_ma and  # 上一根K線在慢線下方
                rsi < 80):  # RSI不是極度超買

            signal["action"] = "BUY"
            signal["reason"] = f"快線上穿慢線, RSI={rsi:.1f}"

        # 空頭信號：快線下穿慢線且RSI不是極度超賣
        elif (fast_ma < slow_ma and
              self.price_history[-2] >= slow_ma and  # 上一根K線在慢線上方
              rsi > 20):  # RSI不是極度超賣

            signal["action"] = "SELL"
            signal["reason"] = f"快線下穿慢線, RSI={rsi:.1f}"

        # 持倉時的止盈止損檢查
        elif self.position > 0:  # 多頭持倉
            profit_ratio = (current_price - self.entry_price) / self.entry_price

            if profit_ratio >= self.take_profit:
                signal["action"] = "SELL"
                signal["reason"] = f"達到止盈條件, 盈利{profit_ratio * 100:.1f}%"
            elif profit_ratio <= -self.stop_loss:
                signal["action"] = "SELL"
                signal["reason"] = f"達到止損條件, 虧損{abs(profit_ratio) * 100:.1f}%"

        elif self.position < 0:  # 空頭持倉 (如果支持做空)
            profit_ratio = (self.entry_price - current_price) / self.entry_price

            if profit_ratio >= self.take_profit:
                signal["action"] = "BUY"  # 平空倉
                signal["reason"] = f"達到止盈條件, 盈利{profit_ratio * 100:.1f}%"
            elif profit_ratio <= -self.stop_loss:
                signal["action"] = "BUY"  # 平空倉
                signal["reason"] = f"達到止損條件, 虧損{abs(profit_ratio) * 100:.1f}%"

        return signal

    def calculate_position_size(self, price: float) -> float:
        """計算倉位大小"""
        # 獲取賬戶餘額
        balance_info = get_balance()
        if not balance_info:
            return 0

        # 計算可用資金 (這裡需要根據實際API響應調整)
        available_usd = 1000  # 默認值，實際應該從balance_info中解析

        # 根據風險管理計算倉位
        max_trade_usd = available_usd * self.max_position
        trade_usd = min(self.trade_amount_usd, max_trade_usd)

        # 計算交易數量
        quantity = trade_usd / price
        return quantity

    def execute_trade(self, signal: Dict[str, any]):
        """執行交易"""
        action = signal["action"]
        current_price = signal["current_price"]

        if action == "HOLD":
            return

        quantity = self.calculate_position_size(current_price)
        if quantity <= 0:
            print("倉位計算失敗，跳過交易")
            return

        # 執行交易
        if action == "BUY" and self.position <= 0:
            order = place_order(self.pair, "BUY", quantity)
            if order and 'order_id' in order:
                self.position = quantity
                self.entry_price = current_price
                trade_record = {
                    "time": time.time(),
                    "action": "BUY",
                    "price": current_price,
                    "quantity": quantity,
                    "reason": signal["reason"]
                }
                self.trade_history.append(trade_record)
                print(f"✅ 買入: {quantity:.6f} {self.pair} @ {current_price:.2f}")
                print(f"   原因: {signal['reason']}")

        elif action == "SELL" and self.position > 0:
            order = place_order(self.pair, "SELL", self.position)
            if order and 'order_id' in order:
                profit_loss = (current_price - self.entry_price) * self.position
                profit_ratio = (current_price - self.entry_price) / self.entry_price

                self.position = 0
                trade_record = {
                    "time": time.time(),
                    "action": "SELL",
                    "price": current_price,
                    "quantity": quantity,
                    "profit_loss": profit_loss,
                    "profit_ratio": profit_ratio,
                    "reason": signal["reason"]
                }
                self.trade_history.append(trade_record)
                print(f"✅ 賣出: {quantity:.6f} {self.pair} @ {current_price:.2f}")
                print(f"   盈虧: {profit_loss:.2f} USD ({profit_ratio * 100:.2f}%)")
                print(f"   原因: {signal['reason']}")

    def print_status(self, signal: Dict[str, any]):
        """打印當前狀態"""
        print(f"\n📊 策略狀態 - {time.strftime('%Y-%m-%d %H:%M:%S')}")
        print(f"   當前價格: {signal['current_price']:.2f}")
        print(f"   快線EMA({self.fast_ma_period}): {signal['fast_ma']:.2f}")
        print(f"   慢線EMA({self.slow_ma_period}): {signal['slow_ma']:.2f}")
        print(f"   RSI({self.rsi_period}): {signal['rsi']:.1f}")
        print(f"   持倉: {self.position:.6f} {self.pair}")
        if self.position > 0:
            unrealized_pnl = (signal['current_price'] - self.entry_price) * self.position
            unrealized_ratio = (signal['current_price'] - self.entry_price) / self.entry_price
            print(f"   浮動盈虧: {unrealized_pnl:.2f} USD ({unrealized_ratio * 100:.2f}%)")
        print(f"   信號: {signal['action']} - {signal['reason']}")
        print("-" * 50)

    def run_strategy(self, check_interval: int = 300):  # 5分鐘檢查一次
        """運行策略"""
        count=0
        print(f"🚀 啟動趨勢跟隨策略，檢查間隔: {check_interval}秒")

        # 初始數據收集
        print("📈 收集初始價格數據...")
        for _ in range(self.slow_ma_period):
            price = self.get_current_price()
            print(price)
            if price:
                self.update_price_history(price)
            time.sleep(1)

        print("✅ 數據收集完成，開始策略執行...")

        while True:
            try:
                # 獲取當前價格
                current_price = self.get_current_price()
                if not current_price:
                    print("❌ 無法獲取價格，等待重試...")
                    time.sleep(check_interval)
                    continue

                # 更新價格歷史
                self.update_price_history(current_price)

                # 獲取交易信號
                signal = self.get_trading_signal()

                # 打印狀態
                count+=1
                print()
                print(f"第{count}次檢查")
                self.print_status(signal)

                # 執行交易
                self.execute_trade(signal)
                print("\n-----getting account balance-----")
                print(get_balance())

                # 等待下一次檢查
                print(f"⏰ 等待 {check_interval} 秒後再次檢查...")
                time.sleep(check_interval)

            except KeyboardInterrupt:
                print("\n🛑 手動停止策略")
                break
            except Exception as e:
                print(f"❌ 策略執行錯誤: {e}")
                time.sleep(check_interval)


# ------------------------------
# 主程序
# ------------------------------

if __name__ == "__main__":
    # 創建策略實例
    strategy = TrendFollowingStrategy(
        pair="BTC/USD",
        initial_balance=1000
    )

    # 運行策略 (每5分鐘檢查一次)
    strategy.run_strategy(check_interval=300)
