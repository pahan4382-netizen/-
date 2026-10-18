from datetime import datetime
from pathlib import Path
import difflib
import json
import math
import os
import subprocess
import threading
import time
import tkinter as tk
from tkinter import messagebox
import traceback
import webbrowser

import pyttsx3
import speech_recognition as sr

# ---------------- Настройки ----------------

# Слова активации (распознавание может писать имя по-разному)
WAKE_WORDS = ("джарвис", "jarvis", "жарвис", "дарвис", "джервис", "джарвиз")

# Программы без ярлыка: название -> путь к .exe или команда
COMMANDS = {
    "блокнот": "notepad.exe",
    "калькулятор": "calc.exe",
    "телеграмм": "unigram.exe",
}

# Слова, которые не получается угадать автоматически
ALIASES = {
    "стим": "steam",
    "хром": "chrome",
    "фотошоп": "photoshop",
    "юниграм": "unigram",
    "телеграм": "unigram",
    "телеграмм": "unigram",
}

# Сайты: слово -> ссылка ({date} подставится как сегодняшняя дата)
SITES = {
    "ютуб": "https://youtube.com",
    "youtube": "https://youtube.com",
    "гитхаб": "https://github.com",
    "github": "https://github.com",
    "клод": "https://claude.ai/new",
    "вк видео": "https://vkvideo.ru",
    "макс": "https://web.max.ru",
    "литкод": "https://leetcode.com/problemset/",
    "дневник": "https://school.mos.ru/diary/schedules/day?date={date}",
}

TRIGGERS = ("открой", "запусти", "зайди на", "включи", "открой мне", "а ну ка")

SEARCH_DIRS = [
    Path.home() / "Desktop",
    Path.home() / "Documents",
    Path.home() / "Downloads",
]

SHORTCUT_DIRS = [
    Path(os.environ.get("PROGRAMDATA", "")) / "Microsoft" / "Windows" / "Start Menu" / "Programs",
    Path(os.environ.get("APPDATA", "")) / "Microsoft" / "Windows" / "Start Menu" / "Programs",
    Path.home() / "Desktop",
    Path(os.environ.get("PUBLIC", "")) / "Desktop",
]

LOG_FILE = Path.home() / "jarvis_error.log"

TRANSLIT = {
    "а": "a", "б": "b", "в": "v", "г": "g", "д": "d", "е": "e", "ё": "e",
    "ж": "zh", "з": "z", "и": "i", "й": "y", "к": "k", "л": "l", "м": "m",
    "н": "n", "о": "o", "п": "p", "р": "r", "с": "s", "т": "t", "у": "u",
    "ф": "f", "х": "h", "ц": "c", "ч": "ch", "ш": "sh", "щ": "sch",
    "ъ": "", "ы": "y", "ь": "", "э": "e", "ю": "yu", "я": "ya",
}

HELP_TEXT = (
    "Скажите «эй Джарвис» и назовите команду:\n\n"
    "• открой ютуб, открой гитхаб: сайты\n"
    "• открой телеграм, запусти стим: программы\n"
    "• открой файл отчёт: поиск файла или папки\n"
    "• скажи время\n"
    "• обнови программы\n"
    "• выключись\n\n"
    "Можно не говорить «Джарвис»: нажмите на микрофон."
)

# Состояние помощника, которое показывает окно:
# idle - ждёт слово активации, listening - слушает команду,
# speaking - говорит, error - что-то пошло не так
STATE = {"mode": "idle", "quit": False, "manual": False, "reload": False}

engine = None
recognizer = sr.Recognizer()
APP_INDEX = {}

# ---------------- Голос ----------------


def speak(text):
    print("Помощник:", text)
    STATE["mode"] = "speaking"
    engine.say(text)
    engine.runAndWait()
    STATE["mode"] = "idle"


def listen(timeout=None, limit=8):
    """Слушает микрофон. Возвращает распознанный текст или пустую строку."""
    with sr.Microphone() as source:
        recognizer.adjust_for_ambient_noise(source, duration=0.3)
        try:
            audio = recognizer.listen(source, timeout=timeout, phrase_time_limit=limit)
        except sr.WaitTimeoutError:
            return ""
    try:
        text = recognizer.recognize_google(audio, language="ru-RU").lower()
        print("Вы:", text)
        return text
    except sr.UnknownValueError:
        return ""
    except sr.RequestError:
        time.sleep(3)  # нет интернета, не крутим цикл вхолостую
        return ""


# ---------------- Приложения ----------------


def translit(text):
    return "".join(TRANSLIT.get(ch, ch) for ch in text)


def build_app_index():
    """Собирает все приложения: название -> (тип, что запускать)."""
    index = {}

    # Все приложения из «Пуска», включая Microsoft Store
    try:
        result = subprocess.run(
            ["powershell", "-NoProfile", "-Command",
             "[Console]::OutputEncoding=[Text.Encoding]::UTF8; "
             "Get-StartApps | ConvertTo-Json"],
            capture_output=True, text=True, encoding="utf-8", timeout=30,
            creationflags=0x08000000,  # не показывать окно PowerShell
        )
        data = json.loads(result.stdout)
        if isinstance(data, dict):
            data = [data]
        for app in data:
            index[app["Name"].lower()] = ("appid", app["AppID"])
    except Exception:
        pass

    # Ярлыки из «Пуска» и с рабочего стола
    for folder in SHORTCUT_DIRS:
        if folder.exists():
            for lnk in folder.rglob("*.lnk"):
                index.setdefault(lnk.stem.lower(), ("file", str(lnk)))

    return index


def find_app(name):
    name = ALIASES.get(name, name)
    for candidate in (name, translit(name)):
        # точное совпадение
        if candidate in APP_INDEX:
            return APP_INDEX[candidate]
        # слово входит в название
        if len(candidate) >= 3:
            found = [key for key in APP_INDEX if candidate in key]
            if found:
                return APP_INDEX[min(found, key=len)]
        # похожее название (порог защищает от случайных совпадений)
        close = difflib.get_close_matches(candidate, APP_INDEX.keys(), n=1, cutoff=0.75)
        if close:
            return APP_INDEX[close[0]]
    return None


def launch_app(name):
    command = COMMANDS.get(name)
    if command:
        try:
            subprocess.Popen(command)
            return True
        except FileNotFoundError:
            pass  # exe не найден в PATH, ищем среди установленных программ

    app = find_app(name)
    if not app:
        return False

    kind, target = app
    if kind == "appid":
        subprocess.Popen(["explorer", "shell:AppsFolder\\" + target])
    else:
        os.startfile(target)
    return True


def open_site(name):
    for key, url in SITES.items():
        if key in name:
            date = datetime.now().strftime("%d-%m-%Y")
            webbrowser.open(url.format(date=date))
            return True
    return False


def find_file(name):
    for folder in SEARCH_DIRS:
        for root, dirs, files in os.walk(folder):
            for item in files + dirs:
                if name in item.lower():
                    return os.path.join(root, item)
    return None


def get_name(text, words):
    # длинные фразы убираем первыми: «открой мне» раньше, чем «открой»
    for word in sorted(words, key=len, reverse=True):
        text = text.replace(word, "")
    return text.strip()


# ---------------- Команды ----------------


def handle(text):
    """Выполняет команду. Возвращает False, если нужно выключиться."""
    global APP_INDEX

    if "время" in text or "времени" in text or "который час" in text:
        speak("Сейчас " + datetime.now().strftime("%H:%M"))

    elif "обнови программы" in text:
        APP_INDEX = build_app_index()
        speak("Список программ обновлён")

    elif "открой файл" in text or "открой папку" in text:
        name = get_name(text, ("открой файл", "открой папку"))
        path = find_file(name)
        if path:
            speak("Открываю " + name)
            os.startfile(path)
        else:
            speak("Не нашёл " + name)

    elif any(word in text for word in TRIGGERS):
        name = get_name(text, TRIGGERS)
        if open_site(name) or launch_app(name):
            speak("Открываю " + name)
        else:
            speak("Не нашёл программу " + name)

    elif "выключись" in text or "выход" in text:
        speak("До связи, сэр")
        return False

    else:
        speak("Я не понял команду")

    return True


# ---------------- Работа помощника (в отдельном потоке) ----------------


def assistant():
    global engine, APP_INDEX
    try:
        try:
            import pythoncom
            pythoncom.CoInitialize()  # нужно для голоса в отдельном потоке
        except ImportError:
            pass

        engine = pyttsx3.init()
        APP_INDEX = build_app_index()
        speak("Приветствую вас, сэр")

        while not STATE["quit"]:
            if STATE["reload"]:
                STATE["reload"] = False
                APP_INDEX = build_app_index()
                speak("Список программ обновлён")

            heard = listen(timeout=2)

            manual = STATE["manual"]  # нажали на микрофон
            STATE["manual"] = False
            wake = next((w for w in WAKE_WORDS if w in heard), None)
            if not wake and not manual:
                if STATE["mode"] == "listening":
                    STATE["mode"] = "idle"
                continue

            # Команда может прозвучать сразу: «эй джарвис, открой ютуб»
            command = heard.split(wake, 1)[1].strip(" ,.!") if wake else ""
            if not command:
                speak("Слушаю, сэр")
                STATE["mode"] = "listening"
                command = listen(timeout=6, limit=10)
                if not command:
                    STATE["mode"] = "idle"
                    speak("Не расслышал")
                    continue

            STATE["mode"] = "listening"
            try:
                if not handle(command):
                    break
            except Exception:
                LOG_FILE.write_text(traceback.format_exc(), encoding="utf-8")
                STATE["mode"] = "error"
                time.sleep(2)
                STATE["mode"] = "idle"
    except Exception:
        LOG_FILE.write_text(traceback.format_exc(), encoding="utf-8")
        STATE["mode"] = "error"  # красный микрофон: смотрите jarvis_error.log
        return

    STATE["quit"] = True


# ---------------- Окно ----------------

W, H = 200, 126          # размер окна
MIC = (100, 82)          # центр кнопки микрофона
MIC_R = 27               # её радиус
TRANS = "#010203"        # этот цвет окно делает прозрачным

# Тема окна: "classic" - сине-зелёная, как окно голосового ввода Windows,
# "dark" - тёмная версия того же окна
THEME = "classic"

THEMES = {
    "classic": {
        "top": "#0a94be", "bottom": "#11a66c", "border": "#0c86a8", "icon": "#ffffff",
        "btn": "#2aa9a2", "btn_active": "#51b8b3", "ring_idle": "#3eb1aa",
        "colors": {"idle": "#ffffff", "listening": "#ffffff",
                   "speaking": "#ffe08a", "error": "#ff6b5e"},
    },
    "dark": {
        "top": "#112733", "bottom": "#0e2a22", "border": "#28424f", "icon": "#9fb8c6",
        "btn": "#1a323d", "btn_active": "#1f3b49", "ring_idle": "#2f4856",
        "colors": {"idle": "#9fb8c6", "listening": "#19d3ff",
                   "speaking": "#f2b84b", "error": "#ff5d4f"},
    },
}
T = THEMES[THEME]
TOP, BOTTOM, BORDER, ICON = T["top"], T["bottom"], T["border"], T["icon"]
COLORS = T["colors"]  # цвета состояний: ждёт, слушает, говорит, ошибка


def mix(c1, c2, k):
    """Смешивает два цвета: k=0 даёт c1, k=1 даёт c2."""
    a = [int(c1[i:i + 2], 16) for i in (1, 3, 5)]
    b = [int(c2[i:i + 2], 16) for i in (1, 3, 5)]
    return "#%02x%02x%02x" % tuple(round(x + (y - x) * k) for x, y in zip(a, b))


class MicWidget:
    RADIUS = 12  # скругление углов

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Jarvis")
        self.root.overrideredirect(True)           # без рамки
        self.root.attributes("-topmost", True)     # поверх всех окон
        try:
            self.root.attributes("-transparentcolor", TRANS)
        except tk.TclError:
            pass
        self.root.configure(bg=TRANS)

        sw, sh = self.root.winfo_screenwidth(), self.root.winfo_screenheight()
        self.root.geometry("%dx%d+%d+%d" % (W, H, (sw - W) // 2, (sh - H) // 2))  # по центру экрана

        self.canvas = tk.Canvas(self.root, width=W, height=H, bg=TRANS, highlightthickness=0)
        self.canvas.pack()
        self.draw_panel()

        self.t = 0
        self.dx = self.dy = 0
        self.dragging = False

        self.top_var = tk.BooleanVar(value=True)
        self.menu = tk.Menu(self.root, tearoff=0)
        self.menu.add_command(label="Обновить список программ",
                              command=lambda: STATE.update(reload=True))
        self.menu.add_command(label="Открыть файл ошибок", command=self.open_log)
        self.menu.add_checkbutton(label="Поверх всех окон", variable=self.top_var,
                                  command=self.toggle_top)
        self.menu.add_separator()
        self.menu.add_command(label="Выход", command=self.close)

        self.canvas.bind("<ButtonPress-1>", self.on_press)
        self.canvas.bind("<B1-Motion>", self.on_drag)
        self.canvas.bind("<ButtonRelease-1>", lambda e: setattr(self, "dragging", False))
        self.canvas.bind("<Button-3>", lambda e: self.menu.tk_popup(e.x_root, e.y_root))

        self.animate()

    # --- фон панели: тёмный градиент со скруглёнными углами ---
    def draw_panel(self):
        c, r = self.canvas, self.RADIUS
        for y in range(H):
            yy = y + 0.5
            if yy < r:
                inset = r - math.sqrt(r * r - (r - yy) ** 2)
            elif yy > H - r:
                inset = r - math.sqrt(r * r - (yy - (H - r)) ** 2)
            else:
                inset = 0
            x0, x1 = round(inset), W - round(inset)
            c.create_line(x0, y, x1, y, fill=BORDER)
            if 0 < y < H - 1:
                c.create_line(x0 + 1, y, x1 - 1, y, fill=mix(TOP, BOTTOM, y / H))
        # разделитель под верхней полосой
        c.create_line(1, 40, W - 1, 40, fill=mix(BORDER, TOP, 0.4))

    # --- кнопки ---
    def on_press(self, e):
        x, y = e.x, e.y
        if y < 40 and x > W - 34:                       # крестик
            self.close()
        elif math.hypot(x - MIC[0], y - MIC[1]) <= MIC_R + 3:   # микрофон
            STATE["manual"] = True
            if STATE["mode"] == "idle":
                STATE["mode"] = "listening"
        elif abs(x - 32) <= 18 and 62 <= y <= 102:      # шестерёнка
            self.menu.tk_popup(e.x_root, e.y_root)
        elif abs(x - 168) <= 18 and 62 <= y <= 102:     # вопрос
            messagebox.showinfo("Jarvis", HELP_TEXT, parent=self.root)
        else:                                           # остальное: тянем окно
            self.dragging = True
            self.dx, self.dy = x, y

    def on_drag(self, e):
        if self.dragging:
            x = self.root.winfo_x() + e.x - self.dx
            y = self.root.winfo_y() + e.y - self.dy
            self.root.geometry("+%d+%d" % (x, y))

    def toggle_top(self):
        self.root.attributes("-topmost", self.top_var.get())

    def open_log(self):
        if LOG_FILE.exists():
            os.startfile(LOG_FILE)
        else:
            messagebox.showinfo("Jarvis", "Ошибок нет.", parent=self.root)

    def close(self):
        STATE["quit"] = True
        self.root.destroy()

    # --- анимация ---
    def animate(self):
        if STATE["quit"]:
            self.root.destroy()
            return

        mode = STATE["mode"]
        color = COLORS.get(mode, COLORS["idle"])
        c = self.canvas
        c.delete("dyn")
        mx, my = MIC
        self.t += 1

        # верхняя полоса: ручка и крестик
        c.create_line(W // 2 - 20, 20, W // 2 + 20, 20, width=3, capstyle="round",
                      fill=ICON, tags="dyn")
        c.create_line(W - 29, 14, W - 19, 24, width=2, fill=ICON, tags="dyn")
        c.create_line(W - 29, 24, W - 19, 14, width=2, fill=ICON, tags="dyn")

        # шестерёнка слева
        gx, gy = 32, my
        for k in range(8):
            a = k * math.pi / 4
            c.create_line(gx + 6 * math.cos(a), gy + 6 * math.sin(a),
                          gx + 10 * math.cos(a), gy + 10 * math.sin(a),
                          width=3, fill=ICON, tags="dyn")
        c.create_oval(gx - 7, gy - 7, gx + 7, gy + 7, outline=ICON, width=2, tags="dyn")
        c.create_oval(gx - 2.5, gy - 2.5, gx + 2.5, gy + 2.5, outline=ICON, width=2, tags="dyn")

        # вопрос справа
        qx, qy = 168, my
        c.create_oval(qx - 10, qy - 10, qx + 10, qy + 10, outline=ICON, width=2, tags="dyn")
        c.create_text(qx, qy, text="?", fill=ICON, font=("Segoe UI", 11, "bold"), tags="dyn")

        # кольца вокруг микрофона
        if mode == "idle":
            pulse = (math.sin(self.t / 25) + 1) / 2
            r = MIC_R + 3 + 1.5 * pulse
            c.create_oval(mx - r, my - r, mx + r, my + r,
                          outline=mix(color, mix(TOP, BOTTOM, 0.65), 0.6), width=2, tags="dyn")
        else:
            for i in range(2):
                phase = ((self.t / 30) + i * 0.5) % 1
                r = MIC_R + 2 + 11 * phase
                c.create_oval(mx - r, my - r, mx + r, my + r,
                              outline=mix(color, mix(TOP, BOTTOM, 0.65), phase), width=2, tags="dyn")

        # кнопка микрофона
        ring = T["ring_idle"] if mode == "idle" else color
        c.create_oval(mx - MIC_R, my - MIC_R, mx + MIC_R, my + MIC_R,
                      fill=T["btn"] if mode == "idle" else T["btn_active"], outline=ring, width=2, tags="dyn")
        mic = ICON if mode == "idle" else color
        c.create_line(mx, my - 9, mx, my - 2, width=11, capstyle="round", fill=mic, tags="dyn")
        c.create_arc(mx - 10, my - 12, mx + 10, my + 8, start=180, extent=180,
                     style="arc", width=2, outline=mic, tags="dyn")
        c.create_line(mx, my + 8, mx, my + 13, width=2, fill=mic, tags="dyn")
        c.create_line(mx - 5, my + 13, mx + 5, my + 13, width=2, capstyle="round",
                      fill=mic, tags="dyn")

        self.root.after(40, self.animate)

    def run(self):
        self.root.mainloop()


if __name__ == "__main__":
    threading.Thread(target=assistant, daemon=True).start()
    MicWidget().run()
