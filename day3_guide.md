# 📋 День 3 — Webhook-роутер, менеджер ботов, загрузка кода
## Пошаговая инструкция

> **Время:** ~1-1.5 часа
> **Что понадобится:** Терминал (вы уже подключены к серверу)
> **Сервер:** 185.178.47.148

---

## Шаг 1. Подключиться к серверу

Если ещё не подключены:
```
ssh -o ServerAliveInterval=60 root@185.178.47.148
```

Перейти в папку и активировать окружение:
```
cd /opt/drivegpt/backend && source venv/bin/activate
```

---

## Шаг 2. Создать структуру роутеров

Вместо одного длинного `main.py` разобьём код на модули:

```
mkdir -p /opt/drivegpt/backend/routers
mkdir -p /opt/drivegpt/backend/services
```

---

## Шаг 3. Сервис работы с Telegram API

Этот файл будет общаться с Telegram — регистрировать/удалять вебхуки, отправлять сообщения:

```
cat > /opt/drivegpt/backend/services/telegram_api.py << 'EOF'
import httpx

TELEGRAM_API = "https://api.telegram.org/bot{token}"


async def get_bot_info(token: str) -> dict:
    """Получить информацию о боте по токену"""
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.get(f"{TELEGRAM_API.format(token=token)}/getMe")
            data = resp.json()
            if data.get("ok"):
                bot = data["result"]
                return {
                    "ok": True,
                    "username": bot.get("username"),
                    "first_name": bot.get("first_name"),
                    "id": bot.get("id"),
                }
            return {"ok": False, "error": data.get("description", "Неизвестная ошибка")}
    except Exception as e:
        return {"ok": False, "error": str(e)}


async def set_webhook(token: str, webhook_url: str) -> dict:
    """Зарегистрировать webhook в Telegram"""
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{TELEGRAM_API.format(token=token)}/setWebhook",
                json={"url": webhook_url}
            )
            return resp.json()
    except Exception as e:
        return {"ok": False, "error": str(e)}


async def delete_webhook(token: str) -> dict:
    """Удалить webhook из Telegram"""
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{TELEGRAM_API.format(token=token)}/deleteWebhook"
            )
            return resp.json()
    except Exception as e:
        return {"ok": False, "error": str(e)}


async def send_message(token: str, chat_id: str, text: str) -> dict:
    """Отправить сообщение пользователю от имени бота"""
    try:
        async with httpx.AsyncClient() as client:
            resp = await client.post(
                f"{TELEGRAM_API.format(token=token)}/sendMessage",
                json={"chat_id": chat_id, "text": text}
            )
            return resp.json()
    except Exception as e:
        return {"ok": False, "error": str(e)}
EOF
```

---

## Шаг 4. Менеджер ботов

Этот сервис управляет запуском и остановкой ботов через systemd:

```
cat > /opt/drivegpt/backend/services/bot_manager.py << 'EOF'
import os
import subprocess
import shutil
from config import BOT_PROJECTS_DIR


def get_project_dir(project_id: int) -> str:
    """Путь к папке проекта"""
    return os.path.join(BOT_PROJECTS_DIR, f"project-{project_id}")


def get_service_name(project_id: int) -> str:
    """Имя systemd сервиса для бота"""
    return f"drivegpt-bot-{project_id}"


def setup_bot_environment(project_id: int) -> dict:
    """Создать virtualenv и установить зависимости для бота"""
    project_dir = get_project_dir(project_id)
    venv_dir = os.path.join(project_dir, "venv")

    if not os.path.exists(project_dir):
        return {"ok": False, "error": "Папка проекта не найдена. Загрузите код."}

    # Создать virtualenv
    try:
        subprocess.run(
            ["python3", "-m", "venv", venv_dir],
            check=True, capture_output=True, text=True
        )
    except subprocess.CalledProcessError as e:
        return {"ok": False, "error": f"Ошибка создания venv: {e.stderr}"}

    # Установить зависимости из requirements.txt
    req_file = os.path.join(project_dir, "requirements.txt")
    if os.path.exists(req_file):
        try:
            pip_path = os.path.join(venv_dir, "bin", "pip")
            subprocess.run(
                [pip_path, "install", "-r", req_file],
                check=True, capture_output=True, text=True, timeout=120
            )
        except subprocess.CalledProcessError as e:
            return {"ok": False, "error": f"Ошибка установки зависимостей: {e.stderr[:500]}"}
        except subprocess.TimeoutExpired:
            return {"ok": False, "error": "Таймаут установки зависимостей (>2 мин)"}

    return {"ok": True}


def check_syntax(project_id: int) -> dict:
    """Проверить синтаксис Python-файлов"""
    project_dir = get_project_dir(project_id)
    errors = []

    for root, dirs, files in os.walk(project_dir):
        # Пропускаем venv
        if "venv" in root:
            continue
        for f in files:
            if f.endswith(".py"):
                filepath = os.path.join(root, f)
                try:
                    subprocess.run(
                        ["python3", "-m", "py_compile", filepath],
                        check=True, capture_output=True, text=True
                    )
                except subprocess.CalledProcessError as e:
                    errors.append({"file": f, "error": e.stderr.strip()})

    if errors:
        return {"ok": False, "errors": errors}
    return {"ok": True, "message": "Синтаксис ОК"}


def find_main_file(project_id: int) -> str:
    """Найти главный файл бота"""
    project_dir = get_project_dir(project_id)

    # Приоритет: bot.py > main.py > app.py
    for name in ["bot.py", "main.py", "app.py"]:
        if os.path.exists(os.path.join(project_dir, name)):
            return name

    # Если не нашли — ищем любой .py файл
    for f in os.listdir(project_dir):
        if f.endswith(".py") and f != "__init__.py":
            return f

    return None


def create_systemd_service(project_id: int, internal_port: int) -> dict:
    """Создать systemd сервис для бота"""
    project_dir = get_project_dir(project_id)
    service_name = get_service_name(project_id)
    main_file = find_main_file(project_id)

    if not main_file:
        return {"ok": False, "error": "Не найден Python-файл (bot.py, main.py или app.py)"}

    venv_python = os.path.join(project_dir, "venv", "bin", "python")
    main_path = os.path.join(project_dir, main_file)

    service_content = f"""[Unit]
Description=DriveGPT Bot {project_id}
After=network.target

[Service]
Type=simple
User=root
WorkingDirectory={project_dir}
Environment="PATH={project_dir}/venv/bin"
Environment="DRIVEGPT_API_URL=http://127.0.0.1:8000"
Environment="DRIVEGPT_PROJECT_ID={project_id}"
Environment="BOT_INTERNAL_PORT={internal_port}"
ExecStart={venv_python} {main_path}
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
"""

    service_path = f"/etc/systemd/system/{service_name}.service"
    try:
        with open(service_path, "w") as f:
            f.write(service_content)
        subprocess.run(["systemctl", "daemon-reload"], check=True)
        return {"ok": True, "service_name": service_name}
    except Exception as e:
        return {"ok": False, "error": str(e)}


def start_bot(project_id: int) -> dict:
    """Запустить бота"""
    service_name = get_service_name(project_id)
    try:
        subprocess.run(
            ["systemctl", "start", f"{service_name}.service"],
            check=True, capture_output=True, text=True
        )
        return {"ok": True}
    except subprocess.CalledProcessError as e:
        return {"ok": False, "error": e.stderr}


def stop_bot(project_id: int) -> dict:
    """Остановить бота"""
    service_name = get_service_name(project_id)
    try:
        subprocess.run(
            ["systemctl", "stop", f"{service_name}.service"],
            check=True, capture_output=True, text=True
        )
        return {"ok": True}
    except subprocess.CalledProcessError as e:
        return {"ok": False, "error": e.stderr}


def get_bot_status(project_id: int) -> str:
    """Получить статус бота"""
    service_name = get_service_name(project_id)
    try:
        result = subprocess.run(
            ["systemctl", "is-active", f"{service_name}.service"],
            capture_output=True, text=True
        )
        return result.stdout.strip()  # active, inactive, failed
    except Exception:
        return "unknown"


def get_bot_logs(project_id: int, lines: int = 50) -> str:
    """Получить последние логи бота"""
    service_name = get_service_name(project_id)
    try:
        result = subprocess.run(
            ["journalctl", "-u", f"{service_name}.service", "-n", str(lines), "--no-pager"],
            capture_output=True, text=True
        )
        return result.stdout
    except Exception as e:
        return f"Ошибка: {e}"


def cleanup_bot(project_id: int):
    """Полная очистка бота (при удалении проекта)"""
    service_name = get_service_name(project_id)
    project_dir = get_project_dir(project_id)

    # Остановить сервис
    subprocess.run(["systemctl", "stop", f"{service_name}.service"], capture_output=True)
    subprocess.run(["systemctl", "disable", f"{service_name}.service"], capture_output=True)

    # Удалить файл сервиса
    service_path = f"/etc/systemd/system/{service_name}.service"
    if os.path.exists(service_path):
        os.remove(service_path)

    subprocess.run(["systemctl", "daemon-reload"], capture_output=True)

    # Удалить папку проекта
    if os.path.exists(project_dir):
        shutil.rmtree(project_dir)
EOF
```

---

## Шаг 5. Роутер для управления проектами и ботами

```
cat > /opt/drivegpt/backend/routers/projects.py << 'EOF'
import os
import shutil
import zipfile
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File
from sqlalchemy.orm import Session
from models import get_db, Project
from services import telegram_api, bot_manager
from config import BOT_PROJECTS_DIR

router = APIRouter(prefix="/api/projects", tags=["projects"])

# Порты для ботов (каждый бот получает свой порт)
BOT_PORT_START = 9001


@router.get("")
def list_projects(db: Session = Depends(get_db)):
    projects = db.query(Project).all()
    result = []
    for p in projects:
        real_status = bot_manager.get_bot_status(p.id)
        result.append({
            "id": p.id, "name": p.name,
            "status": real_status if real_status != "unknown" else p.status,
            "bot_username": p.bot_username,
            "bot_name": p.bot_name,
            "messenger_type": p.messenger_type,
            "created_at": str(p.created_at),
        })
    return result


@router.post("")
def create_project(name: str, db: Session = Depends(get_db)):
    project = Project(name=name)
    db.add(project)
    db.commit()
    db.refresh(project)

    # Создать папку для проекта
    project_dir = bot_manager.get_project_dir(project.id)
    os.makedirs(project_dir, exist_ok=True)

    return {"id": project.id, "name": project.name, "status": project.status}


@router.get("/{project_id}")
def get_project(project_id: int, db: Session = Depends(get_db)):
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    real_status = bot_manager.get_bot_status(project_id)
    return {
        "id": project.id, "name": project.name,
        "telegram_token": "***" + (project.telegram_token[-4:] if project.telegram_token else ""),
        "bot_username": project.bot_username,
        "bot_name": project.bot_name,
        "status": real_status if real_status != "unknown" else project.status,
        "messenger_type": project.messenger_type,
        "error_message": project.error_message,
        "created_at": str(project.created_at),
    }


@router.put("/{project_id}/token")
async def set_token(project_id: int, token: str, db: Session = Depends(get_db)):
    """Установить Telegram Bot Token и получить инфо о боте"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    # Проверить токен через Telegram API
    bot_info = await telegram_api.get_bot_info(token)
    if not bot_info["ok"]:
        raise HTTPException(status_code=400, detail=f"Неверный токен: {bot_info['error']}")

    project.telegram_token = token
    project.bot_username = bot_info["username"]
    project.bot_name = bot_info["first_name"]
    db.commit()

    return {
        "status": "ok",
        "bot_username": bot_info["username"],
        "bot_name": bot_info["first_name"],
    }


@router.post("/{project_id}/upload")
async def upload_code(project_id: int, file: UploadFile = File(...), db: Session = Depends(get_db)):
    """Загрузить ZIP с кодом бота"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    project_dir = bot_manager.get_project_dir(project_id)

    # Очистить папку (кроме venv)
    if os.path.exists(project_dir):
        for item in os.listdir(project_dir):
            if item == "venv":
                continue
            item_path = os.path.join(project_dir, item)
            if os.path.isdir(item_path):
                shutil.rmtree(item_path)
            else:
                os.remove(item_path)
    else:
        os.makedirs(project_dir)

    # Сохранить загруженный файл
    tmp_path = f"/tmp/upload-{project_id}.zip"
    with open(tmp_path, "wb") as f:
        content = await file.read()
        f.write(content)

    # Распаковать ZIP
    try:
        with zipfile.ZipFile(tmp_path, 'r') as zf:
            zf.extractall(project_dir)
    except zipfile.BadZipFile:
        os.remove(tmp_path)
        raise HTTPException(status_code=400, detail="Файл не является ZIP-архивом")

    os.remove(tmp_path)

    # Список файлов
    files = []
    for root, dirs, filenames in os.walk(project_dir):
        if "venv" in root:
            continue
        for fn in filenames:
            rel_path = os.path.relpath(os.path.join(root, fn), project_dir)
            files.append(rel_path)

    return {"status": "ok", "files": files, "count": len(files)}


@router.get("/{project_id}/files")
def list_files(project_id: int):
    """Список файлов проекта"""
    project_dir = bot_manager.get_project_dir(project_id)
    if not os.path.exists(project_dir):
        return {"files": []}

    files = []
    for root, dirs, filenames in os.walk(project_dir):
        if "venv" in root or "__pycache__" in root:
            continue
        for fn in filenames:
            rel_path = os.path.relpath(os.path.join(root, fn), project_dir)
            size = os.path.getsize(os.path.join(root, fn))
            files.append({"path": rel_path, "size": size})

    return {"files": files}


@router.get("/{project_id}/files/{file_path:path}")
def get_file_content(project_id: int, file_path: str):
    """Получить содержимое файла"""
    project_dir = bot_manager.get_project_dir(project_id)
    full_path = os.path.join(project_dir, file_path)

    if not os.path.exists(full_path) or not os.path.isfile(full_path):
        raise HTTPException(status_code=404, detail="Файл не найден")

    try:
        with open(full_path, "r") as f:
            content = f.read()
        return {"path": file_path, "content": content}
    except UnicodeDecodeError:
        return {"path": file_path, "content": "[бинарный файл]"}


@router.post("/{project_id}/check")
def check_code(project_id: int):
    """Проверить синтаксис Python-кода"""
    result = bot_manager.check_syntax(project_id)
    return result


@router.post("/{project_id}/start")
async def start_project(project_id: int, db: Session = Depends(get_db)):
    """Запустить бота"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    if not project.telegram_token:
        raise HTTPException(status_code=400, detail="Сначала укажите Telegram Bot Token")

    # Проверить синтаксис
    syntax = bot_manager.check_syntax(project_id)
    if not syntax["ok"]:
        project.status = "error"
        project.error_message = str(syntax["errors"])
        db.commit()
        raise HTTPException(status_code=400, detail=f"Ошибка синтаксиса: {syntax['errors']}")

    # Установить зависимости
    env_result = bot_manager.setup_bot_environment(project_id)
    if not env_result["ok"]:
        project.status = "error"
        project.error_message = env_result["error"]
        db.commit()
        raise HTTPException(status_code=500, detail=env_result["error"])

    # Создать systemd сервис
    internal_port = BOT_PORT_START + project_id
    svc_result = bot_manager.create_systemd_service(project_id, internal_port)
    if not svc_result["ok"]:
        raise HTTPException(status_code=500, detail=svc_result["error"])

    # Запустить бота
    start_result = bot_manager.start_bot(project_id)
    if not start_result["ok"]:
        project.status = "error"
        project.error_message = start_result["error"]
        db.commit()
        raise HTTPException(status_code=500, detail=start_result["error"])

    # Зарегистрировать webhook
    webhook_url = f"https://drivegpt.pro/webhook/{project_id}"
    wh_result = await telegram_api.set_webhook(project.telegram_token, webhook_url)

    project.status = "running"
    project.webhook_url = webhook_url
    project.error_message = None
    db.commit()

    return {"status": "running", "webhook_url": webhook_url}


@router.post("/{project_id}/stop")
async def stop_project(project_id: int, db: Session = Depends(get_db)):
    """Остановить бота"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    # Остановить бота
    bot_manager.stop_bot(project_id)

    # Удалить webhook
    if project.telegram_token:
        await telegram_api.delete_webhook(project.telegram_token)

    project.status = "stopped"
    project.webhook_url = None
    db.commit()

    return {"status": "stopped"}


@router.get("/{project_id}/status")
def project_status(project_id: int, db: Session = Depends(get_db)):
    """Статус бота"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    real_status = bot_manager.get_bot_status(project_id)
    return {
        "status": real_status,
        "webhook_url": project.webhook_url,
        "error_message": project.error_message,
    }


@router.get("/{project_id}/logs")
def project_logs(project_id: int, lines: int = 50):
    """Последние логи бота"""
    logs = bot_manager.get_bot_logs(project_id, lines)
    return {"logs": logs}


@router.delete("/{project_id}")
def delete_project(project_id: int, db: Session = Depends(get_db)):
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")

    bot_manager.cleanup_bot(project_id)
    db.delete(project)
    db.commit()
    return {"status": "deleted"}
EOF
```

---

## Шаг 6. Webhook-роутер

Этот файл принимает события от Telegram и передаёт их боту:

```
cat > /opt/drivegpt/backend/routers/webhooks.py << 'EOF'
import httpx
from fastapi import APIRouter, Request, Depends
from sqlalchemy.orm import Session
from models import get_db, Project, BotUser, Message
from datetime import datetime, timezone

router = APIRouter(tags=["webhooks"])


@router.post("/webhook/{project_id}")
async def telegram_webhook(project_id: int, request: Request, db: Session = Depends(get_db)):
    """Принимает входящие события от Telegram"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        return {"ok": False, "error": "project not found"}

    # Получить тело запроса
    body = await request.json()

    # Извлекаем данные из Telegram update
    message = body.get("message", {})
    if not message:
        # Callback query или другой тип — пока пропускаем, передаём боту
        pass
    else:
        # Сохраняем пользователя и сообщение в базу
        from_user = message.get("from", {})
        tg_id = str(from_user.get("id", ""))
        text = message.get("text", "")
        chat_id = str(message.get("chat", {}).get("id", ""))

        # Найти или создать пользователя
        bot_user = db.query(BotUser).filter(
            BotUser.project_id == project_id,
            BotUser.telegram_id == tg_id
        ).first()

        if not bot_user:
            bot_user = BotUser(
                project_id=project_id,
                telegram_id=tg_id,
                username=from_user.get("username"),
                first_name=from_user.get("first_name"),
                last_name=from_user.get("last_name"),
            )
            db.add(bot_user)
            db.commit()
            db.refresh(bot_user)

        # Сохранить сообщение
        msg = Message(
            project_id=project_id,
            bot_user_id=bot_user.id,
            direction="in",
            text=text,
            sent_by="user",
            telegram_message_id=str(message.get("message_id", "")),
        )
        db.add(msg)
        db.commit()

    # Передать запрос боту (на его внутренний порт)
    internal_port = 9001 + project_id
    try:
        async with httpx.AsyncClient(timeout=10) as client:
            resp = await client.post(
                f"http://127.0.0.1:{internal_port}/webhook",
                json=body,
                headers={"Content-Type": "application/json"},
            )
            return resp.json()
    except httpx.ConnectError:
        # Бот не запущен — обрабатываем сами
        return {"ok": True, "note": "bot offline, message saved"}
    except Exception as e:
        return {"ok": True, "note": f"forwarding error: {str(e)}, message saved"}
EOF
```

---

## Шаг 7. Роутер для диалогов

```
cat > /opt/drivegpt/backend/routers/dialogs.py << 'EOF'
from fastapi import APIRouter, Depends, HTTPException
from sqlalchemy.orm import Session
from models import get_db, Project, BotUser, Message
from services import telegram_api

router = APIRouter(prefix="/api/projects/{project_id}", tags=["dialogs"])


@router.get("/dialogs")
def list_dialogs(project_id: int, db: Session = Depends(get_db)):
    users = db.query(BotUser).filter(BotUser.project_id == project_id).all()
    dialogs = []
    for user in users:
        last_msg = (
            db.query(Message)
            .filter(Message.bot_user_id == user.id)
            .order_by(Message.created_at.desc())
            .first()
        )
        msg_count = db.query(Message).filter(Message.bot_user_id == user.id).count()
        dialogs.append({
            "user_id": user.id,
            "telegram_id": user.telegram_id,
            "username": user.username,
            "first_name": user.first_name,
            "last_name": user.last_name,
            "message_count": msg_count,
            "last_message": last_msg.text if last_msg else None,
            "last_message_at": str(last_msg.created_at) if last_msg else None,
        })
    # Сортируем по последнему сообщению
    dialogs.sort(key=lambda d: d["last_message_at"] or "", reverse=True)
    return dialogs


@router.get("/dialogs/{user_id}")
def get_dialog(project_id: int, user_id: int, db: Session = Depends(get_db)):
    messages = (
        db.query(Message)
        .filter(Message.project_id == project_id, Message.bot_user_id == user_id)
        .order_by(Message.created_at.asc())
        .all()
    )
    user = db.query(BotUser).filter(BotUser.id == user_id).first()
    return {
        "user": {
            "id": user.id if user else None,
            "username": user.username if user else None,
            "first_name": user.first_name if user else None,
            "telegram_id": user.telegram_id if user else None,
        },
        "messages": [
            {
                "id": m.id,
                "direction": m.direction,
                "text": m.text,
                "media_url": m.media_url,
                "sent_by": m.sent_by,
                "created_at": str(m.created_at),
            }
            for m in messages
        ]
    }


@router.post("/dialogs/{user_id}/send")
async def send_operator_message(
    project_id: int, user_id: int, text: str, db: Session = Depends(get_db)
):
    """Отправить сообщение от имени бота (оператором)"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project or not project.telegram_token:
        raise HTTPException(status_code=400, detail="Токен бота не указан")

    user = db.query(BotUser).filter(BotUser.id == user_id).first()
    if not user or not user.telegram_id:
        raise HTTPException(status_code=404, detail="Пользователь не найден")

    # Отправить через Telegram API
    result = await telegram_api.send_message(
        project.telegram_token, user.telegram_id, text
    )

    if not result.get("ok"):
        raise HTTPException(status_code=500, detail=f"Ошибка отправки: {result}")

    # Сохранить в базу
    msg = Message(
        project_id=project_id,
        bot_user_id=user_id,
        direction="out",
        text=text,
        sent_by="operator",
    )
    db.add(msg)
    db.commit()

    return {"status": "sent", "message_id": result.get("result", {}).get("message_id")}
EOF
```

---

## Шаг 8. Роутер для пользователей ботов + CSV

```
cat > /opt/drivegpt/backend/routers/bot_users.py << 'EOF'
import csv
import io
from fastapi import APIRouter, Depends, HTTPException, UploadFile, File
from fastapi.responses import StreamingResponse
from sqlalchemy.orm import Session
from models import get_db, BotUser, Request

router = APIRouter(prefix="/api/projects/{project_id}", tags=["bot_users"])


@router.get("/users")
def list_users(project_id: int, db: Session = Depends(get_db)):
    users = db.query(BotUser).filter(BotUser.project_id == project_id).all()
    return [
        {
            "id": u.id, "telegram_id": u.telegram_id,
            "username": u.username, "first_name": u.first_name,
            "last_name": u.last_name, "email": u.email, "phone": u.phone,
            "generations_count": u.generations_count,
            "generations_balance": u.generations_balance,
            "first_purchase": u.first_purchase,
            "created_at": str(u.created_at),
        }
        for u in users
    ]


@router.get("/users/{user_id}")
def get_user(project_id: int, user_id: int, db: Session = Depends(get_db)):
    user = db.query(BotUser).filter(BotUser.id == user_id, BotUser.project_id == project_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="Пользователь не найден")
    return {
        "id": user.id, "telegram_id": user.telegram_id,
        "username": user.username, "first_name": user.first_name,
        "last_name": user.last_name, "email": user.email, "phone": user.phone,
        "generations_count": user.generations_count,
        "generations_balance": user.generations_balance,
        "first_purchase": user.first_purchase,
        "messenger_type": user.messenger_type,
        "created_at": str(user.created_at),
    }


@router.put("/users/{user_id}")
def update_user(
    project_id: int, user_id: int,
    email: str = None, phone: str = None,
    generations_balance: float = None,
    db: Session = Depends(get_db),
):
    user = db.query(BotUser).filter(BotUser.id == user_id, BotUser.project_id == project_id).first()
    if not user:
        raise HTTPException(status_code=404, detail="Пользователь не найден")
    if email is not None:
        user.email = email
    if phone is not None:
        user.phone = phone
    if generations_balance is not None:
        user.generations_balance = generations_balance
    db.commit()
    return {"status": "updated"}


@router.get("/users-export")
def export_csv(project_id: int, db: Session = Depends(get_db)):
    """Экспорт пользователей в CSV"""
    users = db.query(BotUser).filter(BotUser.project_id == project_id).all()

    output = io.StringIO()
    writer = csv.writer(output)
    writer.writerow(["ID", "Telegram ID", "Username", "Имя", "Фамилия", "Email", "Телефон", "Генерации", "Баланс", "Первая покупка", "Дата"])

    for u in users:
        writer.writerow([
            u.id, u.telegram_id, u.username, u.first_name, u.last_name,
            u.email, u.phone, u.generations_count, u.generations_balance,
            u.first_purchase, str(u.created_at),
        ])

    output.seek(0)
    return StreamingResponse(
        iter([output.getvalue()]),
        media_type="text/csv",
        headers={"Content-Disposition": "attachment; filename=users.csv"},
    )


@router.post("/users-import")
async def import_csv(project_id: int, file: UploadFile = File(...), db: Session = Depends(get_db)):
    """Импорт пользователей из CSV"""
    content = await file.read()
    text = content.decode("utf-8-sig")
    reader = csv.DictReader(io.StringIO(text))

    imported = 0
    for row in reader:
        user = BotUser(
            project_id=project_id,
            telegram_id=row.get("Telegram ID", row.get("telegram_id", "")),
            username=row.get("Username", row.get("username", "")),
            first_name=row.get("Имя", row.get("first_name", "")),
            last_name=row.get("Фамилия", row.get("last_name", "")),
            email=row.get("Email", row.get("email", "")),
            phone=row.get("Телефон", row.get("phone", "")),
        )
        db.add(user)
        imported += 1

    db.commit()
    return {"status": "ok", "imported": imported}


@router.get("/requests")
def list_requests(project_id: int, page: int = 1, per_page: int = 25, db: Session = Depends(get_db)):
    query = db.query(Request).filter(Request.project_id == project_id)
    total = query.count()
    requests = query.order_by(Request.created_at.desc()).offset((page - 1) * per_page).limit(per_page).all()
    items = []
    for r in requests:
        user = db.query(BotUser).filter(BotUser.id == r.bot_user_id).first()
        items.append({
            "id": r.id,
            "user_name": f"{user.first_name or ''} {user.last_name or ''}".strip() if user else "?",
            "username": user.username if user else None,
            "car_photo_url": r.car_photo_url,
            "wheel_photo_url": r.wheel_photo_url,
            "status": r.status,
            "created_at": str(r.created_at),
        })
    return {"items": items, "total": total, "page": page, "pages": (total + per_page - 1) // per_page}
EOF
```

---

## Шаг 9. Внутренний API для кода ботов

API, которое Python-код бота вызывает для записи данных:

```
cat > /opt/drivegpt/backend/routers/internal.py << 'EOF'
import os
import uuid
from datetime import datetime, timezone, timedelta
from fastapi import APIRouter, Depends, UploadFile, File
from sqlalchemy.orm import Session
from models import get_db, BotUser, Message, Request, UploadedFile
from config import UPLOAD_DIR, TEMP_FILE_DAYS

router = APIRouter(prefix="/internal", tags=["internal"])


@router.post("/users")
def create_or_update_user(
    project_id: int, telegram_id: str,
    username: str = None, first_name: str = None, last_name: str = None,
    db: Session = Depends(get_db),
):
    """Бот создаёт/обновляет пользователя при /start"""
    user = db.query(BotUser).filter(
        BotUser.project_id == project_id, BotUser.telegram_id == telegram_id
    ).first()

    if user:
        if username:
            user.username = username
        if first_name:
            user.first_name = first_name
        if last_name:
            user.last_name = last_name
        db.commit()
        return {"id": user.id, "action": "updated"}
    else:
        user = BotUser(
            project_id=project_id, telegram_id=telegram_id,
            username=username, first_name=first_name, last_name=last_name,
        )
        db.add(user)
        db.commit()
        db.refresh(user)
        return {"id": user.id, "action": "created"}


@router.put("/users/{telegram_id}")
def update_user_data(
    telegram_id: str, project_id: int,
    generations_count: int = None, generations_balance: float = None,
    first_purchase: bool = None, email: str = None, phone: str = None,
    db: Session = Depends(get_db),
):
    """Бот обновляет данные пользователя"""
    user = db.query(BotUser).filter(
        BotUser.project_id == project_id, BotUser.telegram_id == telegram_id
    ).first()
    if not user:
        return {"error": "user not found"}

    if generations_count is not None:
        user.generations_count = generations_count
    if generations_balance is not None:
        user.generations_balance = generations_balance
    if first_purchase is not None:
        user.first_purchase = first_purchase
    if email is not None:
        user.email = email
    if phone is not None:
        user.phone = phone
    db.commit()
    return {"status": "updated"}


@router.post("/messages")
def save_message(
    project_id: int, telegram_id: str, direction: str, text: str = None,
    sent_by: str = "bot", media_url: str = None,
    db: Session = Depends(get_db),
):
    """Бот сохраняет сообщение в диалог"""
    user = db.query(BotUser).filter(
        BotUser.project_id == project_id, BotUser.telegram_id == telegram_id
    ).first()
    if not user:
        return {"error": "user not found"}

    msg = Message(
        project_id=project_id, bot_user_id=user.id,
        direction=direction, text=text, sent_by=sent_by, media_url=media_url,
    )
    db.add(msg)
    db.commit()
    return {"id": msg.id}


@router.post("/requests")
def create_request(
    project_id: int, telegram_id: str,
    car_photo_url: str = None, wheel_photo_url: str = None,
    db: Session = Depends(get_db),
):
    """Бот создаёт заявку"""
    user = db.query(BotUser).filter(
        BotUser.project_id == project_id, BotUser.telegram_id == telegram_id
    ).first()
    if not user:
        return {"error": "user not found"}

    req = Request(
        project_id=project_id, bot_user_id=user.id,
        car_photo_url=car_photo_url, wheel_photo_url=wheel_photo_url,
    )
    db.add(req)

    # Увеличить счётчик генераций
    user.generations_count += 1
    db.commit()
    return {"id": req.id}


@router.post("/upload")
async def upload_file(project_id: int, file: UploadFile = File(...), db: Session = Depends(get_db)):
    """Бот загружает фото (временное хранение)"""
    ext = os.path.splitext(file.filename)[1] if file.filename else ".jpg"
    unique_name = f"{uuid.uuid4().hex}{ext}"
    file_path = os.path.join(UPLOAD_DIR, unique_name)

    with open(file_path, "wb") as f:
        content = await file.read()
        f.write(content)

    uploaded = UploadedFile(
        project_id=project_id,
        original_name=file.filename,
        file_path=file_path,
        file_type="image" if ext in [".jpg", ".jpeg", ".png", ".webp"] else "document",
        file_size=len(content),
        expires_at=datetime.now(timezone.utc) + timedelta(days=TEMP_FILE_DAYS),
    )
    db.add(uploaded)
    db.commit()

    public_url = f"https://drivegpt.pro/files/{unique_name}"
    return {"url": public_url, "expires_in_days": TEMP_FILE_DAYS}
EOF
```

---

## Шаг 10. Создать __init__.py для пакетов

```
touch /opt/drivegpt/backend/routers/__init__.py
touch /opt/drivegpt/backend/services/__init__.py
```

---

## Шаг 11. Обновить main.py

```
cat > /opt/drivegpt/backend/main.py << 'EOF'
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from models import init_db
from routers import projects, webhooks, dialogs, bot_users, internal

app = FastAPI(title="DriveGPT Bot Hosting", version="2.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

@app.on_event("startup")
def startup():
    init_db()

@app.get("/")
def root():
    return {"status": "ok", "service": "DriveGPT", "version": "2.0.0"}

@app.get("/health")
def health():
    return {"healthy": True}

# Подключаем роутеры
app.include_router(projects.router)
app.include_router(webhooks.router)
app.include_router(dialogs.router)
app.include_router(bot_users.router)
app.include_router(internal.router)
EOF
```

---

## Шаг 12. Перезапустить и проверить

```
systemctl restart drivegpt && sleep 2 && systemctl status drivegpt --no-pager
```

Если `active (running)` — откройте в браузере:
```
https://drivegpt.pro/docs
```

Теперь там должно быть **5 групп эндпоинтов**: projects, webhooks, dialogs, bot_users, internal.

---

## ✅ Итоги дня 3

| Что создано | Описание |
|------------|----------|
| `services/telegram_api.py` | Общение с Telegram API |
| `services/bot_manager.py` | Запуск/остановка ботов через systemd |
| `routers/projects.py` | Управление проектами + загрузка кода |
| `routers/webhooks.py` | Приём событий от Telegram |
| `routers/dialogs.py` | Диалоги + ответ оператора |
| `routers/bot_users.py` | Пользователи + CSV |
| `routers/internal.py` | API для кода ботов |

---

## ➡️ День 4: Фронтенд (Lovable)

Завтра начинаем делать интерфейс личного кабинета в Lovable!
