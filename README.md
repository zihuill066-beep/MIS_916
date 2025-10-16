# MIS_916
MIS lesson w6d2
from pathlib import Path
import pandas as pd
import streamlit as st

st.set_page_config(page_title="MoodPattern", page_icon="🧠", layout="wide")
st.title("🧠MoodPattern")

BASE_DIR = Path().parent
DATA_DIR = BASE_DIR / "data"
CSV_PATH = DATA_DIR / "mood_records.csv"
DATA_DIR.mkdir(parents=True, exist_ok=True)

COLUMNS = ["id", "date", "mood_score", "activities", "notes", "created_at","name"]
if CSV_PATH.exists():
    df = pd.read_csv(CSV_PATH)
else:
    df = pd.DataFrame(columns=COLUMNS)
def save_data(df: pd.DataFrame):
    df.to_csv(CSV_PATH, index=False)

st.header("情绪记录表单")
with st.form("mood_form", clear_on_submit=True):
    name = st.text_input("记录人姓名")
    mood_score = st.slider("今日情绪值", min_value=1, max_value=10, value=5)
    activity = st.multiselect("您的今日活动", ['学习', '工作', '社交', '宅家', '运动', '娱乐', '睡觉'])
    notes = st.text_area("请简要填写今日感受", "100字以内", height=100)
    submitted = st.form_submit_button("提交记录")

if submitted:
    new_record = {
        "id": 0 if df.empty else int(df["id"].max()) + 1,
        "date": pd.Timestamp.now().strftime("%Y-%m-%d"),
        "mood_score": mood_score,
        "activities": ", ".join(activity),
        "notes": notes,
        "created_at": pd.Timestamp.now().strftime("%Y-%m-%d %H:%M:%S"),
        "name": name
    }
    df_new = pd.DataFrame([new_record])
    df = pd.concat([df, df_new], ignore_index=True)
    save_data(df)
    st.success("情绪记录已保存")

if not df.empty:
    last_date = pd.to_datetime(df['date']).max()
    if (pd.Timestamp.now() - last_date).days >= 3:
        st.warning("您已经三天没有记录了，记得更新心情哦！")

st.header("情绪记录列表")
if df.empty:
    st.write("暂无数据，请先添加情绪记录")
else:
    show_time = st.checkbox("显示时间戳")
    if show_time:
        st.write(df)
    else:
        st.write(df[["id", "date", "mood_score", "activities", "notes"]])

st.subheader("AI情绪总结")
if len(df) >= 3:
    avg_last3 = df.tail(3)['mood_score'].mean()
    if avg_last3 >= 8:
        summary = "最近三天情绪非常积极，保持好状态！"
    elif avg_last3 >= 5:
        summary = "最近三天情绪比较稳定，继续加油！"
    else:
        summary = "最近三天情绪有些低落，建议适当放松哦。"
    st.info(summary)

    st.subheader("情绪趋势图")
    df['date'] = pd.to_datetime(df['date'])
    daily_mood = df.groupby('date')['mood_score'].mean()
    st.line_chart(daily_mood)

    st.subheader("删除记录")
    delete_id = st.number_input("输入要删除的ID", min_value=0, step=1)
    if st.button("删除该记录"):
        if delete_id in df['id'].values:
            df = df[df['id'] != delete_id]
            save_data(df)
            st.success(f"ID={delete_id} 的记录已删除")
        else:
            st.warning(f"ID={delete_id} 不存在")

    st.subheader("关键词搜索")
    keyword = st.text_input("输入关键词搜索备注")
    if keyword:
        search_result = df[df['notes'].str.contains(keyword, na=False)]
        st.write(search_result if not search_result.empty else "没有找到相关记录")

    st.subheader("按记录人统计情绪均值")
    if not df.empty:
        person_stats = df.groupby('name')['mood_score'].mean()
        st.bar_chart(person_stats)

    st.subheader("按活动筛选记录")
    all_activities = df["activities"].dropna().unique()
    selected_activity = st.selectbox("选择要查看的活动", ["全部"] + list(all_activities))
    if selected_activity != "全部":
        st.write(df[df["activities"].str.contains(selected_activity)])
    else:
        st.write(df)

    display_option = st.radio("显示选项", ["全部记录", "最近3条", "随机1条"])
    if display_option == "最近3条":
        st.write(df.tail(3))
    elif display_option == "随机1条":
        st.write(df.sample(1))

    if st.button("复制数据到剪贴板"):
        df.to_clipboard(index=False)
        st.success("数据已复制到剪贴板，可以直接粘贴到Excel或Word")

    st.subheader("按指定格式输出")
    if st.button("生成填表格式视图"):
        formatted_df = df[["date", "mood_score", "activities", "notes"]]
        formatted_df.rename(columns={
            "date": "日期",
            "mood_score": "情绪值",
            "activities": "活动",
            "notes": "备注"
        }, inplace=True)
        st.write("以下为可直接复制到表格的格式：")
        st.write(formatted_df)
        formatted_df.to_clipboard(index=False)
        st.success("格式化数据已复制到剪贴板")
