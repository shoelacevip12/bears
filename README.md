![](./0bnj-rVRM76YUaX66QSkc_0YyixHTZE8sxAf3u8GFMSi1s86uD84Oy2B9tHhHG_xNWhjQCjIShX1mzGcdc9rxoG8.jpg)
# TeamSpeak 6 + ts6-manager Docker Setup

Проект `ts6-manager` от clusterzx подходит для управления TeamSpeak 6, так как он использует современный **WebQuery HTTP API** (а не устаревший Telnet)

Ниже представлен объединенный и оптимизированный `docker-compose.yml`, файл переменных окружения `.env` и пошаговая инструкция по настройке связки.

## 1. Объединенный `docker-compose.yml`

Сервер `teamspeak6` объединен с сервисами менеджера (`backend`, `frontend`, `sidecar`). Они будут работать в одной сети `ts6-network`, что позволит менеджеру обращаться к серверу по внутреннему имени хоста `teamspeak6`.

Все пароли, ключи и локальные пути вынесены в файл `.env` для безопасности и удобства настройки.

```yaml
version: '3.8'

services:
  # ==========================================
  # TeamSpeak 6 Server
  # ==========================================
  teamspeak6:
    image: teamspeaksystems/teamspeak6-server:latest
    container_name: teamspeak6
    restart: unless-stopped
    networks:
      - ts6-network
    ports:
      - "9987:9987/udp"      # Voice UDP
      - "30033:30033/tcp"    # File Transfer TCP
      - "10022:10022/tcp"    # Query SSH TCP
      - "10080:10080/tcp"    # Query HTTP TCP (Основной для ts6-manager)
      - "10443:10443/tcp"    # Query HTTPS TCP
      - "10011:10011/tcp"    # Legacy Query TCP
      - "3478:3478/udp"      # TURN/STUN UDP
      - "5349:5349/udp"      # TURN/STUN UDP
      - "41144:41144/tcp"    # WebQuery TCP
    volumes:
      - ${TS6_DATA_PATH}:/var/tsserver
    environment:
      - PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin:/opt/tsserver
      - TSSERVER_DATABASE_SQL_PATH=/opt/tsserver/sql/
      - TSSERVER_DATABASE_SQL_CREATE_PATH=/opt/tsserver/sql/create_sqlite/
      - TSSERVER_QUERY_DOCUMENTATION_PATH=/opt/tsserver/serverquerydocs
      - TSSERVER_LICENSE_ACCEPTED=accept
      - TSSERVER_DEFAULT_PORT=9987
      - TSSERVER_VOICE_IP=0.0.0.0
      - TSSERVER_FILE_TRANSFER_PORT=30033
      - TSSERVER_QUERY_HTTP_ENABLED=true
      - TSSERVER_QUERY_SSH_ENABLED=true
      - TSSERVER_QUERY_ADMIN_PASSWORD=${TS6_QUERY_ADMIN_PASSWORD}
      - TSSERVER_QUERY_HTTP_PORT=10080
      - TSSERVER_QUERY_SSH_PORT=10022
      - TSSERVER_VOICE_UDP_THREADS=16
      - TSSERVER_DATABASE_SKIP_INTEGRITY_CHECK=true
      - TSSERVER_QUERY_POOL_SIZE=32
      - TSSERVER_QUERY_LOG_COMMANDS=true
      - TSSERVER_QUERY_BUFFER_MB=100

  # ==========================================
  # TS6 Manager - Backend (API и логика)
  # ==========================================
  backend:
    image: clusterzx/ts6-manager:backend
    container_name: ts6-backend
    restart: unless-stopped
    environment:
      - NODE_ENV=production
      - PORT=3001
      - DATABASE_URL=file:/app/packages/backend/data/ts6webui.db
      - JWT_SECRET=${JWT_SECRET}
      - ENCRYPTION_KEY=${ENCRYPTION_KEY}
      - TS_ALLOW_SELF_SIGNED=${TS_ALLOW_SELF_SIGNED:-true}
      - JWT_ACCESS_EXPIRY=15m
      - JWT_REFRESH_EXPIRY=7d
      - FRONTEND_URL=${FRONTEND_URL}
      - MUSIC_DIR=/data/music
      - SIDECAR_URL=http://ts6-sidecar:9800
    volumes:
      - ${BACKEND_DATA_PATH}:/app/packages/backend/data
      - ${MUSIC_DATA_PATH}:/data/music
    ports:
      - "3001:3001"
    depends_on:
      - teamspeak6
      - sidecar
    networks:
      - ts6-network

  # ==========================================
  # TS6 Manager - Sidecar (WebRTC Streaming)
  # ==========================================
  sidecar:
    image: clusterzx/ts6-manager:sidecar
    container_name: ts6-sidecar
    restart: unless-stopped
    environment:
      - SIDECAR_PORT=9800
    ports:
      - "9800:9800"
    networks:
      - ts6-network

  # ==========================================
  # TS6 Manager - Frontend (Web-интерфейс)
  # ==========================================
  frontend:
    image: clusterzx/ts6-manager:frontend
    container_name: ts6-frontend
    restart: unless-stopped
    ports:
      - "${FRONTEND_PORT:-3000}:80"
    depends_on:
      - backend
    networks:
      - ts6-network

networks:
  ts6-network:
    driver: bridge
    name: ts6-network
```

---

## 2. Файл `.env`

Создайте файл с именем `.env` в той же папке, где лежит `docker-compose.yml`. Все чувствительные данные и пути вынесены сюда.

```bash
# ==========================================
# СЕКРЕТНЫЕ КЛЮЧИ (обязательно измените!)
# ==========================================
# Сгенерируйте случайные строки (минимум 32 символа).
# В Linux это можно сделать командой: openssl rand -hex 32
JWT_SECRET=замените_на_очень_длинную_случайную_строку_минимум_32_символа
ENCRYPTION_KEY=замените_на_другую_длинную_случайную_строку_минимум_32_символа

# ==========================================
# НАСТРОЙКИ TEAM SPEAK 6
# ==========================================
# Пароль администратора ServerQuery (SSH)
TS6_QUERY_ADMIN_PASSWORD=придумайте_надёжный_пароль_для_serveradmin

# ==========================================
# ПУТИ К ДАННЫМ (локальные пути на хосте)
# ==========================================
# Путь к данным TeamSpeak 6 сервера
TS6_DATA_PATH=/volume1/teamspeak/ts6

# Путь к данным backend (база данных менеджера)
BACKEND_DATA_PATH=/volume1/docker/ts6_project/backend/backend-data

# Путь к музыке для стриминга
MUSIC_DATA_PATH=/volume1/docker/ts6_project/backend/music-data

# ==========================================
# НАСТРОЙКИ ВЕБ-ИНТЕРФЕЙСА
# ==========================================
# URL, по которому будет доступен веб-интерфейс (укажите IP вашего сервера или домен)
FRONTEND_URL=http://localhost:3000

# Порт для веб-интерфейса (по умолчанию 3000)
FRONTEND_PORT=3000

# TeamSpeak 6 по умолчанию использует самоподписанные сертификаты для WebQuery
TS_ALLOW_SELF_SIGNED=true
```

> **Важно:** Перед запуском обязательно замените все значения-заглушки (`замените_на_...`, `придумайте_...`) на реальные значения!

---

## 3. Пошаговая инструкция по запуску

1. Сохраните оба файла (`docker-compose.yml` и `.env`) в одну директорию (например, `/volume1/teamspeak/`).
2. Отредактируйте файл `.env`:
   - Сгенерируйте уникальные `JWT_SECRET` и `ENCRYPTION_KEY` (минимум 32 символа каждый)
   - Задайте надёжный пароль для `TS6_QUERY_ADMIN_PASSWORD`
   - Проверьте и при необходимости измените пути в переменных `*_PATH` под вашу систему
3. Убедитесь, что указанные в `.env` папки существуют и у Docker есть права на запись в них:

   ```bash
   mkdir -vp /volume1/teamspeak/ts6
   mkdir -vp /volume1/docker/ts6_project/backend/backend-data
   mkdir -vp /volume1/docker/ts6_project/backend/music-data
   # !!Ленивый способ предосталения полных прав!!
   chmod -vR 777 /volume1/teamspeak/ts6
   chmod -vR 777 /volume1/docker/ts6_project/backend
   ```

4. Запустите стек командой:

   ```bash
   docker compose up -d
   ```

5. Проверьте статус контейнеров:

   ```bash
   docker compose ps
   ```

   Все 4 контейнера должны быть в состоянии `Up`.

---

## 4. Настройка подключения Manager к TeamSpeak 6

Это самый важный шаг. `ts6-manager` **не использует** пароль администратора (`TS6_QUERY_ADMIN_PASSWORD`) для прямого входа. Он требует **API Key**, сгенерированный на сервере.

### Шаг 4.1: Первоначальная настройка веб-интерфейса

1. Откройте веб-интерфейс менеджера: `http://<IP-вашего-сервера>:${FRONTEND_PORT}/setup`
2. Создайте аккаунт администратора веб-панели:
   - **Username**: `admin` (или любой другой)
   - **Display Name**: Ваше имя
   - **Password**: Придумайте надёжный пароль

### Шаг 4.2: Добавление подключения к TeamSpeak 6

1. После входа перейдите в **Settings → Connections** (Настройки → Подключения).
2. Нажмите **Add Connection** и заполните поля:
   - **Name**: `Local TS6` (или любое другое)
   - **Host**: `teamspeak6` (это внутреннее DNS-имя в Docker сети) или `<IP-вашего-сервера>`
   - **Port**: `10080`
   - **Protocol**: `HTTP` (убедитесь, что выбрано именно HTTP, а не Telnet)
   - **API Key**: (см. инструкцию ниже, как его получить)

### Шаг 4.3: Как получить API Key для TeamSpeak 6

Поскольку у вас включен SSH Query (`TSSERVER_QUERY_SSH_ENABLED=true` и порт `10022`), вы можете подключиться к серверу и создать ключ:

1. Подключитесь по SSH к порту ServerQuery (с вашего компьютера или сервера):

   ```bash
   ssh -p 10022 serveradmin@<IP-вашего-сервера>
   ```

   *(Пароль: значение из `.env` переменной `TS6_QUERY_ADMIN_PASSWORD`)*

2. После успешного входа выполните команду для создания ключа:

   ```text
   apikeyadd scope=manage duration=0
   ```

3. Сервер ответит строкой вида:

   ```text
   apikey=BAD_jVXxx8xxXxLjI4xxxxXXyjexNlddQIUvUGu id=5 sid=0 cldbid=1 scope=manage time_left=1209599 created_at=1789680984 expires_at=1790890584 custom_id
   ```

   Скопируйте значение после `apikey=` (в данном примере: `BAD_jVXxx8xxXxLjI4xxxxXXyjexNlddQIUvUGu`) и вставьте его в поле **API Key** в настройках `ts6-manager`.

---

## 5. Важные примечания

- Путь `/volume1/...` указывает на то, что вы, вероятно, используете **Synology NAS**. Убедитесь, что в настройках брандмауэра NAS открыты порты:
  - `9987/UDP` — голосовой трафик
  - `30033/TCP` — передача файлов
  - `10080/TCP` — WebQuery HTTP (для менеджера)
  - `${FRONTEND_PORT}/TCP` (по умолчанию `3000`) — веб-интерфейс менеджера
  - `9800/TCP` — Sidecar (WebRTC стриминг)

- Если вы планируете использовать функцию стриминга музыки/видео через Sidecar, убедитесь, что порт `9800` также доступен, или настройте обратный прокси (Nginx/Traefik) для маршрутизации трафика.

- Переменная `TS_ALLOW_SELF_SIGNED=true` в `.env` критически важна, так как официальный образ TS6 генерирует самоподписанный SSL-сертификат для WebQuery при первом запуске, и backend менеджера должен иметь разрешение ему доверять.

- **Безопасность**: Не забудьте изменить стандартные порты в production-среде и используйте надёжные пароли. Храните файл `.env` в безопасном месте и никогда не коммитьте его в системы контроля версий (добавьте в `.gitignore`).

---

## 6. Быстрая генерация секретов

Для быстрой генерации случайных строк для `.env` используйте следующие команды:

```bash
# Генерация JWT_SECRET (32 байта = 64 hex-символа)
echo "JWT_SECRET=$(openssl rand -hex 32)"

# Генерация ENCRYPTION_KEY (32 байта = 64 hex-символа)
echo "ENCRYPTION_KEY=$(openssl rand -hex 32)"

# Генерация надёжного пароля для TS6
echo "TS6_QUERY_ADMIN_PASSWORD=$(openssl rand -base64 24)"
```

Скопируйте вывод этих команд и вставьте соответствующие значения в файл `.env`.
