# 📋 День 2 — Домен, HTTPS и начало бэкенда
## Пошаговая инструкция

> **Время:** ~1-1.5 часа
> **Что понадобится:** Терминал + доступ к DNS для домена drivegpt.pro
> **Сервер:** 185.178.47.148

---

## ⚠️ Перед началом: доступ к домену

Для этого дня нужен **доступ к DNS-записям** домена `drivegpt.pro`.

**Что это значит:** нужно зайти в панель, где покупали домен, и добавить записи (указать, что домен ведёт на ваш сервер).

> Если домен покупали на **Reg.ru** — зайдите на reg.ru → Домены → drivegpt.pro → DNS
> Если на **Timeweb** — зайдите в Timeweb → Домены → drivegpt.pro → DNS / Ресурсные записи

**Если доступа к домену нет** — можно пока пропустить шаги 1-3 и начать сразу с Шага 4 (бэкенд). Домен настроим позже.

---

## Шаг 1. Подключиться к серверу

Откройте Терминал на Mac и введите:
```
ssh root@185.178.47.148
```
Введите пароль.

### Проверить, что вчерашнее работает:
```
systemctl status drivegpt --no-pager
```

Должно показать `active (running)` — значит бэкенд работает после вчерашней настройки ✅

> Флаг `--no-pager` нужен, чтобы не попасть в тот экран с `(END)`, откуда надо нажимать `q`.

---

## Шаг 2. Настроить DNS для домена

Зайдите в панель управления доменом `drivegpt.pro` (Reg.ru, Timeweb, или где он зарегистрирован).

Найдите раздел **«DNS-записи»** или **«Управление зоной»** и добавьте две записи:

| Тип записи | Имя (Host) | Значение (Value) | TTL |
|:----------:|:----------:|:-----------------:|:---:|
| **A** | @ | 185.178.47.148 | 3600 |
| **A** | api | 185.178.47.148 | 3600 |

**Пояснение простыми словами:**
- Первая запись: `drivegpt.pro` → ведёт на ваш сервер
- Вторая запись: `api.drivegpt.pro` → тоже ведёт на ваш сервер

**Где что вписать:**
- **Тип** — выбрать из списка **A**
- **Имя / Host** — ввести **@** (для основного) или **api** (для поддомена)
- **Значение / IP / Value** — ввести **185.178.47.148**
- **TTL** — оставить по умолчанию (обычно 3600)

> После добавления записей нужно **подождать 5-30 минут**, пока DNS обновится.

### Проверить, что DNS заработал:

На сервере выполните:
```
ping -c 2 drivegpt.pro
```

Если в ответе видите `185.178.47.148` — DNS настроен ✅

Если показывает другой IP или ошибку — DNS ещё не обновился, подождите и повторите.

---

## Шаг 3. Настроить Nginx + HTTPS

### 3.1 — Создать конфиг Nginx

```
cat > /etc/nginx/sites-available/drivegpt << 'EOF'
server {
    listen 80;
    server_name drivegpt.pro api.drivegpt.pro;

    # Основной сайт (фронтенд) — пока заглушка
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Загрузка файлов — увеличиваем лимит до 50 МБ
    client_max_body_size 50M;

    # Статические файлы (фото пользователей)
    location /files/ {
        alias /opt/drivegpt/uploads/;
        expires 7d;
    }
}
EOF
```

### 3.2 — Активировать конфиг

```
ln -sf /etc/nginx/sites-available/drivegpt /etc/nginx/sites-enabled/drivegpt
```

### 3.3 — Проверить конфиг на ошибки

```
nginx -t
```

Должно показать:
```
nginx: configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Если видите **test is successful** — всё ок ✅

### 3.4 — Перезапустить Nginx

```
systemctl reload nginx
```

### 3.5 — Проверить в браузере

Откройте в браузере:
```
http://drivegpt.pro
```

Если видите `{"status":"ok","service":"DriveGPT","message":"Сервер работает!"}` — Nginx работает ✅

### 3.6 — Получить SSL-сертификат (HTTPS)

```
certbot --nginx -d drivegpt.pro -d api.drivegpt.pro --non-interactive --agree-tos -m your@email.com
```

> ⚠️ Замените `your@email.com` на **ваш реальный email**. На него придут уведомления о продлении сертификата.

Если всё прошло успешно, увидите:
```
Successfully received certificate.
```

### 3.7 — Проверить HTTPS

Откройте в браузере:
```
https://drivegpt.pro
```

Должен быть **зелёный замочек** 🔒 и тот же JSON-ответ.

### 3.8 — Настроить автопродление сертификата

```
systemctl enable certbot.timer
systemctl start certbot.timer
```

Теперь сертификат будет автоматически продлеваться — больше не нужно об этом думать.

**Результат шагов 1-3:**
- ✅ Домен drivegpt.pro ведёт на сервер
- ✅ api.drivegpt.pro тоже работает
- ✅ HTTPS с зелёным замочком
- ✅ Сертификат продлевается автоматически

---

## Шаг 4. Создать базу данных

Теперь начинаем писать реальный бэкенд! Первое — создадим структуру базы данных.

### 4.1 — Создать файл настроек

```
cat > /opt/drivegpt/backend/config.py << 'EOF'
import os

# Путь к базе данных SQLite
DATABASE_URL = "sqlite:///opt/drivegpt/data/drivegpt.db"

# Секретный ключ для JWT-токенов (авторизация)
SECRET_KEY = os.environ.get("SECRET_KEY", "drivegpt-secret-key-change-me-2026")

# Пароль для входа в ЛК
ADMIN_EMAIL = "admin@drivegpt.pro"
ADMIN_PASSWORD = "DriveGPT2026!"

# Настройки загрузки файлов
UPLOAD_DIR = "/opt/drivegpt/uploads"
BOT_PROJECTS_DIR = "/opt/drivegpt/bot-projects"
MAX_UPLOAD_SIZE = 50 * 1024 * 1024  # 50 МБ

# Сколько дней хранить временные фото
TEMP_FILE_DAYS = 7
EOF
```

> ❗ Потом **обязательно поменяйте** пароль `DriveGPT2026!` на свой!

### 4.2 — Создать модели базы данных

```
cat > /opt/drivegpt/backend/models.py << 'EOF'
from sqlalchemy import create_engine, Column, Integer, String, Boolean, DateTime, Text, Float, ForeignKey
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, relationship
from datetime import datetime, timezone

from config import DATABASE_URL

engine = create_engine(DATABASE_URL, connect_args={"check_same_thread": False})
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()


def get_db():
    """Получить сессию базы данных"""
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()


class Project(Base):
    """Проект = один Telegram-бот"""
    __tablename__ = "projects"

    id = Column(Integer, primary_key=True, index=True)
    name = Column(String(255), nullable=False)
    telegram_token = Column(String(255), nullable=True)
    bot_username = Column(String(255), nullable=True)
    bot_name = Column(String(255), nullable=True)
    status = Column(String(50), default="stopped")  # stopped, running, error
    webhook_url = Column(String(500), nullable=True)
    messenger_type = Column(String(50), default="telegram")  # telegram, max (для будущего)
    error_message = Column(Text, nullable=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    updated_at = Column(DateTime, default=lambda: datetime.now(timezone.utc),
                       onupdate=lambda: datetime.now(timezone.utc))

    # Связи
    users = relationship("BotUser", back_populates="project")
    messages = relationship("Message", back_populates="project")
    requests = relationship("Request", back_populates="project")


class BotUser(Base):
    """Пользователь Telegram-бота"""
    __tablename__ = "bot_users"

    id = Column(Integer, primary_key=True, index=True)
    project_id = Column(Integer, ForeignKey("projects.id"), nullable=False)
    telegram_id = Column(String(50), nullable=True)
    username = Column(String(255), nullable=True)
    first_name = Column(String(255), nullable=True)
    last_name = Column(String(255), nullable=True)
    email = Column(String(255), nullable=True)
    phone = Column(String(50), nullable=True)
    generations_count = Column(Integer, default=0)
    generations_balance = Column(Float, default=0)
    first_purchase = Column(Boolean, default=False)
    messenger_type = Column(String(50), default="telegram")
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))
    updated_at = Column(DateTime, default=lambda: datetime.now(timezone.utc),
                       onupdate=lambda: datetime.now(timezone.utc))

    # Связи
    project = relationship("Project", back_populates="users")
    messages = relationship("Message", back_populates="bot_user")
    requests = relationship("Request", back_populates="bot_user")


class Message(Base):
    """Сообщение в диалоге"""
    __tablename__ = "messages"

    id = Column(Integer, primary_key=True, index=True)
    project_id = Column(Integer, ForeignKey("projects.id"), nullable=False)
    bot_user_id = Column(Integer, ForeignKey("bot_users.id"), nullable=False)
    direction = Column(String(10), nullable=False)  # in (от пользователя), out (от бота)
    text = Column(Text, nullable=True)
    media_url = Column(String(500), nullable=True)
    media_type = Column(String(50), nullable=True)  # photo, document, voice
    sent_by = Column(String(20), default="bot")  # bot, operator, user
    telegram_message_id = Column(String(50), nullable=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))

    # Связи
    project = relationship("Project", back_populates="messages")
    bot_user = relationship("BotUser", back_populates="messages")


class Request(Base):
    """Заявка (генерация) — фото авто + фото диска"""
    __tablename__ = "requests"

    id = Column(Integer, primary_key=True, index=True)
    project_id = Column(Integer, ForeignKey("projects.id"), nullable=False)
    bot_user_id = Column(Integer, ForeignKey("bot_users.id"), nullable=False)
    car_photo_url = Column(String(500), nullable=True)
    wheel_photo_url = Column(String(500), nullable=True)
    status = Column(String(50), default="pending")  # pending, processing, completed
    result_url = Column(String(500), nullable=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))

    # Связи
    project = relationship("Project", back_populates="requests")
    bot_user = relationship("BotUser", back_populates="requests")


class UploadedFile(Base):
    """Временный загруженный файл"""
    __tablename__ = "uploaded_files"

    id = Column(Integer, primary_key=True, index=True)
    project_id = Column(Integer, ForeignKey("projects.id"), nullable=True)
    original_name = Column(String(255), nullable=True)
    file_path = Column(String(500), nullable=False)
    file_type = Column(String(50), nullable=True)  # image, document
    file_size = Column(Integer, nullable=True)
    expires_at = Column(DateTime, nullable=True)
    created_at = Column(DateTime, default=lambda: datetime.now(timezone.utc))


# Создать все таблицы в базе данных
def init_db():
    Base.metadata.create_all(bind=engine)
    print("База данных создана!")
EOF
```

### 4.3 — Создать базу данных

```
cd /opt/drivegpt/backend && source venv/bin/activate
python -c "from models import init_db; init_db()"
```

Должно показать: **«База данных создана!»** ✅

### 4.4 — Проверить, что файл базы появился

```
ls -la /opt/drivegpt/data/
```

Должен быть файл `drivegpt.db` — это ваша база данных.

---

## Шаг 5. Обновить бэкенд (реальный main.py)

Теперь заменим тестовый `main.py` на полноценный:

```
cat > /opt/drivegpt/backend/main.py << 'EOF'
from fastapi import FastAPI, Depends, HTTPException
from fastapi.middleware.cors import CORSMiddleware
from sqlalchemy.orm import Session
from models import get_db, init_db, Project, BotUser, Message, Request
from config import ADMIN_EMAIL, ADMIN_PASSWORD

# Инициализация
app = FastAPI(title="DriveGPT Bot Hosting", version="1.0.0")

# Разрешаем запросы с фронтенда (CORS)
app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Создать таблицы при запуске
@app.on_event("startup")
def startup():
    init_db()


# ==================== ОБЩИЕ ====================

@app.get("/")
def root():
    return {"status": "ok", "service": "DriveGPT", "version": "1.0.0"}

@app.get("/health")
def health():
    return {"healthy": True}


# ==================== ПРОЕКТЫ ====================

@app.get("/api/projects")
def list_projects(db: Session = Depends(get_db)):
    """Список всех проектов"""
    projects = db.query(Project).all()
    return [
        {
            "id": p.id,
            "name": p.name,
            "status": p.status,
            "bot_username": p.bot_username,
            "messenger_type": p.messenger_type,
            "created_at": str(p.created_at),
        }
        for p in projects
    ]

@app.post("/api/projects")
def create_project(name: str, db: Session = Depends(get_db)):
    """Создать новый проект"""
    project = Project(name=name)
    db.add(project)
    db.commit()
    db.refresh(project)
    return {"id": project.id, "name": project.name, "status": project.status}

@app.get("/api/projects/{project_id}")
def get_project(project_id: int, db: Session = Depends(get_db)):
    """Получить проект по ID"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")
    return {
        "id": project.id,
        "name": project.name,
        "telegram_token": project.telegram_token,
        "bot_username": project.bot_username,
        "bot_name": project.bot_name,
        "status": project.status,
        "webhook_url": project.webhook_url,
        "messenger_type": project.messenger_type,
        "error_message": project.error_message,
        "created_at": str(project.created_at),
    }

@app.put("/api/projects/{project_id}")
def update_project(
    project_id: int,
    name: str = None,
    telegram_token: str = None,
    db: Session = Depends(get_db)
):
    """Обновить проект"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")
    if name:
        project.name = name
    if telegram_token:
        project.telegram_token = telegram_token
    db.commit()
    return {"id": project.id, "name": project.name, "status": "updated"}

@app.delete("/api/projects/{project_id}")
def delete_project(project_id: int, db: Session = Depends(get_db)):
    """Удалить проект"""
    project = db.query(Project).filter(Project.id == project_id).first()
    if not project:
        raise HTTPException(status_code=404, detail="Проект не найден")
    db.delete(project)
    db.commit()
    return {"status": "deleted"}


# ==================== ПОЛЬЗОВАТЕЛИ БОТОВ ====================

@app.get("/api/projects/{project_id}/users")
def list_bot_users(project_id: int, db: Session = Depends(get_db)):
    """Список пользователей бота"""
    users = db.query(BotUser).filter(BotUser.project_id == project_id).all()
    return [
        {
            "id": u.id,
            "telegram_id": u.telegram_id,
            "username": u.username,
            "first_name": u.first_name,
            "last_name": u.last_name,
            "email": u.email,
            "generations_count": u.generations_count,
            "generations_balance": u.generations_balance,
            "first_purchase": u.first_purchase,
            "created_at": str(u.created_at),
        }
        for u in users
    ]

@app.get("/api/projects/{project_id}/users/{user_id}")
def get_bot_user(project_id: int, user_id: int, db: Session = Depends(get_db)):
    """Карточка пользователя"""
    user = db.query(BotUser).filter(
        BotUser.id == user_id, BotUser.project_id == project_id
    ).first()
    if not user:
        raise HTTPException(status_code=404, detail="Пользователь не найден")
    return {
        "id": user.id,
        "telegram_id": user.telegram_id,
        "username": user.username,
        "first_name": user.first_name,
        "last_name": user.last_name,
        "email": user.email,
        "phone": user.phone,
        "generations_count": user.generations_count,
        "generations_balance": user.generations_balance,
        "first_purchase": user.first_purchase,
        "messenger_type": user.messenger_type,
        "created_at": str(user.created_at),
    }


# ==================== ДИАЛОГИ ====================

@app.get("/api/projects/{project_id}/dialogs")
def list_dialogs(project_id: int, db: Session = Depends(get_db)):
    """Список диалогов (пользователи с последним сообщением)"""
    users = db.query(BotUser).filter(BotUser.project_id == project_id).all()
    dialogs = []
    for user in users:
        last_msg = (
            db.query(Message)
            .filter(Message.bot_user_id == user.id)
            .order_by(Message.created_at.desc())
            .first()
        )
        dialogs.append({
            "user_id": user.id,
            "username": user.username,
            "first_name": user.first_name,
            "last_message": last_msg.text if last_msg else None,
            "last_message_at": str(last_msg.created_at) if last_msg else None,
        })
    return dialogs

@app.get("/api/projects/{project_id}/dialogs/{user_id}")
def get_dialog(project_id: int, user_id: int, db: Session = Depends(get_db)):
    """История сообщений с пользователем"""
    messages = (
        db.query(Message)
        .filter(Message.project_id == project_id, Message.bot_user_id == user_id)
        .order_by(Message.created_at.asc())
        .all()
    )
    return [
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


# ==================== ЛОГИ ЗАЯВОК ====================

@app.get("/api/projects/{project_id}/requests")
def list_requests(
    project_id: int,
    page: int = 1,
    per_page: int = 25,
    date_from: str = None,
    date_to: str = None,
    db: Session = Depends(get_db)
):
    """Список заявок с пагинацией и фильтром по датам"""
    query = db.query(Request).filter(Request.project_id == project_id)

    if date_from:
        from datetime import datetime
        try:
            dt_from = datetime.fromisoformat(date_from)
            query = query.filter(Request.created_at >= dt_from)
        except ValueError:
            pass

    if date_to:
        from datetime import datetime
        try:
            dt_to = datetime.fromisoformat(date_to)
            query = query.filter(Request.created_at <= dt_to)
        except ValueError:
            pass

    total = query.count()
    requests = (
        query.order_by(Request.created_at.desc())
        .offset((page - 1) * per_page)
        .limit(per_page)
        .all()
    )

    items = []
    for r in requests:
        user = db.query(BotUser).filter(BotUser.id == r.bot_user_id).first()
        items.append({
            "id": r.id,
            "user_name": f"{user.first_name or ''} {user.last_name or ''}".strip() if user else "Неизвестный",
            "username": user.username if user else None,
            "generations_count": user.generations_count if user else 0,
            "car_photo_url": r.car_photo_url,
            "wheel_photo_url": r.wheel_photo_url,
            "status": r.status,
            "user_id": r.bot_user_id,
            "project_id": r.project_id,
            "created_at": str(r.created_at),
        })

    return {
        "items": items,
        "total": total,
        "page": page,
        "per_page": per_page,
        "pages": (total + per_page - 1) // per_page,
    }
EOF
```

### 5.1 — Перезапустить бэкенд

```
systemctl restart drivegpt
systemctl status drivegpt --no-pager
```

Должно показать `active (running)` ✅

---

## Шаг 6. Проверить API

### 6.1 — Проверить главную страницу

В браузере откройте:
```
http://185.178.47.148:8000
```

Должно показать версию `1.0.0`.

### 6.2 — Открыть документацию API

FastAPI автоматически создаёт красивую документацию! Откройте:
```
http://185.178.47.148:8000/docs
```

Вы увидите список всех API-эндпоинтов — можно прямо там тестировать.

### 6.3 — Создать тестовый проект через API

Прямо на странице `/docs`:
1. Найдите **POST /api/projects**
2. Нажмите **«Try it out»**
3. В поле `name` введите: **DriveGPT Bot**
4. Нажмите **«Execute»**
5. Должно вернуть: `{"id": 1, "name": "DriveGPT Bot", "status": "stopped"}`

### 6.4 — Проверить список проектов

На той же странице `/docs`:
1. Найдите **GET /api/projects**
2. Нажмите **«Try it out»** → **«Execute»**
3. Должен появиться ваш проект

**Результат шага 6:**
- ✅ API работает
- ✅ Документация доступна
- ✅ Можно создавать проекты

---

## ✅ Итоги дня 2

| Что сделано | Статус |
|-------------|:------:|
| DNS для drivegpt.pro | ✅ или ⏳ (если ждём доступ) |
| Nginx настроен | ✅ или ⏳ |
| HTTPS сертификат | ✅ или ⏳ |
| Файл настроек (config.py) | ✅ |
| Модели базы данных (5 таблиц) | ✅ |
| База данных создана (SQLite) | ✅ |
| API проектов (CRUD) | ✅ |
| API пользователей ботов | ✅ |
| API диалогов | ✅ |
| API логов заявок (с пагинацией) | ✅ |
| Документация API (/docs) | ✅ |

---

## 🆘 Если что-то пошло не так

### «nginx: [emerg] could not build server_names_hash»
```
nano /etc/nginx/nginx.conf
```
Найдите строку `server_names_hash_bucket_size` и раскомментируйте (уберите #). Или добавьте: `server_names_hash_bucket_size 64;` внутри блока `http {}`. Затем `nginx -t && systemctl reload nginx`.

### «Error: no such table: projects»
```
cd /opt/drivegpt/backend && source venv/bin/activate
python -c "from models import init_db; init_db()"
systemctl restart drivegpt
```

### Certbot не работает / ошибка DNS
DNS ещё не обновился. Подождите 30 минут и повторите. Или проверьте, что записи добавлены правильно.

### Сервер не отвечает после перезапуска
```
journalctl -u drivegpt -n 20 --no-pager
```
Скиньте вывод — разберёмся!

---

## ➡️ Следующий шаг (День 3)

- Webhook-роутер (приём событий от Telegram)
- Менеджер ботов (запуск/остановка)
- Загрузка кода проекта (ZIP)
- Импорт/экспорт CSV
