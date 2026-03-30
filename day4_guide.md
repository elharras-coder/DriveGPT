# 📋 День 4 — Фронтенд личного кабинета
## Пошаговая инструкция

> **Время:** ~1.5-2 часа
> **Что понадобится:** Терминал (подключение к серверу)
> **Результат:** Красивый ЛК на `https://app.drivegpt.pro`

---

## Шаг 1. Подключиться к серверу

```
ssh -o ServerAliveInterval=60 root@185.178.47.148
```

---

## Шаг 2. Создать папку для фронтенда

```
mkdir -p /opt/drivegpt/frontend
```

---

## Шаг 3. Обновить Nginx — разделить app и api

```
cat > /etc/nginx/sites-available/drivegpt << 'EOF'
# Главная страница (drivegpt.pro) — пока редирект на app
server {
    listen 80;
    server_name drivegpt.pro;
    return 301 https://app.drivegpt.pro$request_uri;
}

# Фронтенд (app.drivegpt.pro)
server {
    listen 80;
    server_name app.drivegpt.pro;

    root /opt/drivegpt/frontend;
    index index.html;

    location / {
        try_files $uri $uri/ /index.html;
    }

    # API проксирование
    location /api/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /webhook/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /internal/ {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }

    location /docs {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
    }

    location /openapi.json {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
    }

    location /files/ {
        alias /opt/drivegpt/uploads/;
        expires 7d;
    }

    client_max_body_size 50M;
}

# API (api.drivegpt.pro) — прямой доступ к API
server {
    listen 80;
    server_name api.drivegpt.pro;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    client_max_body_size 50M;
}
EOF
```

Применить:
```
nginx -t && systemctl reload nginx
```

Обновить SSL (добавить новую конфигурацию):
```
certbot --nginx -d drivegpt.pro -d api.drivegpt.pro -d app.drivegpt.pro --non-interactive --agree-tos -m hotel-go@yandex.ru
```

---

## Шаг 4. Создать CSS (дизайн-система)

```
cat > /opt/drivegpt/frontend/styles.css << 'CSSEOF'
@import url('https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap');

:root {
  --bg-primary: #0f1117;
  --bg-secondary: #1a1d27;
  --bg-card: #21242f;
  --bg-hover: #282c3a;
  --accent: #6c5ce7;
  --accent-light: #a29bfe;
  --accent-glow: rgba(108, 92, 231, 0.3);
  --success: #00b894;
  --danger: #ff6b6b;
  --warning: #ffd93d;
  --text-primary: #ffffff;
  --text-secondary: #a0a4b8;
  --text-muted: #6b7089;
  --border: #2d3142;
  --border-light: #363a4f;
  --radius: 12px;
  --radius-sm: 8px;
  --shadow: 0 4px 24px rgba(0, 0, 0, 0.3);
}

* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: 'Inter', -apple-system, sans-serif;
  background: var(--bg-primary);
  color: var(--text-primary);
  min-height: 100vh;
}

/* === LAYOUT === */
.app { display: flex; min-height: 100vh; }

.sidebar {
  width: 260px;
  background: var(--bg-secondary);
  border-right: 1px solid var(--border);
  padding: 24px 16px;
  display: flex;
  flex-direction: column;
  position: fixed;
  height: 100vh;
  z-index: 10;
}

.sidebar-logo {
  font-size: 20px;
  font-weight: 700;
  margin-bottom: 8px;
  display: flex;
  align-items: center;
  gap: 10px;
}

.sidebar-logo span { color: var(--accent); }

.sidebar-version {
  font-size: 11px;
  color: var(--text-muted);
  margin-bottom: 32px;
}

.sidebar-nav { flex: 1; }

.nav-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 12px 16px;
  border-radius: var(--radius-sm);
  color: var(--text-secondary);
  text-decoration: none;
  cursor: pointer;
  transition: all 0.2s;
  margin-bottom: 4px;
  font-size: 14px;
}

.nav-item:hover { background: var(--bg-hover); color: var(--text-primary); }
.nav-item.active { background: var(--accent); color: white; }

.nav-divider {
  height: 1px;
  background: var(--border);
  margin: 16px 0;
}

.main-content {
  flex: 1;
  margin-left: 260px;
  padding: 32px;
}

/* === TOP BAR === */
.topbar {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 32px;
}

.topbar h1 {
  font-size: 24px;
  font-weight: 700;
}

.topbar-actions { display: flex; gap: 12px; }

/* === BUTTONS === */
.btn {
  padding: 10px 20px;
  border-radius: var(--radius-sm);
  border: none;
  font-family: inherit;
  font-size: 14px;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.2s;
  display: inline-flex;
  align-items: center;
  gap: 8px;
}

.btn-primary {
  background: var(--accent);
  color: white;
}
.btn-primary:hover { background: var(--accent-light); box-shadow: 0 0 20px var(--accent-glow); }

.btn-secondary {
  background: var(--bg-card);
  color: var(--text-primary);
  border: 1px solid var(--border);
}
.btn-secondary:hover { border-color: var(--accent); }

.btn-danger { background: var(--danger); color: white; }
.btn-danger:hover { opacity: 0.8; }

.btn-sm { padding: 6px 14px; font-size: 13px; }

.btn-icon {
  width: 36px;
  height: 36px;
  padding: 0;
  display: flex;
  align-items: center;
  justify-content: center;
  border-radius: var(--radius-sm);
  background: var(--bg-card);
  border: 1px solid var(--border);
  color: var(--text-secondary);
  cursor: pointer;
  transition: all 0.2s;
}
.btn-icon:hover { border-color: var(--accent); color: var(--accent); }

/* === CARDS === */
.card {
  background: var(--bg-card);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  padding: 24px;
  transition: all 0.2s;
}

.card:hover { border-color: var(--border-light); }
.card-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 16px; }
.card-title { font-size: 16px; font-weight: 600; }

/* === GRID === */
.grid-2 { display: grid; grid-template-columns: repeat(2, 1fr); gap: 20px; }
.grid-3 { display: grid; grid-template-columns: repeat(3, 1fr); gap: 20px; }
.grid-4 { display: grid; grid-template-columns: repeat(4, 1fr); gap: 16px; }

/* === STAT CARD === */
.stat-card {
  background: var(--bg-card);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  padding: 20px;
}

.stat-card .stat-label { font-size: 13px; color: var(--text-muted); margin-bottom: 8px; }
.stat-card .stat-value { font-size: 28px; font-weight: 700; }
.stat-card .stat-change { font-size: 12px; color: var(--success); margin-top: 4px; }

/* === TABLE === */
.table-wrap {
  background: var(--bg-card);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  overflow: hidden;
}

table { width: 100%; border-collapse: collapse; }
th { text-align: left; padding: 14px 20px; font-size: 12px; text-transform: uppercase; letter-spacing: 0.5px; color: var(--text-muted); border-bottom: 1px solid var(--border); background: var(--bg-secondary); }
td { padding: 14px 20px; font-size: 14px; border-bottom: 1px solid var(--border); }
tr:last-child td { border-bottom: none; }
tr:hover td { background: var(--bg-hover); }

/* === STATUS BADGES === */
.badge {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 4px 12px;
  border-radius: 50px;
  font-size: 12px;
  font-weight: 500;
}

.badge-running { background: rgba(0, 184, 148, 0.15); color: var(--success); }
.badge-stopped { background: rgba(160, 164, 184, 0.15); color: var(--text-secondary); }
.badge-error { background: rgba(255, 107, 107, 0.15); color: var(--danger); }
.badge-pending { background: rgba(255, 217, 61, 0.15); color: var(--warning); }

.badge-dot {
  width: 6px;
  height: 6px;
  border-radius: 50%;
  background: currentColor;
}

/* === TABS === */
.tabs {
  display: flex;
  gap: 4px;
  border-bottom: 1px solid var(--border);
  margin-bottom: 24px;
}

.tab {
  padding: 12px 20px;
  cursor: pointer;
  font-size: 14px;
  color: var(--text-secondary);
  border-bottom: 2px solid transparent;
  transition: all 0.2s;
}

.tab:hover { color: var(--text-primary); }
.tab.active { color: var(--accent); border-bottom-color: var(--accent); }

/* === FORM === */
.form-group { margin-bottom: 20px; }
.form-label { display: block; font-size: 13px; color: var(--text-secondary); margin-bottom: 8px; }

.form-input {
  width: 100%;
  padding: 12px 16px;
  background: var(--bg-primary);
  border: 1px solid var(--border);
  border-radius: var(--radius-sm);
  color: var(--text-primary);
  font-family: inherit;
  font-size: 14px;
  outline: none;
  transition: border-color 0.2s;
}
.form-input:focus { border-color: var(--accent); }

.form-input::placeholder { color: var(--text-muted); }

/* === DROP ZONE === */
.dropzone {
  border: 2px dashed var(--border);
  border-radius: var(--radius);
  padding: 48px;
  text-align: center;
  cursor: pointer;
  transition: all 0.3s;
}

.dropzone:hover, .dropzone.dragover {
  border-color: var(--accent);
  background: rgba(108, 92, 231, 0.05);
}

.dropzone-icon { font-size: 48px; margin-bottom: 16px; }
.dropzone-text { color: var(--text-secondary); font-size: 14px; }
.dropzone-hint { color: var(--text-muted); font-size: 12px; margin-top: 8px; }

/* === DIALOG === */
.dialog-list { max-height: calc(100vh - 250px); overflow-y: auto; }

.dialog-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 14px 20px;
  border-bottom: 1px solid var(--border);
  cursor: pointer;
  transition: background 0.2s;
}
.dialog-item:hover { background: var(--bg-hover); }
.dialog-item.active { background: var(--accent); border-radius: var(--radius-sm); }

.dialog-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: var(--accent);
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  font-size: 16px;
  flex-shrink: 0;
}

.dialog-info { flex: 1; min-width: 0; }
.dialog-name { font-size: 14px; font-weight: 500; }
.dialog-preview { font-size: 13px; color: var(--text-muted); white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.dialog-time { font-size: 11px; color: var(--text-muted); }
.dialog-count { font-size: 11px; background: var(--accent); color: white; padding: 2px 8px; border-radius: 10px; }

/* === CHAT === */
.chat-container { display: flex; flex-direction: column; height: calc(100vh - 250px); }
.chat-messages { flex: 1; overflow-y: auto; padding: 20px; }

.chat-message {
  max-width: 70%;
  margin-bottom: 12px;
  padding: 12px 16px;
  border-radius: 16px;
  font-size: 14px;
  line-height: 1.5;
}

.chat-message.incoming {
  background: var(--bg-secondary);
  border-bottom-left-radius: 4px;
  align-self: flex-start;
}

.chat-message.outgoing {
  background: var(--accent);
  border-bottom-right-radius: 4px;
  align-self: flex-end;
  margin-left: auto;
}

.chat-message .msg-time { font-size: 11px; color: var(--text-muted); margin-top: 4px; }
.chat-message.outgoing .msg-time { color: rgba(255,255,255,0.6); }
.chat-message .msg-sender { font-size: 11px; color: var(--accent-light); margin-bottom: 4px; }

.chat-input-wrap {
  display: flex;
  gap: 12px;
  padding: 16px 20px;
  border-top: 1px solid var(--border);
}

.chat-input {
  flex: 1;
  padding: 12px 16px;
  background: var(--bg-primary);
  border: 1px solid var(--border);
  border-radius: 24px;
  color: var(--text-primary);
  font-family: inherit;
  font-size: 14px;
  outline: none;
}
.chat-input:focus { border-color: var(--accent); }

.chat-send {
  width: 44px;
  height: 44px;
  border-radius: 50%;
  background: var(--accent);
  border: none;
  color: white;
  cursor: pointer;
  font-size: 18px;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.2s;
}
.chat-send:hover { transform: scale(1.1); }

/* === MODAL === */
.modal-overlay {
  position: fixed;
  top: 0; left: 0; right: 0; bottom: 0;
  background: rgba(0,0,0,0.6);
  backdrop-filter: blur(4px);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 100;
  opacity: 0;
  visibility: hidden;
  transition: all 0.3s;
}
.modal-overlay.active { opacity: 1; visibility: visible; }

.modal {
  background: var(--bg-card);
  border-radius: var(--radius);
  border: 1px solid var(--border);
  padding: 32px;
  width: 100%;
  max-width: 500px;
  transform: translateY(20px);
  transition: transform 0.3s;
}
.modal-overlay.active .modal { transform: translateY(0); }

.modal-title { font-size: 20px; font-weight: 600; margin-bottom: 24px; }

/* === LOGIN === */
.login-page {
  min-height: 100vh;
  display: flex;
  align-items: center;
  justify-content: center;
  background: var(--bg-primary);
}

.login-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: var(--radius);
  padding: 48px;
  width: 100%;
  max-width: 420px;
  text-align: center;
}

.login-logo { font-size: 32px; font-weight: 700; margin-bottom: 8px; }
.login-logo span { color: var(--accent); }
.login-subtitle { color: var(--text-muted); font-size: 14px; margin-bottom: 40px; }

/* === PAGINATION === */
.pagination {
  display: flex;
  justify-content: center;
  gap: 8px;
  margin-top: 24px;
}

.page-btn {
  padding: 8px 14px;
  border-radius: var(--radius-sm);
  border: 1px solid var(--border);
  background: var(--bg-card);
  color: var(--text-secondary);
  cursor: pointer;
  font-size: 13px;
  transition: all 0.2s;
}
.page-btn:hover { border-color: var(--accent); }
.page-btn.active { background: var(--accent); color: white; border-color: var(--accent); }

/* === EMPTY STATE === */
.empty-state {
  text-align: center;
  padding: 60px 20px;
  color: var(--text-muted);
}
.empty-state-icon { font-size: 48px; margin-bottom: 16px; }
.empty-state-text { font-size: 16px; margin-bottom: 8px; }
.empty-state-hint { font-size: 14px; }

/* === TOAST === */
.toast {
  position: fixed;
  bottom: 24px;
  right: 24px;
  padding: 14px 24px;
  border-radius: var(--radius-sm);
  font-size: 14px;
  z-index: 200;
  transform: translateY(100px);
  opacity: 0;
  transition: all 0.3s;
}
.toast.show { transform: translateY(0); opacity: 1; }
.toast-success { background: var(--success); color: white; }
.toast-error { background: var(--danger); color: white; }

/* === LOADING === */
.spinner {
  width: 24px;
  height: 24px;
  border: 3px solid var(--border);
  border-top-color: var(--accent);
  border-radius: 50%;
  animation: spin 0.8s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }

/* === RESPONSIVE === */
@media (max-width: 768px) {
  .sidebar { display: none; }
  .main-content { margin-left: 0; padding: 16px; }
  .grid-2, .grid-3, .grid-4 { grid-template-columns: 1fr; }
}
CSSEOF
```

---

## Шаг 5. Создать главный HTML-файл

```
cat > /opt/drivegpt/frontend/index.html << 'HTMLEOF'
<!DOCTYPE html>
<html lang="ru">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>DriveGPT — Личный кабинет</title>
  <link rel="stylesheet" href="/styles.css">
</head>
<body>

<!-- ============ LOGIN ============ -->
<div id="loginPage" class="login-page">
  <div class="login-card">
    <div class="login-logo">Drive<span>GPT</span></div>
    <div class="login-subtitle">Управление Telegram-ботами</div>
    <div class="form-group">
      <input type="email" class="form-input" id="loginEmail" placeholder="Email" value="admin@drivegpt.pro">
    </div>
    <div class="form-group">
      <input type="password" class="form-input" id="loginPassword" placeholder="Пароль">
    </div>
    <button class="btn btn-primary" style="width:100%;justify-content:center;padding:14px" onclick="doLogin()">Войти</button>
    <div id="loginError" style="color:var(--danger);font-size:13px;margin-top:16px;display:none"></div>
  </div>
</div>

<!-- ============ MAIN APP ============ -->
<div id="appPage" class="app" style="display:none">

  <!-- SIDEBAR -->
  <aside class="sidebar">
    <div class="sidebar-logo">Drive<span>GPT</span></div>
    <div class="sidebar-version">Bot Hosting v2.0</div>
    <nav class="sidebar-nav">
      <a class="nav-item active" onclick="showPage('projects')" id="nav-projects">📋 Проекты</a>
      <div class="nav-divider"></div>
      <div id="projectNav" style="display:none">
        <a class="nav-item" onclick="showPage('code')" id="nav-code">📄 Python-код</a>
        <a class="nav-item" onclick="showPage('dialogs')" id="nav-dialogs">💬 Диалоги</a>
        <a class="nav-item" onclick="showPage('users')" id="nav-users">👥 База данных</a>
        <a class="nav-item" onclick="showPage('requests')" id="nav-requests">📊 Логи заявок</a>
        <a class="nav-item" onclick="showPage('settings')" id="nav-settings">⚙️ Настройки</a>
      </div>
    </nav>
    <div style="border-top:1px solid var(--border);padding-top:16px">
      <a class="nav-item" onclick="doLogout()">🚪 Выход</a>
    </div>
  </aside>

  <!-- CONTENT -->
  <main class="main-content">

    <!-- === PROJECTS PAGE === -->
    <div id="page-projects">
      <div class="topbar">
        <h1>Проекты</h1>
        <div class="topbar-actions">
          <button class="btn btn-primary" onclick="showCreateModal()">+ Новый проект</button>
        </div>
      </div>
      <div id="projectsList"></div>
    </div>

    <!-- === CODE PAGE === -->
    <div id="page-code" style="display:none">
      <div class="topbar">
        <h1>Python-код — <span id="codeProjectName"></span></h1>
        <div class="topbar-actions">
          <button class="btn btn-secondary" onclick="checkCode()">🔍 Проверить</button>
        </div>
      </div>
      <div class="dropzone" id="dropzone" onclick="document.getElementById('fileInput').click()">
        <div class="dropzone-icon">📁</div>
        <div class="dropzone-text">Перетащите ZIP-архив сюда или нажмите для выбора</div>
        <div class="dropzone-hint">Поддерживается: .zip до 50 МБ</div>
      </div>
      <input type="file" id="fileInput" accept=".zip" style="display:none" onchange="uploadCode(this.files[0])">
      <div id="filesList" style="margin-top:24px"></div>
    </div>

    <!-- === DIALOGS PAGE === -->
    <div id="page-dialogs" style="display:none">
      <div class="topbar"><h1>Диалоги — <span id="dialogsProjectName"></span></h1></div>
      <div style="display:flex;gap:20px">
        <div style="width:340px">
          <div class="card" style="padding:0">
            <div id="dialogsList" class="dialog-list"></div>
          </div>
        </div>
        <div style="flex:1">
          <div class="card" style="padding:0" id="chatArea">
            <div class="empty-state">
              <div class="empty-state-icon">💬</div>
              <div class="empty-state-text">Выберите диалог</div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- === USERS PAGE === -->
    <div id="page-users" style="display:none">
      <div class="topbar">
        <h1>База данных — <span id="usersProjectName"></span></h1>
        <div class="topbar-actions">
          <button class="btn btn-secondary btn-sm" onclick="exportCSV()">📥 Экспорт CSV</button>
          <button class="btn btn-secondary btn-sm" onclick="document.getElementById('csvInput').click()">📤 Импорт CSV</button>
          <input type="file" id="csvInput" accept=".csv" style="display:none" onchange="importCSV(this.files[0])">
        </div>
      </div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr>
              <th>ID</th>
              <th>Пользователь</th>
              <th>Username</th>
              <th>Email</th>
              <th>Генерации</th>
              <th>Баланс</th>
              <th>1-я покупка</th>
              <th>Дата</th>
            </tr>
          </thead>
          <tbody id="usersTable"></tbody>
        </table>
      </div>
    </div>

    <!-- === REQUESTS PAGE === -->
    <div id="page-requests" style="display:none">
      <div class="topbar">
        <h1>Логи заявок — <span id="requestsProjectName"></span></h1>
        <div class="topbar-actions">
          <input type="date" class="form-input" style="width:auto" id="dateFrom" onchange="loadRequests()">
          <input type="date" class="form-input" style="width:auto" id="dateTo" onchange="loadRequests()">
        </div>
      </div>
      <div class="table-wrap">
        <table>
          <thead>
            <tr><th>ID</th><th>Пользователь</th><th>Username</th><th>Фото авто</th><th>Фото диска</th><th>Статус</th><th>Дата</th></tr>
          </thead>
          <tbody id="requestsTable"></tbody>
        </table>
      </div>
      <div id="requestsPagination" class="pagination"></div>
    </div>

    <!-- === SETTINGS PAGE === -->
    <div id="page-settings" style="display:none">
      <div class="topbar"><h1>Настройки — <span id="settingsProjectName"></span></h1></div>
      <div class="grid-2">
        <div class="card">
          <div class="card-title" style="margin-bottom:20px">Telegram Bot Token</div>
          <div class="form-group">
            <input type="text" class="form-input" id="tokenInput" placeholder="123456:ABC-DEF...">
          </div>
          <button class="btn btn-primary" onclick="saveToken()">Сохранить токен</button>
          <div id="botInfo" style="margin-top:16px"></div>
        </div>
        <div class="card">
          <div class="card-title" style="margin-bottom:20px">Управление ботом</div>
          <div id="botStatus" style="margin-bottom:20px"></div>
          <div style="display:flex;gap:12px">
            <button class="btn btn-primary" onclick="startBot()">▶️ Запустить</button>
            <button class="btn btn-danger" onclick="stopBot()">⏹ Остановить</button>
          </div>
          <div style="margin-top:20px">
            <button class="btn btn-secondary btn-sm" onclick="showLogs()">📋 Показать логи</button>
          </div>
          <pre id="botLogs" style="margin-top:12px;font-size:12px;color:var(--text-muted);max-height:200px;overflow-y:auto;display:none"></pre>
        </div>
      </div>
      <div class="card" style="margin-top:20px;border-color:var(--danger)">
        <div class="card-title" style="color:var(--danger)">Опасная зона</div>
        <p style="color:var(--text-muted);font-size:14px;margin:12px 0">Удаление проекта необратимо. Все данные будут потеряны.</p>
        <button class="btn btn-danger btn-sm" onclick="deleteCurrentProject()">🗑 Удалить проект</button>
      </div>
    </div>

  </main>
</div>

<!-- === MODAL === -->
<div class="modal-overlay" id="createModal">
  <div class="modal">
    <div class="modal-title">Новый проект</div>
    <div class="form-group">
      <label class="form-label">Название проекта</label>
      <input type="text" class="form-input" id="projectNameInput" placeholder="Мой Telegram-бот">
    </div>
    <div style="display:flex;gap:12px;justify-content:flex-end">
      <button class="btn btn-secondary" onclick="hideCreateModal()">Отмена</button>
      <button class="btn btn-primary" onclick="createProject()">Создать</button>
    </div>
  </div>
</div>

<!-- TOAST -->
<div class="toast" id="toast"></div>

<script src="/app.js"></script>
</body>
</html>
HTMLEOF
```

---

## Шаг 6. Создать JavaScript (логика)

```
cat > /opt/drivegpt/frontend/app.js << 'JSEOF'
const API = '/api';
let currentProject = null;
let currentPage = 'projects';
const ADMIN_EMAIL = 'admin@drivegpt.pro';
const ADMIN_PASSWORD = 'DriveGPT2026!';

// === AUTH ===
function doLogin() {
  const email = document.getElementById('loginEmail').value;
  const password = document.getElementById('loginPassword').value;
  if (email === ADMIN_EMAIL && password === ADMIN_PASSWORD) {
    localStorage.setItem('logged', 'true');
    document.getElementById('loginPage').style.display = 'none';
    document.getElementById('appPage').style.display = 'flex';
    loadProjects();
  } else {
    const err = document.getElementById('loginError');
    err.textContent = 'Неверный email или пароль';
    err.style.display = 'block';
  }
}

function doLogout() {
  localStorage.removeItem('logged');
  location.reload();
}

// Check auth on load
if (localStorage.getItem('logged') === 'true') {
  document.getElementById('loginPage').style.display = 'none';
  document.getElementById('appPage').style.display = 'flex';
  loadProjects();
}

// Enter key on login
document.getElementById('loginPassword').addEventListener('keydown', e => { if (e.key === 'Enter') doLogin(); });

// === NAVIGATION ===
function showPage(page) {
  currentPage = page;
  document.querySelectorAll('.main-content > div').forEach(d => d.style.display = 'none');
  document.getElementById('page-' + page).style.display = 'block';
  document.querySelectorAll('.nav-item').forEach(n => n.classList.remove('active'));
  const nav = document.getElementById('nav-' + page);
  if (nav) nav.classList.add('active');

  if (page === 'projects') loadProjects();
  if (page === 'code' && currentProject) loadFiles();
  if (page === 'dialogs' && currentProject) loadDialogs();
  if (page === 'users' && currentProject) loadUsers();
  if (page === 'requests' && currentProject) loadRequests();
  if (page === 'settings' && currentProject) loadProjectSettings();
}

function selectProject(id, name) {
  currentProject = id;
  document.getElementById('projectNav').style.display = 'block';
  ['code','dialogs','users','requests','settings'].forEach(p => {
    const el = document.getElementById(p + 'ProjectName');
    if (el) el.textContent = name;
  });
  showPage('code');
}

// === API HELPER ===
async function api(endpoint, options = {}) {
  try {
    const resp = await fetch(API + endpoint, {
      headers: { 'Content-Type': 'application/json', ...options.headers },
      ...options
    });
    return await resp.json();
  } catch (e) {
    toast('Ошибка сети: ' + e.message, 'error');
    return null;
  }
}

// === PROJECTS ===
async function loadProjects() {
  const projects = await api('/projects');
  if (!projects) return;
  const el = document.getElementById('projectsList');
  if (projects.length === 0) {
    el.innerHTML = '<div class="empty-state"><div class="empty-state-icon">📋</div><div class="empty-state-text">Нет проектов</div><div class="empty-state-hint">Нажмите «+ Новый проект» для создания</div></div>';
    return;
  }
  el.innerHTML = '<div class="grid-2">' + projects.map(p => {
    const statusClass = p.status === 'active' ? 'running' : p.status === 'failed' ? 'error' : 'stopped';
    return '<div class="card" style="cursor:pointer" onclick="selectProject(' + p.id + ',\'' + esc(p.name) + '\')">' +
      '<div class="card-header"><div class="card-title">' + esc(p.name) + '</div>' +
      '<span class="badge badge-' + statusClass + '"><span class="badge-dot"></span>' + p.status + '</span></div>' +
      '<div style="color:var(--text-muted);font-size:13px">' +
      (p.bot_username ? '🤖 @' + p.bot_username : '🔑 Токен не указан') + '<br>' +
      '📅 ' + formatDate(p.created_at) + '</div></div>';
  }).join('') + '</div>';
}

function showCreateModal() { document.getElementById('createModal').classList.add('active'); }
function hideCreateModal() { document.getElementById('createModal').classList.remove('active'); }

async function createProject() {
  const name = document.getElementById('projectNameInput').value.trim();
  if (!name) return;
  await api('/projects?name=' + encodeURIComponent(name), { method: 'POST' });
  hideCreateModal();
  document.getElementById('projectNameInput').value = '';
  toast('Проект создан!');
  loadProjects();
}

async function deleteCurrentProject() {
  if (!currentProject) return;
  if (!confirm('Удалить проект? Это действие необратимо!')) return;
  await api('/projects/' + currentProject, { method: 'DELETE' });
  currentProject = null;
  document.getElementById('projectNav').style.display = 'none';
  toast('Проект удалён');
  showPage('projects');
}

// === CODE ===
async function loadFiles() {
  const data = await api('/projects/' + currentProject + '/files');
  if (!data || !data.files || data.files.length === 0) {
    document.getElementById('filesList').innerHTML = '<div class="empty-state"><div class="empty-state-icon">📄</div><div class="empty-state-text">Файлы не загружены</div></div>';
    return;
  }
  let html = '<div class="table-wrap"><table><thead><tr><th>Файл</th><th>Размер</th></tr></thead><tbody>';
  data.files.forEach(f => {
    html += '<tr><td>📄 ' + esc(f.path) + '</td><td>' + formatSize(f.size) + '</td></tr>';
  });
  html += '</tbody></table></div>';
  document.getElementById('filesList').innerHTML = html;
}

async function uploadCode(file) {
  if (!file || !currentProject) return;
  const form = new FormData();
  form.append('file', file);
  toast('Загрузка...');
  try {
    const resp = await fetch(API + '/projects/' + currentProject + '/upload', { method: 'POST', body: form });
    const data = await resp.json();
    toast('Загружено ' + data.count + ' файлов!');
    loadFiles();
  } catch (e) { toast('Ошибка загрузки: ' + e.message, 'error'); }
}

async function checkCode() {
  const result = await api('/projects/' + currentProject + '/check', { method: 'POST' });
  if (result && result.ok) toast('✅ Синтаксис ОК!');
  else toast('❌ Ошибки: ' + JSON.stringify(result.errors), 'error');
}

// Drag & Drop
const dz = document.getElementById('dropzone');
if (dz) {
  dz.addEventListener('dragover', e => { e.preventDefault(); dz.classList.add('dragover'); });
  dz.addEventListener('dragleave', () => dz.classList.remove('dragover'));
  dz.addEventListener('drop', e => { e.preventDefault(); dz.classList.remove('dragover'); if (e.dataTransfer.files[0]) uploadCode(e.dataTransfer.files[0]); });
}

// === DIALOGS ===
async function loadDialogs() {
  const dialogs = await api('/projects/' + currentProject + '/dialogs');
  const el = document.getElementById('dialogsList');
  if (!dialogs || dialogs.length === 0) {
    el.innerHTML = '<div class="empty-state" style="padding:40px"><div class="empty-state-icon">💬</div><div class="empty-state-text">Нет диалогов</div></div>';
    return;
  }
  el.innerHTML = dialogs.map(d => {
    const initials = (d.first_name || d.username || '?')[0].toUpperCase();
    return '<div class="dialog-item" onclick="openDialog(' + d.user_id + ')">' +
      '<div class="dialog-avatar">' + initials + '</div>' +
      '<div class="dialog-info"><div class="dialog-name">' + esc(d.first_name || d.username || 'ID: ' + d.telegram_id) + '</div>' +
      '<div class="dialog-preview">' + esc(d.last_message || 'Нет сообщений') + '</div></div>' +
      '<div><div class="dialog-time">' + (d.last_message_at ? formatDate(d.last_message_at) : '') + '</div>' +
      (d.message_count ? '<div class="dialog-count">' + d.message_count + '</div>' : '') + '</div></div>';
  }).join('');
}

async function openDialog(userId) {
  const data = await api('/projects/' + currentProject + '/dialogs/' + userId);
  if (!data) return;
  const area = document.getElementById('chatArea');
  let html = '<div class="chat-container"><div style="padding:16px 20px;border-bottom:1px solid var(--border);font-weight:600">' +
    esc(data.user.first_name || data.user.username || 'Пользователь') +
    (data.user.username ? ' <span style="color:var(--text-muted);font-weight:400">@' + data.user.username + '</span>' : '') + '</div>' +
    '<div class="chat-messages" style="display:flex;flex-direction:column">';
  data.messages.forEach(m => {
    const cls = m.direction === 'in' ? 'incoming' : 'outgoing';
    const sender = m.sent_by === 'operator' ? '👤 Оператор' : (m.direction === 'out' ? '🤖 Бот' : '');
    html += '<div class="chat-message ' + cls + '">' +
      (sender ? '<div class="msg-sender">' + sender + '</div>' : '') +
      esc(m.text || '[медиа]') +
      '<div class="msg-time">' + formatTime(m.created_at) + '</div></div>';
  });
  html += '</div><div class="chat-input-wrap">' +
    '<input class="chat-input" id="chatInput" placeholder="Написать от имени бота..." onkeydown="if(event.key===\'Enter\')sendMsg(' + userId + ')">' +
    '<button class="chat-send" onclick="sendMsg(' + userId + ')">➤</button></div></div>';
  area.innerHTML = html;
}

async function sendMsg(userId) {
  const input = document.getElementById('chatInput');
  const text = input.value.trim();
  if (!text) return;
  input.value = '';
  await api('/projects/' + currentProject + '/dialogs/' + userId + '/send?text=' + encodeURIComponent(text), { method: 'POST' });
  openDialog(userId);
}

// === USERS ===
async function loadUsers() {
  const users = await api('/projects/' + currentProject + '/users');
  const tbody = document.getElementById('usersTable');
  if (!users || users.length === 0) {
    tbody.innerHTML = '<tr><td colspan="8" style="text-align:center;color:var(--text-muted);padding:40px">Нет пользователей</td></tr>';
    return;
  }
  tbody.innerHTML = users.map(u =>
    '<tr><td>' + u.id + '</td>' +
    '<td>' + esc(u.first_name || '') + ' ' + esc(u.last_name || '') + '</td>' +
    '<td>' + (u.username ? '@' + esc(u.username) : '—') + '</td>' +
    '<td>' + (u.email || '—') + '</td>' +
    '<td>' + u.generations_count + '</td>' +
    '<td>' + u.generations_balance + '</td>' +
    '<td>' + (u.first_purchase ? '✅' : '—') + '</td>' +
    '<td>' + formatDate(u.created_at) + '</td></tr>'
  ).join('');
}

function exportCSV() {
  window.open(API + '/projects/' + currentProject + '/users-export');
}

async function importCSV(file) {
  if (!file) return;
  const form = new FormData();
  form.append('file', file);
  const resp = await fetch(API + '/projects/' + currentProject + '/users-import', { method: 'POST', body: form });
  const data = await resp.json();
  toast('Импортировано: ' + data.imported + ' пользователей');
  loadUsers();
}

// === REQUESTS ===
let reqPage = 1;
async function loadRequests(page) {
  if (page) reqPage = page;
  let url = '/projects/' + currentProject + '/requests?page=' + reqPage;
  const df = document.getElementById('dateFrom').value;
  const dt = document.getElementById('dateTo').value;
  if (df) url += '&date_from=' + df;
  if (dt) url += '&date_to=' + dt;
  const data = await api(url);
  if (!data) return;
  const tbody = document.getElementById('requestsTable');
  if (!data.items || data.items.length === 0) {
    tbody.innerHTML = '<tr><td colspan="7" style="text-align:center;color:var(--text-muted);padding:40px">Нет заявок</td></tr>';
    document.getElementById('requestsPagination').innerHTML = '';
    return;
  }
  tbody.innerHTML = data.items.map(r =>
    '<tr><td>' + r.id + '</td><td>' + esc(r.user_name) + '</td><td>' + (r.username ? '@' + esc(r.username) : '—') + '</td>' +
    '<td>' + (r.car_photo_url ? '<a href="' + r.car_photo_url + '" target="_blank">📷</a>' : '—') + '</td>' +
    '<td>' + (r.wheel_photo_url ? '<a href="' + r.wheel_photo_url + '" target="_blank">📷</a>' : '—') + '</td>' +
    '<td><span class="badge badge-' + r.status + '"><span class="badge-dot"></span>' + r.status + '</span></td>' +
    '<td>' + formatDate(r.created_at) + '</td></tr>'
  ).join('');
  // Pagination
  let phtml = '';
  for (let i = 1; i <= data.pages; i++) {
    phtml += '<button class="page-btn' + (i === data.page ? ' active' : '') + '" onclick="loadRequests(' + i + ')">' + i + '</button>';
  }
  document.getElementById('requestsPagination').innerHTML = phtml;
}

// === SETTINGS ===
async function loadProjectSettings() {
  const p = await api('/projects/' + currentProject);
  if (!p) return;
  if (p.telegram_token && p.telegram_token !== '***') document.getElementById('tokenInput').placeholder = p.telegram_token;
  document.getElementById('botInfo').innerHTML = p.bot_username ?
    '<div style="color:var(--success)">🤖 @' + p.bot_username + ' (' + esc(p.bot_name || '') + ')</div>' : '';
  const st = await api('/projects/' + currentProject + '/status');
  if (st) {
    const cls = st.status === 'active' ? 'running' : st.status === 'failed' ? 'error' : 'stopped';
    document.getElementById('botStatus').innerHTML = '<span class="badge badge-' + cls + '"><span class="badge-dot"></span>' + st.status + '</span>' +
      (st.webhook_url ? '<div style="font-size:12px;color:var(--text-muted);margin-top:8px">Webhook: ' + st.webhook_url + '</div>' : '') +
      (st.error_message ? '<div style="font-size:12px;color:var(--danger);margin-top:8px">' + esc(st.error_message) + '</div>' : '');
  }
}

async function saveToken() {
  const token = document.getElementById('tokenInput').value.trim();
  if (!token) return toast('Введите токен', 'error');
  const result = await api('/projects/' + currentProject + '/token?token=' + encodeURIComponent(token), { method: 'PUT' });
  if (result && result.status === 'ok') {
    toast('Токен сохранён! Бот: @' + result.bot_username);
    loadProjectSettings();
  } else {
    toast('Ошибка: ' + (result?.detail || 'Неверный токен'), 'error');
  }
}

async function startBot() {
  toast('Запуск бота...');
  const result = await api('/projects/' + currentProject + '/start', { method: 'POST' });
  if (result && result.status === 'running') toast('✅ Бот запущен!');
  else toast('Ошибка: ' + (result?.detail || JSON.stringify(result)), 'error');
  loadProjectSettings();
}

async function stopBot() {
  const result = await api('/projects/' + currentProject + '/stop', { method: 'POST' });
  if (result && result.status === 'stopped') toast('Бот остановлен');
  else toast('Ошибка', 'error');
  loadProjectSettings();
}

async function showLogs() {
  const el = document.getElementById('botLogs');
  el.style.display = el.style.display === 'none' ? 'block' : 'none';
  if (el.style.display === 'block') {
    const data = await api('/projects/' + currentProject + '/logs?lines=30');
    el.textContent = data?.logs || 'Нет логов';
  }
}

// === HELPERS ===
function esc(s) { if (!s) return ''; const d = document.createElement('div'); d.textContent = s; return d.innerHTML; }
function formatDate(s) { if (!s || s === 'None') return '—'; try { return new Date(s).toLocaleDateString('ru-RU'); } catch(e) { return s; } }
function formatTime(s) { if (!s || s === 'None') return ''; try { return new Date(s).toLocaleTimeString('ru-RU', {hour:'2-digit',minute:'2-digit'}); } catch(e) { return s; } }
function formatSize(b) { if (b < 1024) return b + ' Б'; if (b < 1024*1024) return (b/1024).toFixed(1) + ' КБ'; return (b/1024/1024).toFixed(1) + ' МБ'; }

function toast(msg, type = 'success') {
  const el = document.getElementById('toast');
  el.textContent = msg;
  el.className = 'toast toast-' + type + ' show';
  setTimeout(() => el.classList.remove('show'), 3000);
}
JSEOF
```

---

## Шаг 7. Проверить и обновить HTTPS

```
nginx -t && systemctl reload nginx
```

Если нужно обновить SSL:
```
certbot --nginx -d drivegpt.pro -d api.drivegpt.pro -d app.drivegpt.pro --non-interactive --agree-tos -m hotel-go@yandex.ru
```

---

## Шаг 8. Открыть в браузере

```
https://app.drivegpt.pro
```

**Логин:**
- Email: `admin@drivegpt.pro`
- Пароль: `DriveGPT2026!`

---

## ✅ Итоги дня 4

| Что создано | Описание |
|------------|----------|
| `styles.css` | Полная дизайн-система (тёмная тема, кнопки, таблицы, чат) |
| `index.html` | SPA с 6 страницами |
| `app.js` | Вся логика (API-запросы, навигация, D&D, чат) |
| Nginx обновлён | app / api / основной — разделены |

### Страницы:
- 🔐 **Вход** — email + пароль
- 📋 **Проекты** — список с карточками
- 📄 **Python-код** — загрузка ZIP + проверка синтаксиса
- 💬 **Диалоги** — мессенджер с ответом оператора
- 👥 **База данных** — таблица + CSV
- 📊 **Логи заявок** — пагинация + фильтр по датам
- ⚙️ **Настройки** — токен, старт/стоп, логи, удаление

---

## ➡️ День 5: Тестирование с реальным ботом

- Создать тестовый Telegram-бот
- Загрузить Python-код
- Проверить webhook
- Отправить сообщение из ЛК
