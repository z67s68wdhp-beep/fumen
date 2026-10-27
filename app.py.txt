import streamlit as st
import pandas as pd
import sqlite3
import hashlib
import os

# ページ設定
st.set_page_config(page_title="大会集計システム", layout="centered")

SCORE_RANGES = {'4': 90, '3.5': 85, '3': 80, '2.5': 75, '2': 70, '1.5': 65, '1': 60}
MIN_COMMENT_LENGTH_FOR_LOW_SCORE = 200
DB_NAME = "taikai_master.db"

def hash_real_name(real_name):
    if not real_name:
        return ""
    return hashlib.sha256(real_name.strip().lower().encode('utf-8')).hexdigest()

def get_connection():
    conn = sqlite3.connect(DB_NAME, timeout=5)
    return conn

def init_db():
    conn = get_connection()
    cursor = conn.cursor()
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS Participants (
            id INTEGER PRIMARY KEY,
            song_name TEXT NOT NULL,
            artist_name TEXT,
            real_name_hash TEXT,
            competition_name TEXT,
            intention TEXT,
            request_comment INTEGER DEFAULT 0,
            UNIQUE (song_name, artist_name, competition_name) 
        )
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS Evaluations (
            evaluator_name TEXT,
            fumen_id INTEGER,
            score INTEGER NOT NULL,
            comment TEXT,
            favorite INTEGER DEFAULT 0,
            PRIMARY KEY (evaluator_name, fumen_id)
        )
    """)
    cursor.execute("""
        CREATE TABLE IF NOT EXISTS Settings (
            key TEXT PRIMARY KEY,
            value TEXT
        )
    """)
    conn.commit()
    conn.close()

init_db()

# サイドバーでモード切り替え
st.sidebar.title("🛠️ モード選択")
mode = st.sidebar.radio("モードを選んでください", ["📝 評価者モード（スマホ・PC対応）", "🚀 主催者モード（管理・集計）"])

# =================================================================
# 📝 評価者モード
# =================================================================
if mode == "📝 評価者モード（スマホ・PC対応）":
    st.title("📝 大会 評価フォーム")
    
    conn = get_connection()
    
    # 評価者情報入力
    st.subheader("1. あなたの情報を入力")
    col1, col2 = st.columns(2)
    with col1:
        evaluator_name = st.text_input("評価者名（ハンドルネーム等）", value="")
    with col2:
        real_name = st.text_input("あなたの譜面の「本名義」（自己評価除外用）", value="")
        
    if not evaluator_name:
        st.warning("評価者名を入力すると評価フォームが表示されます。")
        st.stop()
        
    real_name_hash = hash_real_name(real_name)
    
    # 譜面リストの取得
    participants_df = pd.read_sql_query("SELECT * FROM ID, song_name, artist_name, competition_name, intention, request_comment FROM Participants", conn) if False else pd.read_sql_query("SELECT id, song_name, artist_name, competition_name, intention, request_comment FROM Participants", conn)
    
    if participants_df.empty:
        st.info("現在登録されている譜面がありません。主催者による譜面登録をお待ちください。")
        st.stop()
        
    # 自己評価の除外チェック
    valid_fumens = []
    for _, row in participants_df.iterrows():
        comp_hash = hash_real_name(row['competition_name'])
        is_banned = (comp_hash == real_name_hash and real_name_hash != "")
        if not is_banned:
            valid_fumens.append(row)
            
    if not valid_fumens:
        st.error("評価できる譜面がありません（自分の譜面を除外した結果0件になりました）。")
        st.stop()
        
    # セレクトボックス用選択肢
    fumen_options = {f"【{f['competition_name']}】 {f['song_name']} ({f['artist_name']})": f['id'] for f in valid_fumens}
    selected_display = st.selectbox("評価する譜面を選択", list(fumen_options.keys()))
    selected_fumen_id = fumen_options[selected_display]
    
    # 選択された譜面の詳細
    target_fumen = next(f for f in valid_fumens if f['id'] == selected_fumen_id)
    req_comment = target_fumen['request_comment']
    intention = target_fumen['intention']
    
    if req_comment == 1:
        st.markdown("🔴 **【この譜面は感想希望（コメント必須）です】**")
    else:
        st.markdown("🔵 **【感想任意】**")
        
    if intention:
        st.info(f"💡 **意気込み:** {intention}")
        
    # 既存の評価データを取得
    existing_eval = pd.read_sql_query(
        "SELECT score, comment, favorite FROM Evaluations WHERE evaluator_name = ? AND fumen_id = ?",
        conn, params=(evaluator_name, selected_fumen_id)
    )
    
    default_score = int(existing_eval.iloc[0]['score']) if not existing_eval.empty else 70
    default_comment = existing_eval.iloc[0]['comment'] if not existing_eval.empty else ""
    default_fav = bool(existing_eval.iloc[0]['favorite']) if not existing_eval.empty else False

    st.subheader("2. 採点と感想")
    score = st.slider("点数 (0〜100)", 0, 100, default_score)
    favorite = st.checkbox("⭐ お気に入りに追加する", value=default_fav)
    comment = st.text_area("総合コメント・感想", value=default_comment, height=150)
    
    # 保存ボタン
    if st.button("評価を保存する", type="primary"):
        # バリデーション
        if score < 50 and len(comment) < MIN_COMMENT_LENGTH_FOR_LOW_SCORE:
            st.error(f"❌ 50点未満にする場合は、最低 {MIN_COMMENT_LENGTH_FOR_LOW_SCORE} 文字以上の感想が必要です。（現在: {len(comment)}文字）")
        elif req_comment == 1 and not comment.strip():
            st.error("❌ この譜面は感想希望が出ているため、コメントの記入が必須です。")
        else:
            cursor = conn.cursor()
            cursor.execute(
                "INSERT OR REPLACE INTO Evaluations (evaluator_name, fumen_id, score, comment, favorite) VALUES (?, ?, ?, ?, ?)",
                (evaluator_name, selected_fumen_id, score, comment, 1 if favorite else 0)
            )
            conn.commit()
            st.success("✨ 評価を保存しました！")
            
    conn.close()

# =================================================================
# 🚀 主催者モード
# =================================================================
elif mode == "🚀 主催者モード（管理・集計）":
    st.title("🚀 主催者ダッシュボード")
    
    conn = get_connection()
    
    tab1, tab2, tab3 = st.tabs(["📁 譜面一括登録 (CSV)", "📊 総合集計・順位", "⚙️ 設定"])
    
    with tab1:
        st.subheader("スプレッドシート等からのCSV一括インポート")
        uploaded_file = st.file_uploader("CSVファイルを選択", type=["csv", "txt"])
        if uploaded_file is not None:
            df = pd.read_csv(uploaded_file, encoding='utf-8-sig', dtype=str).fillna("")
            st.write("プレビュー:", df.head())
            if st.button("データベースに登録する"):
                cursor = conn.cursor()
                success = 0
                for _, row in df.iterrows():
                    try:
                        song = row.get('曲名', '').strip()
                        artist = row.get('作曲者名', '').strip()
                        comp = row.get('大会名義', '').strip()
                        real = row.get('本名義', '').strip()
                        intent = row.get('意気込み', '').strip()
                        req = 1 if str(row.get('感想希望', '')).strip().upper() in ['TRUE', '1', '希望'] else 0
                        
                        if song:
                            cursor.execute(
                                "INSERT OR IGNORE INTO Participants (song_name, artist_name, real_name_hash, competition_name, intention, request_comment) VALUES (?, ?, ?, ?, ?, ?)",
                                (song, artist, hash_real_name(real), comp, intent, req)
                            )
                            success += 1
                    except Exception as e:
                        pass
                conn.commit()
                st.success(f"🎉 {success} 件の処理が完了しました！")
                
    with tab2:
        st.subheader("📊 リアルタイム集計結果（最低点除外）")
        eval_df = pd.read_sql_query("""
            SELECT E.evaluator_name, E.fumen_id, P.song_name || ' (' || P.artist_name || ')' AS fumen_name, 
                   E.score, E.favorite, P.competition_name
            FROM Evaluations E JOIN Participants P ON E.fumen_id = P.id
        """, conn)
        
        if not eval_df.empty:
            grouped = eval_df.groupby(['fumen_id', 'fumen_name', 'competition_name'])['score']
            summary = grouped.agg(評価者数='count', 総合合計点='sum', 中央値='median', 最低点='min').reset_index()
            
            def calc_avg_no_min(scores):
                l = scores.tolist()
                if len(l) <= 1: return l[0] if l else 0
                l.remove(min(l))
                return sum(l) / len(l)
                
            summary['平均点_最低点除外'] = grouped.apply(calc_avg_no_min).values
            
            fav = eval_df.groupby('fumen_id')['favorite'].sum().reset_index().rename(columns={'favorite': 'お気に入り総数'})
            summary = pd.merge(summary, fav, on='fumen_id', how='left').fillna(0)
            
            # 順位付け
            summary_sorted = summary.sort_values(by=['平均点_最低点除外', '総合合計点', 'お気に入り総数', '中央値'], ascending=[False, False, False, False])
            summary_sorted['総合順位'] = summary_sorted.groupby(['平均点_最低点除外', '総合合計点', 'お気に入り総数', '中央値'], sort=False).ngroup() + 1
            
            st.dataframe(summary_sorted[['総合順位', 'fumen_name', 'competition_name', '評価者数', '平均点_最低点除外', '総合合計点', 'お気に入り総数']])
        else:
            st.info("まだ評価データがありません。")
            
    with tab3:
        st.subheader("大会設定")
        limit = st.text_input("点数被り許容数", value="5")
        if st.button("設定を保存"):
            cursor = conn.cursor()
            cursor.execute("INSERT OR REPLACE INTO Settings (key, value) VALUES ('duplicate_score_limit', ?)", (limit,))
            conn.commit()
            st.success("保存しました！")

    conn.close()
