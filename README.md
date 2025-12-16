# Python-Test

import threading
import time
import gradio as gr
import pandas as pd
import datetime

# --- 核心數據結構與鎖定機制 ---
ACTIVE_TIMERS = []
TIMERS_LOCK = threading.Lock()
COMPLETED_HISTORY = []

class HiiT_Timer(threading.Thread):
    def __init__(self, name: str, work_s: int, rest_s: int, rounds: int):
        super().__init__()
        self.name = name
        self.work_duration = work_s
        self.rest_duration = rest_s
        self.total_rounds = rounds
        self.current_round = 1
        self.is_work_phase = True
        self.remaining_phase_time = work_s
        self.is_running = True
        self.alerted = False

    def run(self):
        while self.current_round <= self.total_rounds and self.is_running:
            if self.remaining_phase_time > 0:
                time.sleep(1)
                with TIMERS_LOCK:
                    if self.remaining_phase_time > 0:
                        self.remaining_phase_time -= 1
                continue

            if self.is_work_phase:
                print(f"\n📢 {self.name} - 第 {self.current_round} 回合工作完成！")
                if self.current_round < self.total_rounds:
                    print(f"😴 {self.name} 開始休息...")
                    self.is_work_phase = False
                    self.remaining_phase_time = self.rest_duration
                else:
                    break
            else:
                print(f"\n📣 {self.name} - 第 {self.current_round} 回合休息完成！")
                self.current_round += 1
                if self.current_round <= self.total_rounds:
                    print(f"💪 {self.name} 進入第 {self.current_round} 回合工作！")
                    self.is_work_phase = True
                    self.remaining_phase_time = self.work_duration
                else:
                    break

        if self.is_running:
            self.alerted = True
            print(f"\n🎉🎉 HIIT訓練完成！項目：{self.name} 已結束！🎉🎉")

            with TIMERS_LOCK:
                COMPLETED_HISTORY.append({
                    "名稱": self.name,
                    "設定": f"{self.get_remaining_str(self.work_duration)}/{self.get_remaining_str(self.rest_duration)} x {self.total_rounds}",
                    "完成時間": datetime.datetime.now().strftime("%H:%M:%S")
                })

    def stop_timer(self):
        self.is_running = False

    def get_remaining_str(self, seconds: int):
        total_seconds = seconds
        hours = total_seconds // 3600
        minutes = (total_seconds % 3600) // 60
        secs = total_seconds % 60
        if hours > 0:
            return f"{hours}:{minutes:02d}:{secs:02d}"
        else:
            return f"{minutes:02d}:{secs:02d}"

# --- Gradio 介面邏輯 (新增歷史紀錄顯示) ---

def add_hiit_timer(timer_name: str, work_minutes: float, rest_minutes: float, rounds: int) -> str:
    """處理使用者新增計時器的操作"""
    if not timer_name or work_minutes <= 0 or rest_minutes < 0 or rounds <= 0:
        return "❌ 錯誤：請輸入有效的名稱、時間和回合數！"

    work_s = int(work_minutes * 60)
    rest_s = int(rest_minutes * 60)

    new_timer = HiiT_Timer(timer_name, work_s, rest_s, rounds)

    with TIMERS_LOCK:
        ACTIVE_TIMERS.append(new_timer)

    new_timer.start()

    return f"✅ 成功啟動：'{timer_name}'，工作 {work_minutes} 分/休息 {rest_minutes} 分，共 {rounds} 回合。"

def display_timers() -> tuple[pd.DataFrame, pd.DataFrame]:
    """處理介面數據刷新和完成後的移除"""
    data = []
    timers_to_remove = []

    with TIMERS_LOCK:
        for timer in ACTIVE_TIMERS:
            status_text = "運行中"
            phase_time_str = ""

            if timer.alerted:
                status_text = "已完成！🎉"
                phase_time_str = "00:00"
                timers_to_remove.append(timer)
            elif timer.is_running:
                phase = "💪 工作中" if timer.is_work_phase else "😴 休息中"
                phase_time_str = timer.get_remaining_str(timer.remaining_phase_time)
                status_text = f"第 {timer.current_round}/{timer.total_rounds} 回合 ({phase})"

            data.append([
                timer.name,
                f"{timer.get_remaining_str(timer.work_duration)}/{timer.get_remaining_str(timer.rest_duration)}",
                status_text,
                phase_time_str
            ])

        for timer in timers_to_remove:
            ACTIVE_TIMERS.remove(timer)
            timer.stop_timer()

    if not data:
        active_df = pd.DataFrame(columns=["訓練項目", "W/R 時間", "當前進度/狀態", "階段剩餘時間"])
    else:
        active_df = pd.DataFrame(data, columns=["訓練項目", "W/R 時間", "當前進度/狀態", "階段剩餘時間"])

    history_df = pd.DataFrame(COMPLETED_HISTORY)
    if history_df.empty:
        history_df = pd.DataFrame(columns=["名稱", "設定", "完成時間"])

    return active_df, history_df

# --- Gradio 介面設定 ---

with gr.Blocks(title="HIIT 間歇訓練計時器") as demo:
    gr.Markdown("## 🏋️ HIIT 間歇訓練計時器：新手專案成果")
    gr.Markdown("此程式可並行運行多組訓練。狀態將每秒自動更新。")
    gr.Markdown("---")

    gr.Markdown("### 1. 訓練參數設定")

    with gr.Row():
        training_name_input = gr.Textbox(label="訓練項目名稱", placeholder="例如：TABATA - 深蹲", scale=2)
        work_time_input = gr.Number(label="工作時間 (分)", minimum=0.01, value=0.5, scale=1)
        rest_time_input = gr.Number(label="休息時間 (分)", minimum=0, value=0.25, scale=1)
        total_rounds_input = gr.Number(label="總回合數", minimum=1, value=8, precision=0, scale=1)
        start_btn = gr.Button("🚀 開始訓練", scale=1, variant="primary")

    output_message = gr.Textbox(label="操作訊息", lines=1, show_copy_button=False)

    gr.Markdown("---")
    gr.Markdown("### 2. 即時狀態列表 (S4) - 運行中項目")

    # 🎯 新增：手動刷新按鈕 (用於診斷)
    manual_refresh_btn = gr.Button("🔄 手動刷新狀態 (點擊診斷問題)", variant="secondary")

    timer_table = gr.Dataframe(
        headers=["訓練項目", "W/R 時間", "當前進度/狀態", "階段剩餘時間"],
        datatype=["str", "str", "str", "str"],
        row_count=5,
        col_count=(4, "fixed"),
        interactive=False,
        label="間歇訓練狀態列表 (運行中項目會在完成後自動移除)"
    )

    gr.Markdown("---")
    gr.Markdown("### 3. 已完成訓練歷史紀錄")

    history_table = gr.Dataframe(
        headers=["名稱", "設定", "完成時間"],
        datatype=["str", "str", "str"],
        row_count=5,
        col_count=(3, "fixed"),
        interactive=False,
        label="本次執行期間已完成的訓練"
    )

    # --- 事件處理 ---

    start_btn.click(
        fn=add_hiit_timer,
        inputs=[training_name_input, work_time_input, rest_time_input, total_rounds_input],
        outputs=output_message
    )

    # 🎯 診斷：手動刷新按鈕的點擊事件
    manual_refresh_btn.click(
        fn=display_timers,
        outputs=[timer_table, history_table],
        show_progress="minimal"
    )

    # JavaScript 驅動定時刷新機制 (最穩健機制)
    refresh_btn = gr.Button("Refresh Data", visible=False)

    refresh_btn.click(
        fn=display_timers,
        outputs=[timer_table, history_table]
    )

    js_refresh_loop = f"""
    function start_refresh_loop() {{
        const refreshButton = document.querySelector('button[data-testid="{refresh_btn._id}"]');
        if (refreshButton) {{
            setInterval(function() {{
                refreshButton.click();
            }}, 1000);
        }}
    }}
    start_refresh_loop();
    """

    demo.load(
        fn=None,
        inputs=None,
        outputs=None,
        js=js_refresh_loop
    )

# 啟動 Gradio 介面
if __name__ == "__main__":
    try:
        print("正在啟動 HIIT 計時器伺服器...")
        demo.launch(share=True)
    except Exception as e:
        print(f"啟動 Gradio 發生錯誤：{e}")
