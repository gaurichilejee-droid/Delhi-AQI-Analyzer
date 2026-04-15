# Delhi-AQI-Analyzer
import streamlit as st
import pandas as pd
import matplotlib.pyplot as plt
import base64

# ---------------- PAGE CONFIG ----------------
st.set_page_config(page_title="Delhi AQI Dashboard", layout="wide")

# ---------------- LOAD DATA ----------------
df = pd.read_csv("final_dataset.csv")

# ---------------- BACKGROUND FUNCTION ----------------
def set_bg(image_file):
    with open(image_file, "rb") as f:
        encoded = base64.b64encode(f.read()).decode()

    st.markdown(f"""
    <style>
    [data-testid="stAppViewContainer"] {{
        background: linear-gradient(rgba(0,0,0,0.6), rgba(0,0,0,0.6)),
        url("data:image/jpeg;base64,{encoded}");
        background-size: cover;
        background-position: center;
        background-attachment: fixed;
    }}
    </style>
    """, unsafe_allow_html=True)

# Apply background
set_bg("bg.jpeg")

# ---------------- GLOBAL CSS ----------------
st.markdown("""
<style>

/* Remove padding */
.block-container {
    padding-top: 1rem;
    padding-bottom: 0rem;
}

/* Sidebar glass effect */
section[data-testid="stSidebar"] {
    background: rgba(255,255,255,0.08);
    backdrop-filter: blur(10px);
}

/* Center layout */
.main-container {
    height: 15vh;   /* FIXED: was 10vh causing scroll issue */
    display: flex;
    flex-direction: column;
    justify-content: center;
    align-items: center;
}

/* Text styles */
.city {
    font-size: 30px;
    color: white;
    opacity: 0.8;
}

.aqi-value {
    font-size: 120px;
    font-weight: bold;
    color: white;
    margin: 0;
}

.label {
    font-size: 16px;
    color: #ddd;
}

/* Stats row */
.stats-row {
    display: flex;
    gap: 20px;
    margin-top: 30px;
}

/* Cards */
.stat-card {
    background: rgba(255,255,255,0.1);
    padding: 15px 25px;
    border-radius: 12px;
    text-align: center;
    color: white;
}

/* AQI category card */
.aqi-card {
    background: rgba(255,255,255,0.2);
    padding: 15px;
    border-radius: 12px;
    text-align: center;
    color: white;
    font-weight: 600;
}

</style>
""", unsafe_allow_html=True)

# ---------------- NAVIGATION ----------------
page = st.sidebar.radio("Navigation", ["Home", "Dashboard"])

# ---------------- HOME PAGE ----------------
if page == "Home":

    latest = df.iloc[-1]

    aqi = int(latest["AQI"])
    pm25 = int(latest["PM2.5"])
    pm10 = int(latest["PM10"])
    no2 = int(latest["NO2"])

    st.markdown('<div class="main-container">', unsafe_allow_html=True)

    st.markdown('<div class="city">Delhi</div>', unsafe_allow_html=True)
    st.markdown(f'<div class="aqi-value">{aqi}</div>', unsafe_allow_html=True)
    st.markdown('<div class="label">Air Quality Index</div>', unsafe_allow_html=True)

    st.markdown('<div class="stats-row">', unsafe_allow_html=True)

    st.markdown(f"""
    <div class="stat-card">
        <div>PM2.5</div>
        <div>{pm25}</div>
    </div>
    """, unsafe_allow_html=True)

    st.markdown(f"""
    <div class="stat-card">
        <div>PM10</div>
        <div>{pm10}</div>
    </div>
    """, unsafe_allow_html=True)

    st.markdown(f"""
    <div class="stat-card">
        <div>NO₂</div>
        <div>{no2}</div>
    </div>
    """, unsafe_allow_html=True)

    st.markdown('</div></div>', unsafe_allow_html=True)

# ---------------- DASHBOARD ----------------
elif page == "Dashboard":

    st.title("📊 Delhi Air Quality Dashboard")

    # Create datetime column
    df["FullDate"] = pd.to_datetime(
        dict(year=df["Year"], month=df["Month"], day=df["Date"])
    )

    # ---------------- FILTERS ----------------
    st.sidebar.header("Filters")

    months = sorted(df["Month"].unique())

    selected_months = st.sidebar.multiselect(
        "Select Months",
        months,
        default=months
    )

    pollutant = st.sidebar.selectbox(
        "Select Pollutant",
        ["PM2.5","PM10","NO2","SO2","CO","Ozone","AQI"]
    )

    # ---------------- FILTER DATA ----------------
    filtered_df = df[df["Month"].isin(selected_months)]

    # ---------------- METRICS ----------------
    avg = round(filtered_df[pollutant].mean(), 2)
    maxv = filtered_df[pollutant].max()

    col1, col2 = st.columns(2)
    col1.metric(f"Average {pollutant}", avg)
    col2.metric(f"Max {pollutant}", maxv)

    # ---------------- AQI CATEGORY ----------------
    def get_aqi_category(aqi):
        if aqi <= 50:
            return "🟢 Good","green","Air quality is satisfactory"
        elif aqi <= 100:
            return "🟡 Moderate","yellow","Limit outdoor activity"
        elif aqi <= 200:
            return "🟠 Poor","orange","Breathing discomfort possible"
        elif aqi <= 300:
            return "🔴 Very Poor","red","Avoid outdoor activity"
        else:
            return "🟣 Severe","purple","Health emergency"

    if pollutant == "AQI":
        cat, color, advice = get_aqi_category(avg)
        st.markdown(
            f'<div class="aqi-card">{cat}<br><small>{advice}</small></div>',
            unsafe_allow_html=True
        )

    # ---------------- CHART ----------------
    fig, ax = plt.subplots(figsize=(12,4))

    if pollutant == "AQI":
        for i in range(len(filtered_df)-1):
            val = filtered_df["AQI"].iloc[i]
            _, color, _ = get_aqi_category(val)

            ax.plot(
                filtered_df["FullDate"].iloc[i:i+2],
                filtered_df["AQI"].iloc[i:i+2],
                color=color,
                linewidth=2
            )
    else:
        ax.plot(filtered_df["FullDate"], filtered_df[pollutant], linewidth=2)

    ax.grid(True, alpha=0.3)
    plt.xticks(rotation=45)
    st.pyplot(fig)

    # ---------------- TABLE ----------------
    st.markdown("### 📋 Data")
    st.dataframe(filtered_df.sort_values("FullDate", ascending=False))

