import streamlit as st
import pandas as pd
import numpy as np
import plotly.graph_objects as go
from datetime import datetime, timedelta

# --- 1. పేజ్ సెటప్ (UI Design) ---
st.set_page_config(page_title="Quotex Pattern Analyzer", layout="wide")
st.title("📈 Quotex - Candlestick Pattern Analyzer")
st.markdown("ఈ డాష్‌బోర్డ్ జపనీస్ క్యాండిల్‌స్టిక్ డేటాను విశ్లేషించి **Bullish / Bearish Engulfing** ప్యాటర్న్స్‌ను గుర్తిస్తుంది.")

# --- 2. శాంపిల్ డేటా జనరేషన్ (Quotex డేటాకు బదులుగా) ---
@st.cache_data
def generate_sample_data():
    np.random.seed(42)
    dates = [datetime.now() - timedelta(minutes=i*5) for i in range(50, 0, -1)]
    
    # ట్రెండ్ లాగా కనిపించడానికి రాండమ్ వాక్ (Random Walk) వాడుతున్నాము
    close_prices = 100 + np.random.randn(50).cumsum()
    open_prices = close_prices + np.random.randn(50) * 0.5
    high_prices = np.maximum(open_prices, close_prices) + np.random.rand(50) * 1.5
    low_prices = np.minimum(open_prices, close_prices) - np.random.rand(50) * 1.5
    
    df = pd.DataFrame({
        'Date': dates, 'Open': open_prices, 'High': high_prices, 
        'Low': low_prices, 'Close': close_prices
    })
    
    # ఎన్‌గల్ఫింగ్ ప్యాటర్న్స్ క్రియేట్ అయ్యేలా ఒక రెండు క్యాండిల్స్‌ను మాన్యువల్‌గా సెట్ చేద్దాం (టెస్టింగ్ కోసం)
    df.loc[40, ['Open', 'Close']] = [105, 102] # Bearish
    df.loc[41, ['Open', 'Close', 'High', 'Low']] = [101, 106, 107, 100] # Bullish Engulfing
    
    return df

# --- 3. ఎన్‌గల్ఫింగ్ లాజిక్ ---
def detect_patterns(df):
    data = df.copy()
    data['Prev_Open'] = data['Open'].shift(1)
    data['Prev_Close'] = data['Close'].shift(1)
    
    is_current_bullish = data['Close'] > data['Open']
    is_current_bearish = data['Close'] < data['Open']
    is_prev_bullish = data['Prev_Close'] > data['Prev_Open']
    is_prev_bearish = data['Prev_Close'] < data['Prev_Open']
    
    # Bullish Engulfing
    data['Bullish_Engulfing'] = (
        is_prev_bearish & is_current_bullish & 
        (data['Open'] <= data['Prev_Close']) & (data['Close'] >= data['Prev_Open'])
    )
    
    # Bearish Engulfing
    data['Bearish_Engulfing'] = (
        is_prev_bullish & is_current_bearish & 
        (data['Open'] >= data['Prev_Close']) & (data['Close'] <= data['Prev_Open'])
    )
    return data

# --- 4. డేటా ప్రాసెసింగ్ ---
df = generate_sample_data()
df = detect_patterns(df)

# --- 5. చార్ట్ డిజైన్ (Plotly) ---
fig = go.Figure(data=[go.Candlestick(
    x=df['Date'], open=df['Open'], high=df['High'], 
    low=df['Low'], close=df['Close'], name="Candlesticks"
)])

# చార్ట్ మీద ఎన్‌గల్ఫింగ్ సిగ్నల్స్ చూపించడం
bullish_signals = df[df['Bullish_Engulfing']]
if not bullish_signals.empty:
    fig.add_trace(go.Scatter(
        x=bullish_signals['Date'], y=bullish_signals['Low'] - 1,
        mode='markers', marker=dict(symbol='triangle-up', size=15, color='green'),
        name='BUY Signal (Bullish Engulfing)'
    ))

bearish_signals = df[df['Bearish_Engulfing']]
if not bearish_signals.empty:
    fig.add_trace(go.Scatter(
        x=bearish_signals['Date'], y=bearish_signals['High'] + 1,
        mode='markers', marker=dict(symbol='triangle-down', size=15, color='red'),
        name='SELL Signal (Bearish Engulfing)'
    ))

fig.update_layout(
    xaxis_rangeslider_visible=False, 
    template="plotly_dark", # డార్క్ థీమ్ (Quotex లాగా)
    height=600,
    margin=dict(l=20, r=20, t=20, b=20)
)

# --- 6. అప్లికేషన్‌లో చూపించడం ---
st.plotly_chart(fig, use_container_width=True)

st.subheader("📊 సిగ్నల్స్ డేటా టేబుల్")
# కేవలం సిగ్నల్స్ వచ్చిన డేటాను మాత్రమే ఫిల్టర్ చేసి టేబుల్‌లో చూపించడం
signals_df = df[(df['Bullish_Engulfing'] == True) | (df['Bearish_Engulfing'] == True)]
st.dataframe(signals_df[['Date', 'Open', 'High', 'Low', 'Close', 'Bullish_Engulfing', 'Bearish_Engulfing']])
