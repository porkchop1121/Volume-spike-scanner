import yfinance as yf
import pandas as pd
import requests
import streamlit as st
from datetime import datetime, timedelta

# === CONFIG ===
NEWS_API_KEY = st.secrets["NEWS_API_KEY"]
VOLUME_THRESHOLD = 1.20
LOOKBACK_DAYS = 60

@st.cache_data(ttl=3600)
def fetch_stock_data(ticker):
    end = datetime.now()
    start = end - timedelta(days=LOOKBACK_DAYS)
    df = yf.download(ticker, start=start.strftime('%Y-%m-%d'), progress=False)
    return df

def check_volume_spike(df):
    if df.empty or len(df) < 51:
        return False, 0, 0, 0, 0
    df = df.tail(60)
    today_vol = df['Volume'].iloc[-1]
    today_close = df['Close'].iloc[-1]
    avg10_vol = df['Volume'].iloc[-11:-1].mean()
    avg50_vol = df['Volume'].iloc[-51:-1].mean()
    avg10_close = df['Close'].iloc[-11:-1].mean()
    spike10 = today_vol / avg10_vol
    spike50 = today_vol / avg50_vol
    price_change = today_close / avg10_close
    if spike10 > VOLUME_THRESHOLD and spike50 > VOLUME_THRESHOLD:
        return True, today_vol, spike10, spike50, price_change
    return False, today_vol, spike10, spike50, price_change

def get_news_headlines(ticker):
    url = f"https://newsapi.org/v2/everything?q={ticker}&sortBy=publishedAt&pageSize=3&apiKey={NEWS_API_KEY}"
    r = requests.get(url)
    if r.status_code != 200:
        return []
    data = r.json()
    return [article['title'] for article in data.get('articles', [])]

def scan_tickers(tickers):
    results = []
    for ticker in tickers:
        df = fetch_stock_data(ticker)
        spike, vol, spike10, spike50, price_change = check_volume_spike(df)
        if spike:
            news = get_news_headlines(ticker)
            results.append({
                "Ticker": ticker,
                "Volume": vol,
                "10D Spike": round(spike10, 2),
                "50D Spike": round(spike50, 2),
                "Price Change": round((price_change - 1) * 100, 2),
                "Direction": "UP" if price_change > 1 else "DOWN",
                "News": news
            })
    return results

# === STREAMLIT APP ===

st.title("📈 Volume Spike Scanner (US & Canada)")
tickers_input = st.text_area("Enter tickers (comma-separated):", "AAPL,MSFT,SHOP.TO,RY.TO")
tickers = [t.strip().upper() for t in tickers_input.split(",") if t.strip()]

if st.button("🔍 Scan Market"):
    st.info("Scanning...")
    results = scan_tickers(tickers)
    if results:
        for res in results:
            color = "🟢" if res['Direction'] == "UP" else "🔴"
            st.subheader(f"{color} {res['Ticker']}")
            st.write(f"**Volume**: {res['Volume']:,}")
            st.write(f"**10D Spike**: {res['10D Spike']}x")
            st.write(f"**50D Spike**: {res['50D Spike']}x")
            st.write(f"**Price Change**: {res['Price Change']}%")
            st.markdown("**📰 News Headlines:**")
            for news in res['News']:
                st.markdown(f"- {news}")
            st.markdown("---")
    else:
        st.success("✅ No volume spikes detected.")
