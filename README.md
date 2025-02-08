import tkinter as tk
import win32gui
import win32con
import win32api
import time
import threading
import inputs


class CTRLSenderApp:
    def __init__(self, master):
        self.master = master
        master.title("Input Sender")
        master.geometry("300x450")

        self.title_label = tk.Label(master, text="Window Title:")
        self.title_label.pack(pady=(10, 0))
        self.title_entry = tk.Entry(master, width=40)
        self.title_entry.pack(pady=(0, 10))
        self.title_entry.insert(0, "Endless Online")

        self.toggle_on = False
        self.running = True

        self.auto_button = tk.Button(master, text="Start Auto CTRL", command=self.toggle_auto_ctrl)
        self.auto_button.pack(pady=5)

        move_frame = tk.Frame(master)
        move_frame.pack(pady=10)

        # Buttons for individual key presses
        self.w_button = tk.Button(move_frame, text="W", width=5, command=lambda: self.send_single_move('w'))
        self.w_button.grid(row=0, column=1)

        self.a_button = tk.Button(move_frame, text="A", width=5, command=lambda: self.send_single_move('a'))
        self.a_button.grid(row=1, column=0)

        self.s_button = tk.Button(move_frame, text="S", width=5, command=lambda: self.send_single_move('s'))
        self.s_button.grid(row=1, column=1)

        self.d_button = tk.Button(move_frame, text="D", width=5, command=lambda: self.send_single_move('d'))
        self.d_button.grid(row=1, column=2)

        self.f11_button = tk.Button(master, text="F11", command=lambda: self.send_single_move('f11'))
        self.f11_button.pack(pady=5)

        self.f12_button = tk.Button(master, text="F12", command=lambda: self.send_single_move('f12'))
        self.f12_button.pack(pady=5)

        self.f2_button = tk.Button(master, text="F2", command=lambda: self.send_single_move('f2'))
        self.f2_button.pack(pady=5)

        self.status_label = tk.Label(master, text="Ready", fg="green")
        self.status_label.pack(pady=10)

        # Threads for auto control and Xbox controller listener
        self.auto_thread = threading.Thread(target=self.auto_send_ctrl, daemon=True)
        self.auto_thread.start()

        self.controller_thread = threading.Thread(target=self.listen_to_xbox_controller, daemon=True)
        self.controller_thread.start()

    def find_window_by_title(self, title):
        def callback(hwnd, hwnds):
            if win32gui.IsWindowVisible(hwnd) and title in win32gui.GetWindowText(hwnd):
                hwnds.append(hwnd)

        hwnds = []
        win32gui.EnumWindows(callback, hwnds)
        return hwnds[0] if hwnds else None

    def send_single_move(self, direction):
        window_title = self.title_entry.get()
        hwnd = self.find_window_by_title(window_title)

        key_map = {
            'w': 0x57,  # W key
            's': 0x53,  # S key
            'a': 0x41,  # A key
            'd': 0x44,  # D key
            'f11': win32con.VK_F11,  # F11 key
            'f12': win32con.VK_F12,  # F12 key
            'f2': win32con.VK_F2     # F2 key
        }

        if hwnd:
            win32gui.ShowWindow(hwnd, win32con.SW_RESTORE)  # Bring window to focus
            win32gui.SetForegroundWindow(hwnd)
            key_code = key_map[direction]
            win32api.PostMessage(hwnd, win32con.WM_KEYDOWN, key_code, 0)
            time.sleep(0.05)
            win32api.PostMessage(hwnd, win32con.WM_KEYUP, key_code, 0)

            self.update_status(f"{direction.upper()} sent to {window_title}", "green")
        else:
            self.update_status(f"Window '{window_title}' not found!", "red")

    def send_global_key(self, key_code):
        """Send a key globally without focusing the window."""
        win32api.keybd_event(key_code, 0, win32con.KEYEVENTF_EXTENDEDKEY, 0)
        time.sleep(0.05)
        win32api.keybd_event(key_code, 0, win32con.KEYEVENTF_KEYUP, 0)

    def auto_send_ctrl(self):
        while self.running:
            if self.toggle_on:
                window_title = self.title_entry.get()
                hwnd = self.find_window_by_title(window_title)
                if hwnd:
                    # Directly send CTRL key without focusing the window
                    win32api.PostMessage(hwnd, win32con.WM_KEYDOWN, win32con.VK_CONTROL, 0)
                    time.sleep(0.05)
                    win32api.PostMessage(hwnd, win32con.WM_KEYUP, win32con.VK_CONTROL, 0)
                else:
                    self.update_status(f"Window '{window_title}' not found!", "red")
            time.sleep(1)

    def toggle_auto_ctrl(self):
        self.toggle_on = not self.toggle_on
        status = "enabled" if self.toggle_on else "disabled"
        button_text = "Stop Auto CTRL" if self.toggle_on else "Start Auto CTRL"

        self.auto_button.config(text=button_text)
        self.update_status(f"Auto CTRL {status}", "green" if self.toggle_on else "red")

    def listen_to_xbox_controller(self):
        while self.running:
            try:
                events = inputs.get_gamepad()
                for event in events:
                    # Toggle Auto CTRL with the left joystick button
                    if event.ev_type == "Key" and event.code == "BTN_THUMBL" and event.state == 1:
                        self.toggle_auto_ctrl()

                    # Press F12 with the right joystick button down
                    if event.ev_type == "Key" and event.code == "BTN_THUMBR" and event.state == 1:
                        window_title = self.title_entry.get()
                        hwnd = self.find_window_by_title(window_title)
                        if hwnd:
                            win32api.PostMessage(hwnd, win32con.WM_KEYDOWN, win32con.VK_F12, 0)
                            time.sleep(0.05)
                            win32api.PostMessage(hwnd, win32con.WM_KEYUP, win32con.VK_F12, 0)
                            self.update_status("F12 triggered by right joystick", "green")
                        else:
                            self.update_status(f"Window '{window_title}' not found!", "red")

            except Exception as e:
                self.update_status(f"Controller Error: {e}", "red")

    def update_status(self, message, color="green"):
        def update():
            self.status_label.config(text=message, fg=color)

        self.master.after(0, update)


def main():
    root = tk.Tk()
    app = CTRLSenderApp(root)
    root.mainloop()


if __name__ == "__main__":
    main()
