from datetime import datetime
from pathlib import Path
import difflib
import json
import os
import subprocess
import webbrowser

import pyttsx3
import speech_recognition as sr

engine = pyttsx3.init()
recognizer = sr.Recognizer()

# Программы без ярлыка: название -> путь к .exe или команда
COMMANDS = {
    "блокнот": "notepad.exe",
    "калькулятор": "calc.exe",
    "телеграмм":"unigram.exe"
}

# Слова, которые не получается угадать автоматически
ALIASES = {
    "стим": "steam",
    "хром": "chrome",
    "фотошоп": "photoshop",
    "юниграм":"unigram"
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

TRANSLIT = {
    "а": "a", "б": "b", "в": "v", "г": "g", "д": "d", "е": "e", "ё": "e",
    "ж": "zh", "з": "z", "и": "i", "й": "y", "к": "k", "л": "l", "м": "m",
    "н": "n", "о": "o", "п": "p", "р": "r", "с": "s", "т": "t", "у": "u",
    "ф": "f", "х": "h", "ц": "c", "ч": "ch", "ш": "sh", "щ": "sch",
    "ъ": "", "ы": "y", "ь": "", "э": "e", "ю": "yu", "я": "ya",
}


def speak(text):
    print("Помощник:", text)
    engine.say(text)
    engine.runAndWait()


def listen():
    with sr.Microphone() as source:
        print("Слушаю...")
        recognizer.adjust_for_ambient_noise(source, duration=0.5)
        audio = recognizer.listen(source)
    try:
        text = recognizer.recognize_google(audio, language="ru-RU")
        print("Вы:", text)
        return text.lower()
    except sr.UnknownValueError:
        return ""
    except sr.RequestError:
        speak("Нет связи с сервисом распознавания")
        return ""


def translit(text):
    return "".join(TRANSLIT.get(ch, ch) for ch in text)


def build_app_index():
    """Собирает все приложения: название -> (тип, что запускать)."""
    index = {}

    # 1. Все приложения из «Пуска», включая Microsoft Store
    try:
        result = subprocess.run(
            ["powershell", "-NoProfile", "-Command",
             "[Console]::OutputEncoding=[Text.Encoding]::UTF8; "
             "Get-StartApps | ConvertTo-Json"],
            capture_output=True, text=True, encoding="utf-8", timeout=30,
        )
        data = json.loads(result.stdout)
        if isinstance(data, dict):
            data = [data]
        for app in data:
            index[app["Name"].lower()] = ("appid", app["AppID"])
    except Exception:
        pass

    # 2. Ярлыки из «Пуска» и с рабочего стола
    for folder in SHORTCUT_DIRS:
        if folder.exists():
            for lnk in folder.rglob("*.lnk"):
                index.setdefault(lnk.stem.lower(), ("file", str(lnk)))

    return index


APP_INDEX = build_app_index()


def find_app(name):
    name = ALIASES.get(name, name)
    for candidate in (name, translit(name)):
        # точное совпадение
        if candidate in APP_INDEX:
            return APP_INDEX[candidate]
        # слово входит в название ("telegram" -> "unigram")
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
        subprocess.Popen(command)
        return True

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
    for word in words:
        text = text.replace(word, "")
    return text.strip()


print("Найдено программ:", len(APP_INDEX))
speak("Приветствую вас, сэр")

while True:
    text = listen()
    if not text:
        continue

    if "время" in text:
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
        if open_site(name):
            speak("Открываю " + name)
        elif launch_app(name):
            speak("Открываю " + name)
        else:
            speak("Не нашёл программу " + name)

    elif "выход" in text:
        speak("Пока!")
        break

    else:
        speak("Я не понял команду")
