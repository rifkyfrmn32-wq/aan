import math
from urllib.parse import quote

import pandas as pd
import plotly.express as px
import streamlit as st
from streamlit_autorefresh import st_autorefresh

# ================= KONFIGURASI =================
SHEET_ID = "1GD92oGOfp3EosYyz8_FFwqjVqK6t8lOSZmJMqPsMpqE"  # ganti dengan ID spreadsheet Anda
SHEET_NAME = "Monitoring Realtime"

st.set_page_config(page_title="Dashboard Monitoring Listrik", page_icon="⚡", layout="wide")


# ================= BAGIAN DATA =================
def csv_url(sheet_name: str) -> str:
    """Link export CSV dari Google Sheets (tanpa API key)."""
    return (f"https://docs.google.com/spreadsheets/d/{SHEET_ID}"
            f"/gviz/tq?tqx=out:csv&sheet={quote(sheet_name)}")


@st.cache_data(ttl=20)
def load_raw() -> pd.DataFrame:
    return pd.read_csv(csv_url(SHEET_NAME))


def find_col(df: pd.DataFrame, *keys: str):
    """Cari kolom berdasarkan kata kunci (kebal terhadap perbedaan spasi/huruf besar)."""
    for col in df.columns:
        name = str(col).lower()
        if all(k in name for k in keys):
            return col
    return None


def parse_num(series: pd.Series) -> pd.Series:
    """Angka format Indonesia (239,46 atau 309.924,84) -> float."""
    s = series.astype(str).str.strip().str.replace(".", "", regex=False)
    s = s.str.replace(",", ".", regex=False)
    return pd.to_numeric(s, errors="coerce")


def prepare(raw: pd.DataFrame) -> pd.DataFrame:
    mapping = {
        "waktu": ("waktu",),
        "vll1": ("vll-1",), "vll2": ("vll-2",), "vll3": ("vll-3",),
        "i1": ("i-1",), "i2": ("i-2",), "i3": ("i-3",),
        "pf": ("pf",),
        "kwh": ("kwh akum",),
    }
    cols, missing = {}, []
    for std, keys in mapping.items():
        c = find_col(raw, *keys)
        if c is None:
            missing.append(std)
        else:
            cols[std] = c
    if missing:
        raise ValueError(f"Kolom tidak ditemukan: {missing}. Kolom yang ada: {list(raw.columns)}")

    df = raw[list(cols.values())].copy()
    df.columns = list(cols.keys())
    df["waktu"] = pd.to_datetime(df["waktu"], dayfirst=True, errors="coerce")
    for c in df.columns:
        if c != "waktu":
            df[c] = parse_num(df[c])

    df = df.dropna(subset=["waktu"]).sort_values("waktu").reset_index(drop=True)
    df["v_avg"] = df[["vll1", "vll2", "vll3"]].mean(axis=1)
    df["i_avg"] = df[["i1", "i2", "i3"]].mean(axis=1)
    # Estimasi daya 3 fasa: P = √3 × V × I × PF
    df["p_kw"] = math.sqrt(3) * df["v_avg"] * df["i_avg"] * df["pf"] / 1000
    return df


def daily_recap(df: pd.DataFrame) -> pd.DataFrame:
    daily = df.groupby(df["waktu"].dt.date).agg(
        v_avg=("v_avg", "mean"), v_min=("v_avg", "min"), v_max=("v_avg", "max"),
        i_avg=("i_avg", "mean"), i_max=("i_avg", "max"),
        pf_avg=("pf", "mean"), p_avg=("p_kw", "mean"),
        kwh_awal=("kwh", "min"), kwh_akhir=("kwh", "max"),
        jumlah=("waktu", "count"),
    ).reset_index().rename(columns={"waktu": "tanggal"})
    daily["kwh_pakai"] = daily["kwh_akhir"] - daily["kwh_awal"]
    return daily


def fmt(x, pattern="{:.2f}"):
    return "-" if pd.isna(x) else pattern.format(x)


# ================= SIDEBAR =================
with st.sidebar:
    st.header("⚙️ Pengaturan")
    refresh = st.slider("Auto-refresh (detik)", 10, 300, 60)
    window = st.slider("Jumlah data realtime", 20, 500, 100)
    st.caption("Data diambil dari Google Spreadsheet (sheet 'Monitoring Realtime').")

st_autorefresh(interval=refresh * 1000, key="auto_refresh")

# ================= LOAD DATA =================
try:
    df = prepare(load_raw())
except Exception as e:
    st.error(f"Gagal membaca data: {e}")
    st.info("Pastikan spreadsheet sudah dibagikan (Anyone with the link → Viewer) "
            "dan nama sheet 'Monitoring Realtime' sudah benar.")
    st.stop()

if df.empty:
    st.warning("Belum ada data di spreadsheet.")
    st.stop()

# ================= HEADER & KPI =================
last = df.iloc[-1]
st.title("⚡ Dashboard Monitoring Daya Listrik")
st.caption(f"Data terakhir: {last.waktu:%d/%m/%Y %H:%M:%S}  •  Total data: {len(df)}")

k1, k2, k3, k4 = st.columns(4)
k1.metric("Tegangan rata-rata", f"{fmt(last.v_avg)} V")
k2.metric("Arus rata-rata", f"{fmt(last.i_avg)} A")
k3.metric("Power Factor", fmt(last.pf, "{:.3f}"))
k4.metric("Daya estimasi", f"{fmt(last.p_kw)} kW")

tab1, tab2 = st.tabs(["📈 Realtime", "📅 Rekap Harian"])

# ================= TAB REALTIME =================
with tab1:
    rt = df.tail(window)

    c1, c2 = st.columns(2)
    with c1:
        fig = px.line(rt, x="waktu", y=["vll1", "vll2", "vll3"],
                      title="Tegangan Line-to-Line (V)",
                      labels={"value": "Tegangan (V)", "waktu": "Waktu", "variable": "Fasa"})
        st.plotly_chart(fig, use_container_width=True)
    with c2:
        fig = px.line(rt, x="waktu", y=["i1", "i2", "i3"],
                      title="Arus per Fasa (A)",
                      labels={"value": "Arus (A)", "waktu": "Waktu", "variable": "Fasa"})
        st.plotly_chart(fig, use_container_width=True)

    c3, c4 = st.columns(2)
    with c3:
        fig = px.line(rt, x="waktu", y="pf", title="Power Factor",
                      labels={"pf": "PF", "waktu": "Waktu"})
        st.plotly_chart(fig, use_container_width=True)
    with c4:
        fig = px.line(rt, x="waktu", y="p_kw", title="Daya Estimasi (kW)",
                      labels={"p_kw": "Daya (kW)", "waktu": "Waktu"})
        st.plotly_chart(fig, use_container_width=True)

    st.subheader("Data Terakhir")
    table = rt.tail(20).iloc[::-1][["waktu", "vll1", "vll2", "vll3", "i1", "i2", "i3", "pf", "p_kw", "kwh"]]
    st.dataframe(table.rename(columns={
        "waktu": "Waktu", "vll1": "VLL-1 (V)", "vll2": "VLL-2 (V)", "vll3": "VLL-3 (V)",
        "i1": "I-1 (A)", "i2": "I-2 (A)", "i3": "I-3 (A)",
        "pf": "PF", "p_kw": "Daya (kW)", "kwh": "kWh Akumulasi"}),
        use_container_width=True, hide_index=True)

# ================= TAB REKAP HARIAN =================
with tab2:
    daily = daily_recap(df)

    m1, m2, m3 = st.columns(3)
    m1.metric("Total energi (periode data)", f"{fmt(daily.kwh_pakai.sum())} kWh")
    m2.metric("Rata-rata tegangan harian", f"{fmt(daily.v_avg.mean())} V")
    m3.metric("Hari tercatat", len(daily))

    c5, c6 = st.columns(2)
    with c5:
        fig = px.bar(daily, x="tanggal", y="kwh_pakai", title="Energi Terpakai per Hari (kWh)",
                     labels={"kwh_pakai": "kWh", "tanggal": "Tanggal"})
        st.plotly_chart(fig, use_container_width=True)
    with c6:
        fig = px.line(daily, x="tanggal", y=["v_avg", "i_avg"],
                      title="Rata-rata Harian Tegangan (V) dan Arus (A)",
                      labels={"value": "Nilai", "tanggal": "Tanggal", "variable": "Parameter"})
        st.plotly_chart(fig, use_container_width=True)

    st.subheader("Tabel Rekap Harian")
    show = daily.round(2).rename(columns={
        "tanggal": "Tanggal", "v_avg": "V rata-rata", "v_min": "V min", "v_max": "V maks",
        "i_avg": "I rata-rata", "i_max": "I maks", "pf_avg": "PF rata-rata",
        "p_avg": "Daya rata-rata (kW)", "kwh_awal": "kWh awal", "kwh_akhir": "kWh akhir",
        "kwh_pakai": "kWh terpakai", "jumlah": "Jumlah data"})
    st.dataframe(show, use_container_width=True, hide_index=True)

    st.download_button("⬇️ Unduh rekap harian (CSV)",
                       data=show.to_csv(index=False).encode("utf-8"),
                       file_name="rekap_harian.csv", mime="text/csv")pip install streamlit pandas plotly streamlit-autorefresh

