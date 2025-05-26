import time
import threading
import math
from pynput import keyboard
from pynput.mouse import Controller as MouseController
from screeninfo import get_monitors

# ========================
# 🔧 設定セクション
# ========================
REVERSE_KEY = keyboard.Key.f1
EXIT_KEY = keyboard.Key.f12
TOGGLE_ACCELERATION_KEY = keyboard.Key.f2  # 加速度ON/OFF切り替えキー
# ========================

# 初期化
mouse = MouseController()
reverse = False
poll_interval = 0.005
MIN_DELTA = 1.0
reverse_multiplier = 1
MAX_MULTIPLIER = 2
MIN_MULTIPLIER = 0.6
reverse_start_position = None
has_saved_start = False
acceleration_enabled = False  # 加速度がONかどうかのフラグ

# モニタサイズ取得
screen = get_monitors()[0]
screen_width, screen_height = screen.width, screen.height

def monitor_mouse():
    global reverse_start_position, has_saved_start, acceleration_enabled

    last_x, last_y = mouse.position

    while True:
        time.sleep(poll_interval)
        current_x, current_y = mouse.position
        dx = current_x - last_x
        dy = current_y - last_y
        distance = math.hypot(dx, dy)

        # 反転倍率を考慮した最小移動距離の調整
        adjusted_min_delta = MIN_DELTA / reverse_multiplier

        if reverse and distance >= adjusted_min_delta:
            # 加速度が有効な場合のみ加速度を適用
            if acceleration_enabled:
                acceleration_factor = 1 + (distance / 1000)
                smooth_multiplier = reverse_multiplier * acceleration_factor
            else:
                smooth_multiplier = reverse_multiplier

            new_x = int(current_x - 2 * dx * smooth_multiplier)
            new_y = int(current_y - 2 * dy * smooth_multiplier)

            # 画面外に出ないように制限を加える
            new_x = max(0, min(new_x, screen_width - 1))
            new_y = max(0, min(new_y, screen_height - 1))

            mouse.position = (new_x, new_y)
            last_x, last_y = new_x, new_y
        else:
            last_x, last_y = current_x, current_y

def on_press(key):
    global reverse, reverse_multiplier, has_saved_start, reverse_start_position, acceleration_enabled

    try:
        if key == REVERSE_KEY:
            reverse = True
            if not has_saved_start:
                reverse_start_position = mouse.position  # 反転開始時に位置を保存
                has_saved_start = True
                print(f"[反転ON] 開始位置: {reverse_start_position}")
            print(f"[反転ON] 倍率: {reverse_multiplier}倍")

        elif key == EXIT_KEY:
            print("終了キーが押されました。プログラムを終了します。")
            exit(0)

        elif key == TOGGLE_ACCELERATION_KEY:
            acceleration_enabled = not acceleration_enabled  # 加速度のON/OFF切替
            state = "ON" if acceleration_enabled else "OFF"
            print(f"[加速度切替] 加速度: {state}")

        elif hasattr(key, 'char'):
            if key.char == '+':
                new_multiplier = round(min(reverse_multiplier + 0.2, MAX_MULTIPLIER), 1)
                if new_multiplier != reverse_multiplier:
                    reverse_multiplier = new_multiplier
                    print(f"[倍率UP] 現在の倍率: {reverse_multiplier}倍")

            elif key.char == '-':
                new_multiplier = round(max(reverse_multiplier - 0.2, MIN_MULTIPLIER), 1)
                if new_multiplier != reverse_multiplier:
                    reverse_multiplier = new_multiplier
                    print(f"[倍率DOWN] 現在の倍率: {reverse_multiplier}倍")

    except Exception as e:
        print(f"エラーが発生しました: {e}")

def on_release(key):
    global reverse, reverse_start_position, has_saved_start

    try:
        if key == REVERSE_KEY:
            reverse = False
            print("[反転OFF] 反対側の点にカーソルを移動します。")

            if reverse_start_position:
                end_x, end_y = mouse.position
                start_x, start_y = reverse_start_position

                dx = end_x - start_x
                dy = end_y - start_y
                mirror_x = start_x - dx
                mirror_y = start_y - dy

                # 画面外に出ないように制限
                mirror_x = max(0, min(mirror_x, screen_width - 1))
                mirror_y = max(0, min(mirror_y, screen_height - 1))

                mouse.position = (mirror_x, mirror_y)
                print(f"[対称移動] 開始: {start_x}, {start_y} / 終了: {end_x}, {end_y} → 対称: {mirror_x}, {mirror_y}")

                reverse_start_position = None
                has_saved_start = False  # 次回の反転に備えてリセット

    except Exception as e:
        print(f"エラーが発生しました: {e}")

# マウス監視スレッド
mouse_thread = threading.Thread(target=monitor_mouse)
mouse_thread.daemon = True
mouse_thread.start()

# キーボード監視
with keyboard.Listener(on_press=on_press, on_release=on_release) as listener:
    print(f"{REVERSE_KEY} を押している間マウス反転。+/-で倍率変更、{EXIT_KEY}で終了。")
    print(f"{TOGGLE_ACCELERATION_KEY} で加速度のON/OFFを切り替え。")
    listener.join()
