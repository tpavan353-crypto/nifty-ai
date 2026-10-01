# nifty-ai
import streamlit as st
import pandas as pd
from nsepython import nse_optionchain_scrapper
import google.generativeai as genai
import time

# --- CONFIGURATION ---
st.set_page_config(page_title="Nifty AI Analyst", layout="wide")

# Fetch API Key from Streamlit Secrets
GEMINI_API_KEY = st.secrets["GEMINI_API_KEY"]

genai.configure(api_key=GEMINI_API_KEY)
model = genai.GenerativeModel('gemini-1.5-flash')

def get_oi_data():
    try:
        payload = nse_optionchain_scrapper("NIFTY")
        data = payload['filtered']['data']
        price = payload['records']['underlyingValue']
        rows = [{"Strike": s['strikePrice'], "CE_OI": s['CE']['openInterest'], 
                 "CE_CHG": s['CE']['changeinOpenInterest'], "PE_OI": s['PE']['openInterest'], 
                 "PE_CHG": s['PE']['changeinOpenInterest']} for s in data]
        df = pd.DataFrame(rows)
        pcr = df['PE_OI'].sum() / df['CE_OI'].sum()
        return df, price, pcr
    except Exception as e:
        st.error(f"NSE Error: {e}")
        return None, None, None

st.title("🤖 Nifty Real-Time AI Analyst")

# Layout
df, price, pcr = get_oi_data()
if df is not None:
    c1, c2 = st.columns(2)
    c1.metric("Nifty Spot", price)
    c2.metric("PCR", round(pcr, 2))

    # AI Analysis
    atm = round(price / 50) * 50
    subset = df[(df['Strike'] >= atm - 150) & (df['Strike'] <= atm + 150)]
    
    if st.button("Generate AI Insight"):
        prompt = f"Nifty at {price}, PCR {pcr:.2f}. Data: {subset.to_string()}. Analyze sentiment briefly."
        response = model.generate_content(prompt)
        st.info(response.text)

    st.dataframe(df)

time.sleep(180)
st.rerun()