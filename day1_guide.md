# 📋 День 1 — Подготовка сервера
## Пошаговая инструкция (для новичка)

> **Время:** ~30-40 минут
> **Что понадобится:** приложение «Терминал» на вашем Mac
> **Сервер:** 185.178.47.148 (Timeweb Cloud, yookassa-api)

---

## Шаг 1. Найти пароль от сервера

**Где искать:**
1. Откройте [timeweb.cloud](https://timeweb.cloud) в браузере
2. Слева нажмите **«Облачные серверы»**
3. Нажмите на **«yookassa-api»**
4. Перейдите на вкладку **«Доступ»**
5. Там будет **пароль root** — скопируйте его

> Если пароля нет — нажмите «Сбросить пароль root». Новый пароль придёт на почту.

---

## Шаг 2. Открыть Терминал на Mac

**Как найти:**
1. Нажмите **Cmd + Пробел** (откроется Spotlight)
2. Введите **Terminal** или **Терминал**
3. Нажмите Enter

Откроется чёрное (или белое) окно с мигающим курсором — это и есть терминал.

---

## Шаг 3. Подключиться к серверу

В терминале введите эту команду и нажмите **Enter**:

```bash
ssh root@185.178.47.148
```

**Что произойдёт:**

Вариант А — первое подключение:
```
The authenticity of host '185.178.47.148' can't be established.
Are you sure you want to continue connecting (yes/no)?
```
Введите **yes** и нажмите Enter.

Вариант Б — сразу спросит пароль:
```
root@185.178.47.148's password:
```

Введите пароль, который нашли в Шаге 1.

> ⚠️ **Важно:** когда вводите пароль, **символы НЕ отображаются** — это нормально! Просто введите пароль и нажмите Enter.

**Если подключились успешно**, увидите что-то вроде:
```
Welcome to Ubuntu 24.04
root@yookassa-api:~#
```

🎉 **Вы на сервере!** Теперь все команды будут выполняться на сервере, а не на вашем Mac.

---

## Шаг 4. Проверить сервер

Сейчас мы проверим, что сервер обновился и всё работает. **Копируйте каждую команду** и вставляйте в терминал (Cmd+V), затем нажимайте Enter.

### 4.1 — Проверить память

```bash
free -h
```

Должно показать что-то вроде:
```
              total        used        free
Mem:          1.9Gi       500Mi       1.2Gi
```

Если в строке `total` написано **~1.9Gi или 2.0Gi** — значит апгрейд до 2 ГБ прошёл успешно ✅

### 4.2 — Проверить диск

```bash
df -h /
```

Должно показать что-то вроде:
```
Filesystem      Size  Used Avail Use%
/dev/vda1        29G   5G   23G  18%
```

Если Size около **29-30G** — отлично, диск 30 ГБ подключён ✅

### 4.3 — Проверить, что YooKassa работает

```bash
curl -s http://localhost:* 2>/dev/null || echo "Проверим вручную"
ps aux | grep -i yoo
```

> Скиньте мне вывод этих команд. Я посмотрю, как YooKassa запущена, чтобы случайно не мешать ей.

---

## Шаг 5. Обновить систему

Эта команда обновит все программы на сервере до последних версий. Это как обновление приложений на телефоне — просто для безопасности.

```bash
apt update && apt upgrade -y
```

> ⏱ Займёт **1-3 минуты**. Ждите, пока не появится мигающий курсор снова.

Если спросит что-то вроде «Do you want to continue?» — нажмите **Enter** (или введите Y).

Если покажет розовое/фиолетовое окно с вопросом о перезапуске сервисов — просто нажмите **Enter** (выбрать «OK»).

---

## Шаг 6. Установить нужные программы

Эта команда установит всё необходимое за один раз:

```bash
apt install -y python3.11 python3.11-venv python3-pip nginx certbot python3-certbot-nginx git unzip curl
```

> ⏱ Займёт **1-2 минуты**.

### 6.1 — Проверить, что всё установилось

Теперь проверим, что всё на месте. Копируйте всё целиком:

```bash
echo "=== Python ===" && python3.11 --version && echo "=== Nginx ===" && nginx -v && echo "=== Git ===" && git --version && echo "=== Всё установлено! ==="
```

Должно показать:
```
=== Python ===
Python 3.11.x
=== Nginx ===
nginx version: nginx/1.x.x
=== Git ===
git version 2.x.x
=== Всё установлено! ===
```

Если видите все три версии — всё ок ✅

> **Если Python 3.11 не нашёлся**, попробуйте:
> ```bash
> apt install -y software-properties-common
> add-apt-repository ppa:deadsnakes/ppa -y
> apt update
> apt install -y python3.11 python3.11-venv
> ```

---

## Шаг 7. Создать папки для DriveGPT

```bash
mkdir -p /opt/drivegpt/backend
mkdir -p /opt/drivegpt/bot-projects
mkdir -p /opt/drivegpt/uploads
mkdir -p /opt/drivegpt/data
```

### Проверить, что папки создались:

```bash
ls -la /opt/drivegpt/
```

Должно показать:
```
drwxr-xr-x backend
drwxr-xr-x bot-projects
drwxr-xr-x data
drwxr-xr-x uploads
```

4 папки — всё на месте ✅

---

## Шаг 8. Создать виртуальное окружение Python

Виртуальное окружение — это «песочница» для Python. Все библиотеки DriveGPT будут установлены внутри неё, не мешая остальной системе и YooKassa.

```bash
cd /opt/drivegpt/backend
python3.11 -m venv venv
```

### Активировать виртуальное окружение:

```bash
source /opt/drivegpt/backend/venv/bin/activate
```

> После этой команды в начале строки появится **(venv)** — это значит, вы внутри виртуального окружения:
> ```
> (venv) root@yookassa-api:/opt/drivegpt/backend#
> ```

### Установить библиотеки:

```bash
pip install fastapi uvicorn sqlalchemy python-multipart aiofiles httpx
```

> ⏱ Займёт **1 минуту**.

### Проверить:

```bash
python -c "import fastapi; print(f'FastAPI {fastapi.__version__} установлена!')"
```

Должно показать: `FastAPI 0.11x.x установлена!` ✅

---

## Шаг 9. Создать тестовый файл бэкенда

Сейчас создадим простейший сервер, чтобы убедиться, что всё работает:

```bash
cat > /opt/drivegpt/backend/main.py << 'EOF'
from fastapi import FastAPI

app = FastAPI(title="DriveGPT Bot Hosting")

@app.get("/")
def root():
    return {"status": "ok", "service": "DriveGPT", "message": "Сервер работает!"}

@app.get("/health")
def health():
    return {"healthy": True}
EOF
```

> Эта команда создаёт файл `main.py`. Не нужно ничего редактировать — просто скопируйте и вставьте в терминал.

---

## Шаг 10. Запустить и проверить

```bash
cd /opt/drivegpt/backend
source venv/bin/activate
uvicorn main:app --host 0.0.0.0 --port 8000
```

Должно показать:
```
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
INFO:     Started server process
INFO:     Application startup complete.
```

### Проверить из браузера:

Откройте **новую вкладку** в браузере на вашем Mac и перейдите по ссылке:

```
http://185.178.47.148:8000
```

Должно показать:
```json
{"status":"ok","service":"DriveGPT","message":"Сервер работает!"}
```

🎉 **Если видите это — поздравляю, бэкенд работает!**

### Остановить тестовый сервер:

Вернитесь в терминал и нажмите **Ctrl + C** (зажмите Ctrl и нажмите C).

---

## Шаг 11. Настроить автозапуск

Сейчас настроим, чтобы бэкенд запускался автоматически при перезагрузке сервера:

```bash
cat > /etc/systemd/system/drivegpt.service << 'EOF'
[Unit]
Description=DriveGPT Backend
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory=/opt/drivegpt/backend
Environment="PATH=/opt/drivegpt/backend/venv/bin"
ExecStart=/opt/drivegpt/backend/venv/bin/uvicorn main:app --host 127.0.0.1 --port 8000
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Теперь активировать и запустить:

```bash
systemctl daemon-reload
systemctl enable drivegpt
systemctl start drivegpt
```

### Проверить, что работает:

```bash
systemctl status drivegpt
```

Должно показать:
```
● drivegpt.service - DriveGPT Backend
     Active: active (running)
```

Если видите **active (running)** — автозапуск настроен ✅

---

## ✅ Итоги дня 1

Если вы дошли до этого места — вот что вы сделали:

| Шаг | Что сделали | Статус |
|-----|-----------|:------:|
| 1 | Нашли пароль SSH | ✅ |
| 2 | Открыли терминал | ✅ |
| 3 | Подключились к серверу | ✅ |
| 4 | Проверили RAM и диск | ✅ |
| 5 | Обновили систему | ✅ |
| 6 | Установили Python, Nginx, Git | ✅ |
| 7 | Создали папки проекта | ✅ |
| 8 | Создали виртуальное окружение | ✅ |
| 9 | Создали тестовый бэкенд | ✅ |
| 10 | Проверили — сервер отвечает | ✅ |
| 11 | Настроили автозапуск | ✅ |

---

## 🆘 Если что-то пошло не так

### «Permission denied» при подключении
— Неправильный пароль. Сбросьте его в панели Timeweb.

### «Connection refused» или «Connection timed out»
— Сервер ещё перезагружается после апгрейда. Подождите 2-3 минуты и попробуйте снова.

### «python3.11: command not found»
— Используйте команду установки из раздела 6 (с `add-apt-repository`).

### «Address already in use» при запуске uvicorn
— Порт 8000 уже занят. Попробуйте порт 8001:
```bash
uvicorn main:app --host 0.0.0.0 --port 8001
```

### Не могу вставить в терминал
— На Mac: **Cmd + V**. Если не работает — правый клик → «Paste».

### Любая другая ошибка
— **Скопируйте текст ошибки** и отправьте мне — разберёмся!

---

## ➡️ Следующий шаг

После завершения дня 1, напишите мне:
1. Скриншот или текст из команды `systemctl status drivegpt`
2. Скриншот из браузера `http://185.178.47.148:8000`

И мы перейдём к **Дню 2** — настройка домена drivegpt.pro и HTTPS.
