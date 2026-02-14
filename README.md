import streamlit as st
import pandas as pd
import plotly.express as px
import requests
from io import BytesIO

# --- SAYFA YAPILANDIRMASI ---
st.set_page_config(page_title="DropAnalyzer Elite | Arda", layout="wide", page_icon="📈")

# --- CUSTOM CSS (Profesyonel Görünüm İçin) ---
st.markdown("""
    <style>
    .main { background: linear-gradient(135deg, #0e1117 0%, #161b22 100%); color: white; }
    .stMetric { background: rgba(255, 255, 255, 0.05); border-radius: 15px; padding: 20px; border: 1px solid rgba(255,255,255,0.1); transition: 0.3s; }
    .stMetric:hover { border: 1px solid #00d4ff; background: rgba(255, 255, 255, 0.08); }
    div[data-testid="stExpander"] { background: rgba(255, 255, 255, 0.03); border: 1px solid rgba(255,255,255,0.1); border-radius: 10px; }
    .stButton>button { width: 100%; border-radius: 8px; background: linear-gradient(45deg, #00d4ff, #0055ff); color: white; border: none; font-weight: bold; }
    </style>
    """, unsafe_allow_html=True)

# --- GÜVENLİK SİSTEMİ ---
def check_password():
    if "password_correct" not in st.session_state:
        col1, col2, col3 = st.columns([1,2,1])
        with col2:
            st.image("https://cdn-icons-png.flaticon.com/512/1162/1162456.png", width=100)
            st.title("Elite Access")
            u = st.text_input("Yönetici Adı")
            p = st.text_input("Erişim Anahtarı", type="password")
            if st.button("Sistemi Başlat"):
                if u == "Arda" and p == "Arda2026": # Şifreni buradan değiştirebilirsin
                    st.session_state["password_correct"] = True
                    st.rerun()
                else:
                    st.error("❌ Yetkisiz Giriş Engellendi.")
        return False
    return True

# --- ANA UYGULAMA MANTIĞI ---
if check_password():
    
    # Veritabanı ve Kur Çekimi
    if 'history' not in st.session_state: st.session_state['history'] = []
    
    @st.cache_data(ttl=3600)
    def get_rate():
        try: return requests.get("https://api.exchangerate-api.com/v4/latest/USD").json()['rates']['TRY']
        except: return 34.50
    
    usd_try = get_rate()

    # Yan Panel
    with st.sidebar:
        st.markdown("### 🛠️ Kontrol Merkezi")
        p_name = st.text_input("Ürün İsmi", "Premium Smart Watch")
        p_img = st.text_input("Görsel URL", "https://images.unsplash.com/photo-1523275335684-37898b6baf30?w=500")
        platform = st.selectbox("Satış Kanalı", ["Amazon", "Shopify", "Trendyol", "Etsy"])
        
        st.divider()
        b_price = st.number_input("Alış ($)", value=25.0)
        s_price = st.number_input("Satış ($)", value=79.0)
        ship = st.number_input("Lojistik ($)", value=10.0)
        ad = st.number_input("Reklam/CPA ($)", value=15.0)
        
        save = st.button("📊 VERİLERİ ARŞİVLE")
        if st.button("🔒 Güvenli Çıkış"):
            del st.session_state["password_correct"]
            st.rerun()

    # Hesaplamalar
    fee = s_price * {"Amazon": 0.15, "Shopify": 0.02, "Trendyol": 0.18, "Etsy": 0.065}.get(platform, 0.05)
    total_cost = b_price + ship + ad + fee
    profit_usd = s_price - total_cost
    profit_try = profit_usd * usd_try
    roi = (profit_usd / total_cost) * 100 if total_cost > 0 else 0

    # UI Dashboard
    st.title("💎 DropAnalyzer Elite v4")
    st.caption(f"Hoş geldin Arda. Canlı Kur: 1$ = {usd_try}₺ | Durum: Senkronize")

    m1, m2, m3, m4 = st.columns(4)
    m1.metric("NET KÂR (USD)", f"${profit_usd:.2f}")
    m2.metric("NET KÂR (TRY)", f"₺{profit_try:,.2f}")
    m3.metric("ROI", f"%{roi:.1f}")
    m4.metric("MARJ", f"%{(profit_usd/s_price)*100:.1f}")

    st.divider()

    c_left, c_right = st.columns([1, 1.2])
    
    with c_left:
        st.image(p_img, use_container_width=True)
        with st.expander("📝 AI Pazarlama Metni"):
            st.write(f"🚀 **{p_name}** şimdi stoklarda! Kaçırılmayacak fırsat ve ücretsiz kargo avantajıyla hemen keşfet.")

    with c_right:
        st.subheader("📊 Gider Dağılım Grafiği")
        df = pd.DataFrame({"Kalem": ["Ürün", "Lojistik", "Reklam", "Komisyon"], "Tutar": [b_price, ship, ad, fee]})
        fig = px.pie(df, values='Tutar', names='Kalem', hole=0.6, color_discrete_sequence=px.colors.sequential.Tealgrn)
        fig.update_layout(margin=dict(t=0, b=0, l=0, r=0), showlegend=True)
        st.plotly_chart(fig, use_container_width=True)

    # Arşiv ve Excel
    if save:
        st.session_state['history'].append({"Ürün": p_name, "Kâr ($)": profit_usd, "ROI": f"%{roi:.1f}", "Tarih": "14.02.2026"})
    
    if st.session_state['history']:
        with st.expander("📂 Kayıtlı Raporlar & Excel Çıktısı"):
            df_h = pd.DataFrame(st.session_state['history'])
            st.table(df_h)
            
            output = BytesIO()
            with pd.ExcelWriter(output, engine='xlsxwriter') as writer:
                df_h.to_excel(writer, index=False)
            st.download_button("📥 Excel Raporu İndir", data=output.getvalue(), file_name="Arda_Export.xlsx")
