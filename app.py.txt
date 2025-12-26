import streamlit as st
from streamlit_gsheets import GSheetsConnection
import pandas as pd
import plotly.express as px
import plotly.graph_objects as go
from datetime import datetime, date, timedelta

# --- 0. 設定と定数 ---
st.set_page_config(page_title="Sales Cockpit", layout="wide")
TARGET_PT = 2000
MEMBERS = ["矢野", "田川", "尾崎", "原", "寺崎"]

# --- 1. データ接続 (Google Sheets) ---
conn = st.connection("gsheets", type=GSheetsConnection)

# データを読み込む関数 (キャッシュを使って高速化)
def load_data():
    try:
        # シート名 'sales_data' を指定して読み込み
        df = conn.read(worksheet="sales_data", usecols=list(range(10)), ttl=5)
        df["日付"] = pd.to_datetime(df["日付"])
        return df.fillna("") # 空白を埋める
    except Exception as e:
        st.error(f"データの読み込みに失敗しました: {e}")
        return pd.DataFrame()

df = load_data()

# --- 2. 画面構成 (サイドバー) ---
st.sidebar.title("Sales Cockpit 🚀")
page = st.sidebar.radio("メニュー", ["📊 ダッシュボード", "📝 実績入力", "🗂 案件ボード"])

# --- 3. ページ別ロジック ---

# ==========================================
# PAGE 1: ダッシュボード (戦況図)
# ==========================================
if page == "📊 ダッシュボード":
    st.title("🏆 チーム戦況ダッシュボード")

    # 全体集計
    current_pt = df["獲得PT"].replace("", 0).sum() if not df.empty else 0
    remaining_pt = TARGET_PT - current_pt
    progress = min(current_pt / TARGET_PT, 1.0)

    # ビッグカウンター
    col1, col2, col3 = st.columns(3)
    col1.metric("今月の獲得PT", f"{int(current_pt)} pt")
    col2.metric("目標 (2000pt) まで", f"あと {int(remaining_pt)} pt !!", delta_color="inverse")
    col3.progress(progress)

    st.divider()

    # ファネル分析 (KPI集計)
    if not df.empty:
        # 集計ロジック
        total_appt = df["アポ獲得数"].replace("", 0).sum() # デスクワークのアポ数
        total_visit = len(df[df["活動区分"] == "対面商談"]) # 営業実施
        total_next = len(df[df["商談結果"] == "次回面談設定"]) # 面談設定
        total_interview = len(df[df["活動区分"] == "面談"]) # 面談実施
        total_win = len(df[df["商談結果"] == "受注"]) # 受注

        # 面談設定率の計算
        denominator = total_visit + total_interview
        next_rate = (total_next / denominator * 100) if denominator > 0 else 0

        # アラート表示
        if next_rate < 20:
            st.error(f"⚠️ チーム全体の「面談設定率」が {next_rate:.1f}% です！ (目標20%未満)")
        
        # ファネルグラフ
        fig = go.Figure(go.Funnel(
            y=["対面商談設定", "商談/面談実施", "次回面談設定", "受注"],
            x=[total_appt, denominator, total_next, total_win],
            textinfo="value+percent previous"
        ))
        st.plotly_chart(fig, use_container_width=True)

        # ランキング表
        st.subheader("🔥 メンバー別ランキング")
        # メンバーごとに集計
        ranking_df = df.groupby("メンバー")[["獲得PT", "アポ獲得数"]].sum().reset_index()
        # 実施数をカウント
        visit_counts = df[df["活動区分"].isin(["対面商談", "面談"])].groupby("メンバー").size().reset_index(name="実施数")
        ranking_df = pd.merge(ranking_df, visit_counts, on="メンバー", how="left").fillna(0)
        
        st.dataframe(ranking_df.sort_values("獲得PT", ascending=False), hide_index=True)

# ==========================================
# PAGE 2: 実績入力 (アポ＆商談)
# ==========================================
elif page == "📝 実績入力":
    st.title("📝 今日の動きを入力")
    
    with st.form("input_form"):
        date_input = st.date_input("日付", date.today())
        member_input = st.selectbox("メンバー名", MEMBERS)
        
        st.subheader("① デスクワーク報告 (アポ取り)")
        appt_count = st.number_input("今日取ったアポ数 (件)", min_value=0, step=1)

        st.divider()

        st.subheader("② 商談・フィールド報告")
        st.info("※1件ずつ入力して送信してください")
        
        col_a, col_b = st.columns(2)
        type_input = col_a.radio("活動区分", ["事務・アポ取りのみ", "対面商談", "面談"], horizontal=True)
        source_input = col_b.radio("アポ供給元", ["自分", "アポインター", "インバウンド"], horizontal=True)
        
        client_input = st.text_input("顧客名 (必須)", placeholder="株式会社〇〇")
        
        result_input = st.selectbox("商談結果", ["-", "次回面談設定", "継続検討", "受注", "失注/NG", "キャンセル"])
        
        col_c, col_d = st.columns(2)
        next_date_input = col_c.date_input("次回予定日", value=None)
        pt_input = col_d.number_input("獲得PT", min_value=0, step=10)
        
        memo_input = st.text_area("メモ (所感・NG理由など)", height=80)

        submitted = st.form_submit_button("登録する")

        if submitted:
            # 入力チェック
            if type_input != "事務・アポ取りのみ" and not client_input:
                st.error("商談報告の場合、顧客名は必須です！")
            elif result_input == "次回面談設定" and not next_date_input:
                st.error("「次回面談設定」の場合は、次回予定日を入力してください！(田川・寺崎対策)")
            else:
                # データの整形
                new_data = pd.DataFrame([{
                    "日付": date_input.strftime("%Y-%m-%d"),
                    "メンバー": member_input,
                    "活動区分": type_input,
                    "顧客名": client_input,
                    "供給元": source_input,
                    "商談結果": result_input,
                    "次回日付": next_date_input.strftime("%Y-%m-%d") if next_date_input else "",
                    "獲得PT": pt_input,
                    "アポ獲得数": appt_count,
                    "メモ": memo_input
                }])
                
                # スプレッドシートに追記
                updated_df = pd.concat([df, new_data], ignore_index=True)
                conn.update(worksheet="sales_data", data=updated_df)
                st.success("登録しました！お疲れ様です。")
                st.cache_data.clear() # キャッシュクリアして即反映

# ==========================================
# PAGE 3: 案件ボード (カンバン)
# ==========================================
elif page == "🗂 案件ボード":
    st.title("🗂 進行案件管理ボード")
    
    # フィルタリング
    filter_member = st.selectbox("担当者で絞り込み", ["全員"] + MEMBERS)
    
    # 完了案件(受注/失注)を除外したデータを作成
    active_df = df[~df["商談結果"].isin(["受注", "失注/NG", "キャンセル", "-"])].copy()
    
    # 最新の状態だけを取得（同じ会社で複数ログがある場合、日付が新しいものを採用）
    if not active_df.empty:
        active_df = active_df.sort_values("日付", ascending=False).drop_duplicates(subset=["顧客名", "メンバー"], keep="first")

    if filter_member != "全員":
        active_df = active_df[active_df["メンバー"] == filter_member]

    # カンバン表示用の列定義
    col1, col2, col3 = st.columns(3)
    
    with col1:
        st.header("🟡 継続検討 (沼)")
        st.caption("※最終接触から14日以上で赤字")
        
        targets = active_df[active_df["商談結果"] == "継続検討"]
        for index, row in targets.iterrows():
            # 放置日数の計算
            last_date = pd.to_datetime(row["日付"]).date()
            days_diff = (date.today() - last_date).days
            
            # カードのデザイン
            card_color = "red" if days_diff >= 14 else ("orange" if days_diff >= 7 else "black")
            bg_color = "#ffe6e6" if days_diff >= 14 else "#ffffff"
            
            with st.container(border=True):
                st.markdown(f"**{row['顧客名']}**")
                st.caption(f"担当: {row['メンバー']}")
                st.markdown(f"<span style='color:{card_color}'>最終: {last_date} ({days_diff}日前)</span>", unsafe_allow_html=True)
                if days_diff >= 14:
                    st.error("放置危険！")

    with col2:
        st.header("🔥 次回設定済 (熱)")
        targets = active_df[active_df["商談結果"] == "次回面談設定"]
        for index, row in targets.iterrows():
            with st.container(border=True):
                st.markdown(f"**{row['顧客名']}**")
                st.caption(f"担当: {row['メンバー']}")
                st.write(f"📅 次回: **{row['次回日付']}**")

    with col3:
        st.header("🏁 今月の受注 (祝)")
        # 受注だけは履歴全部出す
        wins = df[df["商談結果"] == "受注"]
        if filter_member != "全員":
            wins = wins[wins["メンバー"] == filter_member]
            
        for index, row in wins.iterrows():
            with st.container(border=True):
                st.markdown(f"🎉 **{row['顧客名']}**")
                st.caption(f"{row['日付']} / {row['獲得PT']}pt")