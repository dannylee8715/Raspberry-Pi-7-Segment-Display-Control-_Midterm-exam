# Raspberry Pi-7 Segment Display Control_Midterm-exam
樹莓派七段顯示器控制(進階版本)
---
本期中報告展示如何使用 **`樹莓派`** 控制 七段顯示器。\
此程式支援多種顯示模式、按鈕操作（短按與長按），並可動態調整顯示速度，是114/11/01作業的進階功能。

---

# ⚙️ 功能特色
### 1. 按鈕控制運作
- 使用兩顆按鈕控制七段顯示器：「Start」與「Pause」。
### 2. 多種顯示模式
- CountUp（正向計數）：從 0 開始累加至 9。
- CountDown（倒數）：從 9 開始遞減至 0。
- Blink（閃爍）：整個顯示器閃爍。
- Random（隨機）：顯示隨機數字 0~9。
- HexCycle（字母循環）：循環顯示 A–F、- 及空白。
### 3. 短按與長按操作
-	短按：Start → 開始/繼續；Pause → 暫停。
-	長按：Start → 提升速度（間隔時間減半）；Pause → 切換顯示模式。

---

# 🛠️ 硬體需求
-	樹莓派4B
-	七段顯示器
-	2 顆按鈕
-	電阻（按鈕需要）
-	跳線與麵包板

---

# ▶️ 執行程式碼
<details>
<summary><h3>點我展開meHW.py</h3></summary>
  
```python
# ==============================================================
# 樹莓派七段顯示器控制（進階版本）  
# 功能：
#   1. 使用兩顆按鈕控制七段顯示器的運作
#   2. 具多種顯示模式（正數、倒數、閃爍、隨機、字母循環）
#   3. 短按與長按功能（短按控制啟動/暫停，長按切換模式或加速）
# ==============================================================
import RPi.GPIO as GPIO
import time
import random

# --------------------------------------------------------------
# GPIO 初始化設定
# --------------------------------------------------------------
GPIO.setmode(GPIO.BOARD)      # 採用實體腳位編號方式（BOARD 模式）
GPIO.setwarnings(False)       # 關閉重複警告訊息

# 七段顯示器連接腳位（依照實體腳位）
pins = [8, 10, 12, 16, 18, 22, 24]  # a,b,c,d,e,f,g 對應腳位

# --------------------------------------------------------------
# 七段顯示器各數字/字母對應的開關組合
# （1 = 點亮該段，0 = 關閉）
# --------------------------------------------------------------
Led7 = {
    '0': [1,1,1,1,1,1,0],
    '1': [0,1,1,0,0,0,0],
    '2': [1,1,0,1,1,0,1],
    '3': [1,1,1,1,0,0,1],
    '4': [0,1,1,0,0,1,1],
    '5': [1,0,1,1,0,1,1],
    '6': [1,0,1,1,1,1,1],
    '7': [1,1,1,0,0,0,0],
    '8': [1,1,1,1,1,1,1],
    '9': [1,1,1,1,0,1,1],
    # 額外字母樣式（用於 Hex 模式）
    'A': [1,1,1,0,1,1,1],
    'b': [0,0,1,1,1,1,1],
    'C': [1,0,0,1,1,1,0],
    'd': [0,1,1,1,1,0,1],
    'E': [1,0,0,1,1,1,1],
    'F': [1,0,0,0,1,1,1],
    '-': [0,0,0,0,0,0,1],
    ' ': [0,0,0,0,0,0,0]
}

# 將所有 7 段腳位設為輸出模式
for p in pins:
    GPIO.setup(p, GPIO.OUT)
    GPIO.output(p, GPIO.LOW)

# --------------------------------------------------------------
# 按鈕設定（使用樹莓派實體腳位 13、15）
# --------------------------------------------------------------
btn_start = 13  # Start按鈕腳位，控制啟動/加速
btn_pause = 15  # Pause按鈕腳位，控制暫停/切換模式

# 設定按鈕為輸入並啟用下拉電阻
GPIO.setup(btn_start, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)
GPIO.setup(btn_pause, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)

# --------------------------------------------------------------
# 狀態變數初始化
# --------------------------------------------------------------
is_running = False  # True 表示正在執行（顯示器持續變化）
count = 0           # 當前顯示的數字
speed = 1.0         # 顯示變換速度（單位：秒）
min_speed = 0.05    # 最快速度限制
max_speed = 2.0     # 最慢速度限制

# --------------------------------------------------------------
# 用來偵測短按/長按的變數
# --------------------------------------------------------------
last_start_state = False
last_pause_state = False
last_start_time = 0
last_pause_time = 0

LONG_PRESS_TIME = 1.2  # 長按判定時間（秒）

# --------------------------------------------------------------
# 可選顯示模式列表
# --------------------------------------------------------------
modes = ['CountUp', 'CountDown', 'Blink', 'Random', 'HexCycle']
mode_index = 0  # 當前模式索引（0 表示 CountUp）

# --------------------------------------------------------------
# 函式區 — 顯示控制
# --------------------------------------------------------------
def display_char(ch):
    """
    將指定字元 ch 顯示在七段顯示器上。
    若字元不在 Led7 字典中，顯示空白。
    """
    pat = Led7.get(str(ch), Led7[' '])
    for i in range(7):
        GPIO.output(pins[i], pat[i])

def show_number(n):
    """顯示數字（自動取餘數確保 0~9）"""
    display_char(str(n % 10))

def blink_once(on=True):
    """閃爍整個七段顯示器一次（on=True 亮，False 滅）"""
    display_char('8' if on else ' ')

def cycle_hex(idx):
    """循環顯示 A~F 等字母（用於 HexCycle 模式）"""
    hex_chars = ['A','b','C','d','E','F','-',' ']
    display_char(hex_chars[idx % len(hex_chars)])

# --------------------------------------------------------------
# 按鈕事件處理函式
# --------------------------------------------------------------
def handle_short_press_start():
    """短按 Start：開始或繼續執行"""
    global is_running
    is_running = True

def handle_long_press_start():
    """長按 Start：加速（速度減半）"""
    global speed
    speed = max(min_speed, speed / 2.0)
    print("速度提升，目前速度:", speed)

def handle_short_press_pause():
    """短按 Pause：暫停顯示"""
    global is_running
    is_running = False

def handle_long_press_pause():
    """長按 Pause：切換下一個顯示模式"""
    global mode_index
    mode_index = (mode_index + 1) % len(modes)
    print("模式切換 ->", modes[mode_index])

# --------------------------------------------------------------
# 讀取按鈕並判定短按/長按（含去彈跳）
# --------------------------------------------------------------
def read_buttons_and_handle():
    global last_start_state, last_pause_state
    global last_start_time, last_pause_time

    # 讀取按鈕目前電位狀態（HIGH = 按下）
    s = GPIO.input(btn_start)
    p = GPIO.input(btn_pause)
    now = time.time()

    # ---- START 按鈕 ----
    if s and not last_start_state:
        # 剛被按下（上升沿）
        last_start_time = now
    if not s and last_start_state:
        # 剛放開（下降沿）
        dur = now - last_start_time  # 按下持續時間
        if dur >= LONG_PRESS_TIME:
            handle_long_press_start()  # 長按
        else:
            handle_short_press_start() # 短按
    last_start_state = s  # 更新狀態

    # ---- PAUSE 按鈕 ----
    if p and not last_pause_state:
        last_pause_time = now
    if not p and last_pause_state:
        dur = now - last_pause_time
        if dur >= LONG_PRESS_TIME:
            handle_long_press_pause()
        else:
            handle_short_press_pause()
    last_pause_state = p

# --------------------------------------------------------------
# 主程式迴圈
# --------------------------------------------------------------
try:
    hex_idx = 0  # 用於 HexCycle 模式的字母循環
    while True:
        # 每回圈偵測按鈕輸入
        read_buttons_and_handle()

        # 取得當前顯示模式名稱
        mode = modes[mode_index]

        # 若目前為暫停狀態，顯示目前的數字並略作等待
        if not is_running:
            display_char(str(count % 10))
            time.sleep(0.1)
            continue

        # ------------------------------------------------------
        # 根據模式執行對應動作
        # ------------------------------------------------------
        if mode == 'CountUp':       # 正向計數模式
            show_number(count)
            count += 1
            time.sleep(speed)

        elif mode == 'CountDown':   # 反向計數模式
            show_number(count)
            count -= 1
            time.sleep(speed)

        elif mode == 'Blink':       # 閃爍模式
            blink_once(True)
            time.sleep(0.25)
            blink_once(False)
            time.sleep(max(0.05, speed - 0.25))

        elif mode == 'Random':      # 隨機數字模式
            r = random.randint(0,9)
            show_number(r)
            time.sleep(speed)

        elif mode == 'HexCycle':    # 顯示字母循環 A~F
            cycle_hex(hex_idx)
            hex_idx += 1
            time.sleep(max(0.2, speed/2.0))

        # 防止數字過大導致溢位，保持在合理範圍
        if count >= 1000 or count <= -1000:
            count = count % 10

# --------------------------------------------------------------
# 鍵盤中斷處理（Ctrl + C 停止）
# --------------------------------------------------------------
except KeyboardInterrupt:
    pass

# --------------------------------------------------------------
# 結束前清除 GPIO 設定，讓腳位恢復安全狀態
# --------------------------------------------------------------
finally:
    GPIO.cleanup()
```

</details>

---

# 🔗 GPIO 腳位設定
-	七段顯示器腳位：8, 10, 12, 16, 18, 22, 24 → 對應 a, b, c, d, e, f, g
-	Start 按鈕：GPIO 13
- Pause 按鈕：GPIO 15

所有顯示器腳位設定為**輸出模式**，按鈕設定為**輸入模式**，啟用下拉電阻。

---

# 🚀 核心函式
-	display_char(ch)：在七段顯示器上顯示指定字元。
-	show_number(n)：顯示數字 0~9。
-	blink_once(on=True)：閃爍顯示器一次。
-	cycle_hex(idx)：HexCycle 模式循環顯示 A~F、- 及空白。

---

# ⚫ 按鈕事件處理
- Start 按鈕
    -	短按 → 開始或繼續顯示
    -	長按 → 提升速度（間隔時間減半）
-	Pause 按鈕
    -	短按 → 暫停顯示
    -	長按 → 切換下一個顯示模式
---
# 📚 使用說明
1.	將七段顯示器及按鈕接到指定 GPIO 腳位。
2.	執行 Python 程式：
`python meHW.py`
3.	Start 按鈕：短按 → 開始/繼續；長按 → 提升速度。
4.	Pause 按鈕：短按 → 暫停；長按 → 切換顯示模式。
5.	使用 Ctrl + C 停止程式。
