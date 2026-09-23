# japan-shodou-scanner

import yfinance as yf
import pandas as pd
from datetime import datetime, timedelta

# とりあえずテスト用に数銘柄（あとで全銘柄に拡張）
TICKERS = ["7203.T", "9984.T", "6758.T"]

DAYS = 30
VOL_WINDOW = 10
SHORT_MA = 5
MID_MA = 25
MIN_PCT_CHANGE = 3
VOL_SPIKE_RATIO = 2.0

def fetch_price(ticker):
    end = datetime.now()
    start = end - timedelta(days=DAYS*2)
    df = yf.download(ticker, start=start, end=end, progress=False)
    if df.empty:
        return df
    return df.tail(DAYS)

def analyze_ticker(ticker):
    df = fetch_price(ticker)
    if df.empty or len(df) < VOL_WINDOW + 2:
        return None

    df["MA_short"] = df["Close"].rolling(SHORT_MA).mean()
    df["MA_mid"] = df["Close"].rolling(MID_MA).mean()
    df["VOL_MA"] = df["Volume"].rolling(VOL_WINDOW).mean()

    latest = df.iloc[-1]
    prev = df.iloc[-2]

    pct_change = (latest["Close"] - prev["Close"]) / prev["Close"] * 100
    vol_ratio = latest["Volume"] / latest["VOL_MA"] if latest["VOL_MA"] else 0

    gcross = prev["MA_short"] <= prev["MA_mid"] and latest["MA_short"] > latest["MA_mid"]

    if pct_change >= MIN_PCT_CHANGE and vol_ratio >= VOL_SPIKE_RATIO and gcross:
        return {
            "ticker": ticker,
            "close": latest["Close"],
            "pct_change": round(pct_change, 2),
            "vol_ratio": round(vol_ratio, 2),
        }
    return None

def main():
    results = []
    for t in TICKERS:
        r = analyze_ticker(t)
        if r:
            results.append(r)

    if not results:
        print("初動候補なし")
        return

    df = pd.DataFrame(results)
    df = df.sort_values("pct_change", ascending=False)
    print(df.to_string(index=False))

if __name__ == "__main__":
    main()
