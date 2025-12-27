# Развертывание Matrix Synapse на Windows с Docker Desktop

Полное руководство по установке собственного Matrix homeserver на Windows за 30-60 минут.

---

## 1. Подготовка Windows окружения

### 1.1 Установка Docker Desktop для Windows

1. **Скачайте Docker Desktop:**
   - Перейдите на [https://www.docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
   - Скачайте установщик для Windows

2. **Запустите установщик:**
   - Дважды кликните на `Docker Desktop Installer.exe`
   - Убедитесь, что опция "Use WSL 2 instead of Hyper-V" отмечена (рекомендуется)
   - Следуйте инструкциям установщика

3. **Перезагрузите компьютер** после установки

### 1.2 Проверка WSL2

WSL2 (Windows Subsystem for Linux 2) обеспечивает лучшую производительность Docker на Windows.

**Откройте PowerShell от имени администратора** и выполните:

```powershell
wsl --list --verbose
```

Если WSL2 не установлен, выполните:

```powershell
wsl --install
```

После установки перезагрузите компьютер.

### 1.3 Настройка Docker Desktop

1. **Запустите Docker Desktop** из меню Пуск
2. Дождитесь полного запуска (иконка Docker в трее должна стать активной)
3. **Откройте настройки Docker Desktop:**
   - Кликните правой кнопкой на иконку Docker в трее
   - Выберите "Settings"
4. **Рекомендуемые настройки:**
   - **Resources → Advanced:** Выделите минимум 4GB RAM и 2 CPU
   - **General:** Убедитесь, что "Use the WSL 2 based engine" включено

### 1.4 Проверка работоспособности Docker

Откройте **PowerShell** и выполните:

```powershell
docker --version
docker-compose --version
docker ps
```

Все команды должны выполниться без ошибок.

**Тестовый запуск:**

```powershell
docker run hello-world
```

Если вы видите сообщение "Hello from Docker!", все работает корректно.

---

## 2. Развертывание Matrix Synapse

### 2.1 Создание структуры папок

Откройте **PowerShell** и выполните следующие команды:

```powershell
# Создаем основную папку для Matrix
New-Item -Path "C:\Matrix" -ItemType Directory -Force

# Переходим в созданную папку
Set-Location C:\Matrix

# Создаем подпапки для данных
New-Item -Path "synapse-data" -ItemType Directory -Force
New-Item -Path "postgres-data" -ItemType Directory -Force

# Проверяем структуру
Get-ChildItem
```

Структура папок должна выглядеть так:
```
C:\Matrix\
├── synapse-data\
├── postgres-data\
└── docker-compose.yml (создадим далее)
```

### 2.2 Создание docker-compose.yml

В папке `C:\Matrix` создайте файл `docker-compose.yml` с помощью Блокнота или любого текстового редактора:

```powershell
notepad docker-compose.yml
```

Вставьте следующий код:

```yaml
version: '3.8'

services:
  # PostgreSQL база данных для Matrix Synapse
  postgres:
    image: postgres:15-alpine
    container_name: matrix-postgres
    restart: unless-stopped
    environment:
      # Пароль для базы данных - ИЗМЕНИТЕ НА СВОЙ!
      POSTGRES_PASSWORD: matrix_secure_password_123
      POSTGRES_USER: synapse
      POSTGRES_DB: synapse
      # Настройки для улучшения производительности
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
    volumes:
      # Данные БД хранятся в Windows папке
      - ./postgres-data:/var/lib/postgresql/data
    networks:
      - matrix-net
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U synapse"]
      interval: 10s
      timeout: 5s
      retries: 5

  # Matrix Synapse homeserver
  synapse:
    image: matrixdotorg/synapse:latest
    container_name: matrix-synapse
    restart: unless-stopped
    ports:
      # Основной порт Matrix (HTTP)
      - "8008:8008"
      # Порт для federation (связь с другими серверами)
      # Раскомментируйте если планируете федерацию
      # - "8448:8448"
    volumes:
      # Конфигурация и данные Synapse в Windows папке
      - ./synapse-data:/data
    environment:
      # Имя вашего homeserver (можно использовать localhost для тестов)
      - SYNAPSE_SERVER_NAME=localhost
      - SYNAPSE_REPORT_STATS=no
    networks:
      - matrix-net
    depends_on:
      postgres:
        condition: service_healthy

networks:
  # Внутренняя сеть для контейнеров
  matrix-net:
    driver: bridge
```

**⚠️ ВАЖНО:** Обязательно измените `POSTGRES_PASSWORD` на свой надежный пароль!

Сохраните файл (Ctrl+S) и закройте Блокнот.

### 2.3 Генерация конфигурации Synapse

Теперь нужно сгенерировать файл конфигурации `homeserver.yaml`:

```powershell
# Убедитесь, что вы в папке C:\Matrix
Set-Location C:\Matrix

# Генерируем конфигурацию
docker run -it --rm `
  -v ${PWD}/synapse-data:/data `
  -e SYNAPSE_SERVER_NAME=localhost `
  -e SYNAPSE_REPORT_STATS=no `
  matrixdotorg/synapse:latest generate
```

**Примечание:** Символ `` ` `` (обратный апостроф) используется в PowerShell для переноса длинных команд на новую строку.

После выполнения команды в папке `C:\Matrix\synapse-data` появится файл `homeserver.yaml`.

### 2.4 Настройка homeserver.yaml

Откройте файл для редактирования:

```powershell
notepad C:\Matrix\synapse-data\homeserver.yaml
```

**Найдите и измените следующие параметры:**

#### А) Настройка базы данных PostgreSQL

Найдите секцию `database:` (примерно строка 1000-1100) и замените на:

```yaml
database:
  name: psycopg2
  args:
    user: synapse
    password: matrix_secure_password_123  # ВАШ ПАРОЛЬ из docker-compose.yml
    database: synapse
    host: postgres
    port: 5432
    cp_min: 5
    cp_max: 10
```

#### Б) Включение регистрации пользователей

Найдите строку:
```yaml
enable_registration: false
```

Измените на:
```yaml
enable_registration: true
enable_registration_without_verification: true
```

**⚠️ БЕЗОПАСНОСТЬ:** После создания учетных записей рекомендуется отключить открытую регистрацию!

#### В) Настройка прослушивания портов

Найдите секцию `listeners:` и убедитесь, что она выглядит так:

```yaml
listeners:
  - port: 8008
    tls: false
    type: http
    x_forwarded: true
    bind_addresses: ['0.0.0.0']
    resources:
      - names: [client, federation]
        compress: false
```

#### Г) Отключение требования email (опционально)

Найдите и закомментируйте или удалите секцию `email:` если не планируете настраивать почту:

```yaml
# email:
#   smtp_host: ...
```

Сохраните файл (Ctrl+S) и закройте Блокнот.

### 2.5 Запуск контейнеров

Вернитесь в PowerShell и выполните:

```powershell
# Убедитесь, что вы в папке C:\Matrix
Set-Location C:\Matrix

# Запускаем контейнеры в фоновом режиме
docker-compose up -d
```

Проверьте статус контейнеров:

```powershell
docker-compose ps
```

Вы должны увидеть два запущенных контейнера: `matrix-postgres` и `matrix-synapse` со статусом `Up`.

**Проверка логов:**

```powershell
# Логи Synapse
docker-compose logs synapse

# Логи в реальном времени
docker-compose logs -f synapse
```

Нажмите `Ctrl+C` чтобы остановить просмотр логов.

---

## 3. Создание пользователей и первый вход

### 3.1 Создание администратора

Создайте первого пользователя с правами администратора:

```powershell
docker exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -a
```

Вам будет предложено ввести:
- **Username:** например, `admin`
- **Password:** ваш пароль
- **Confirm password:** повторите пароль
- **Make admin [no]:** введите `yes`

Ваш полный Matrix ID будет: `@admin:localhost`

### 3.2 Создание обычных пользователей

Для создания обычных пользователей (без `-a` флага):

```powershell
docker exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml
```

---

## 4. Подключение клиента

### 4.1 Установка Element Desktop

1. Скачайте Element с [https://element.io/download](https://element.io/download)
2. Установите Element Desktop на Windows

### 4.2 Подключение к вашему серверу

1. Запустите Element
2. Нажмите **"Create Account"** или **"Sign In"**
3. Нажмите **"Edit"** возле имени сервера
4. Введите адрес вашего homeserver:
   ```
   http://localhost:8008
   ```
5. Нажмите **"Continue"**
6. Войдите используя созданные учетные данные:
   - Username: `admin` (без `@` и домена)
   - Password: ваш пароль

**🎉 Поздравляем! Вы подключились к собственному Matrix серверу!**

### 4.3 Проверка работоспособности веб-интерфейсом

Альтернативный способ через браузер:

1. Откройте [https://app.element.io](https://app.element.io)
2. При входе укажите Custom homeserver: `http://localhost:8008`
3. Войдите с вашими учетными данными

---

## 5. Управление сервером

### 5.1 Основные команды PowerShell

**Просмотр статуса:**
```powershell
Set-Location C:\Matrix
docker-compose ps
```

**Остановка сервера:**
```powershell
docker-compose stop
```

**Запуск сервера:**
```powershell
docker-compose start
```

**Перезапуск сервера:**
```powershell
docker-compose restart
```

**Полная остановка и удаление контейнеров (данные сохраняются):**
```powershell
docker-compose down
```

**Пересоздание контейнеров с новой конфигурацией:**
```powershell
docker-compose up -d --force-recreate
```

### 5.2 Просмотр логов

**Все логи:**
```powershell
docker-compose logs
```

**Только Synapse:**
```powershell
docker-compose logs synapse
```

**Последние 100 строк:**
```powershell
docker-compose logs --tail=100 synapse
```

**Логи в реальном времени:**
```powershell
docker-compose logs -f synapse
```

### 5.3 Резервное копирование

#### Ручной бэкап

**Остановите контейнеры:**
```powershell
Set-Location C:\Matrix
docker-compose stop
```

**Создайте архив:**
```powershell
# Создаем папку для бэкапов
New-Item -Path "C:\Matrix-Backups" -ItemType Directory -Force

# Архивируем данные (требуется 7-Zip или встроенный Compress-Archive)
Compress-Archive -Path "C:\Matrix\synapse-data" -DestinationPath "C:\Matrix-Backups\synapse-backup-$(Get-Date -Format 'yyyy-MM-dd').zip"
Compress-Archive -Path "C:\Matrix\postgres-data" -DestinationPath "C:\Matrix-Backups\postgres-backup-$(Get-Date -Format 'yyyy-MM-dd').zip"
```

**Запустите контейнеры обратно:**
```powershell
docker-compose start
```

#### Автоматизация бэкапов

Создайте скрипт `C:\Matrix\backup.ps1`:

```powershell
# backup.ps1
Set-Location C:\Matrix
docker-compose stop

$date = Get-Date -Format 'yyyy-MM-dd-HHmm'
$backupDir = "C:\Matrix-Backups\$date"
New-Item -Path $backupDir -ItemType Directory -Force

Copy-Item -Path "C:\Matrix\synapse-data" -Destination "$backupDir\synapse-data" -Recurse
Copy-Item -Path "C:\Matrix\postgres-data" -Destination "$backupDir\postgres-data" -Recurse

docker-compose start
Write-Host "Backup completed: $backupDir"
```

Запуск бэкапа:
```powershell
powershell -ExecutionPolicy Bypass -File C:\Matrix\backup.ps1
```

### 5.4 Восстановление из бэкапа

```powershell
# Остановите контейнеры
Set-Location C:\Matrix
docker-compose down

# Удалите текущие данные
Remove-Item -Path "C:\Matrix\synapse-data" -Recurse -Force
Remove-Item -Path "C:\Matrix\postgres-data" -Recurse -Force

# Восстановите из архива или скопируйте папки бэкапа
Copy-Item -Path "C:\Matrix-Backups\2024-01-15\synapse-data" -Destination "C:\Matrix\synapse-data" -Recurse
Copy-Item -Path "C:\Matrix-Backups\2024-01-15\postgres-data" -Destination "C:\Matrix\postgres-data" -Recurse

# Запустите контейнеры
docker-compose up -d
```

---

## 6. Доступ из локальной сети

### 6.1 Узнайте IP-адрес вашего компьютера

В PowerShell:
```powershell
ipconfig
```

Найдите вашу локальную сеть (обычно `192.168.x.x` или `10.0.x.x`).

### 6.2 Настройка Firewall

**Откройте порт 8008 в Windows Firewall:**

```powershell
# Запустите PowerShell от имени администратора
New-NetFirewallRule -DisplayName "Matrix Synapse" -Direction Inbound -LocalPort 8008 -Protocol TCP -Action Allow
```

### 6.3 Изменение конфигурации для локальной сети

Если вы хотите использовать IP-адрес вместо localhost:

1. Откройте `homeserver.yaml`:
```powershell
notepad C:\Matrix\synapse-data\homeserver.yaml
```

2. Найдите `server_name: localhost` и измените на ваш IP:
```yaml
server_name: 192.168.1.100  # Замените на ваш IP
```

3. **⚠️ ВАЖНО:** После изменения `server_name` база данных должна быть пересоздана! Или используйте DNS имя с самого начала.

**Рекомендация:** Используйте `localhost` для тестирования, а для продакшена настройте доменное имя.

### 6.4 Подключение с других устройств

На других устройствах в вашей локальной сети используйте:
```
http://192.168.1.100:8008
```
(замените на ваш IP)

---

## 7. Настройка HTTPS (опционально)

### 7.1 С использованием Caddy (простой способ)

Добавьте Caddy в `docker-compose.yml`:

```yaml
  caddy:
    image: caddy:alpine
    container_name: matrix-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile
      - ./caddy-data:/data
      - ./caddy-config:/config
    networks:
      - matrix-net
```

Создайте файл `Caddyfile` в `C:\Matrix`:

```
matrix.yourdomain.com {
    reverse_proxy synapse:8008
}
```

**Примечание:** Требуется настроенный домен и открытые порты 80/443 на роутере.

---

## 8. Решение типичных проблем

### Проблема 1: Docker не запускается

**Решение:**
- Убедитесь, что виртуализация включена в BIOS
- Проверьте, что WSL2 установлен: `wsl --install`
- Перезапустите Docker Desktop
- Перезагрузите компьютер

### Проблема 2: Контейнеры не запускаются

**Проверка:**
```powershell
docker-compose logs
```

**Частые причины:**
- Порт 8008 уже занят другим приложением
- Недостаточно RAM (минимум 4GB)
- Поврежденная конфигурация в `homeserver.yaml`

**Решение для занятого порта:**
Измените порт в `docker-compose.yml`:
```yaml
ports:
  - "8009:8008"  # Используем 8009 вместо 8008
```

### Проблема 3: Не могу подключиться из клиента

**Проверки:**
1. Контейнеры запущены: `docker-compose ps`
2. Порт открыт: `Test-NetConnection -ComputerName localhost -Port 8008`
3. Правильный адрес сервера в клиенте
4. Firewall не блокирует соединение

### Проблема 4: Медленная работа

**Оптимизация:**

1. **Увеличьте ресурсы Docker:**
   - Docker Desktop → Settings → Resources
   - RAM: минимум 4GB, рекомендуется 8GB
   - CPU: минимум 2, рекомендуется 4

2. **Настройки PostgreSQL** в `docker-compose.yml`:
```yaml
environment:
  POSTGRES_PASSWORD: your_password
  POSTGRES_SHARED_BUFFERS: 256MB
  POSTGRES_EFFECTIVE_CACHE_SIZE: 1GB
```

### Проблема 5: Ошибка "Database is locked"

**Решение:**
```powershell
docker-compose restart postgres
docker-compose restart synapse
```

### Проблема 6: Файлы конфигурации не сохраняются

**Причина:** Docker не имеет доступ к папке.

**Решение:**
1. Docker Desktop → Settings → Resources → File Sharing
2. Добавьте `C:\Matrix` в список разрешенных папок
3. Перезапустите Docker Desktop

---

## 9. Рекомендации по производительности

### 9.1 Настройки Windows

**Отключите индексирование для папок Matrix:**
1. Правой кнопкой на `C:\Matrix`
2. Properties → снимите галочку "Allow files in this folder to have contents indexed"

### 9.2 Настройки Docker

**Переместите Docker данные на SSD** если у вас HDD:
1. Docker Desktop → Settings → Resources → Advanced
2. Измените Disk image location на SSD

### 9.3 Оптимизация homeserver.yaml

Добавьте в конец файла:

```yaml
# Кэширование
caches:
  global_factor: 2.0

# Ограничение размера медиа
max_upload_size: 50M

# Производительность БД
database:
  name: psycopg2
  args:
    # ... (существующие настройки)
    cp_min: 5
    cp_max: 10
    keepalives_idle: 10
    keepalives_interval: 10
    keepalives_count: 3
```

---

## 10. Дополнительные команды

### Полная очистка и переустановка

**⚠️ УДАЛИТ ВСЕ ДАННЫЕ:**
```powershell
Set-Location C:\Matrix
docker-compose down -v
Remove-Item -Path "synapse-data" -Recurse -Force
Remove-Item -Path "postgres-data" -Recurse -Force
```

Затем начните заново с шага 2.1.

### Обновление Synapse до новой версии

```powershell
Set-Location C:\Matrix

# Скачиваем новую версию образов
docker-compose pull

# Пересоздаем контейнеры
docker-compose up -d
```

### Проверка версии Synapse

```powershell
docker exec matrix-synapse python -m synapse.app.homeserver --version
```

### Выполнение команд внутри контейнера

```powershell
# Получить shell доступ к Synapse
docker exec -it matrix-synapse /bin/bash

# Получить shell доступ к PostgreSQL
docker exec -it matrix-postgres psql -U synapse
```

---

## 11. Что дальше?

### Рекомендуемые следующие шаги:

1. **Настройте доменное имя** вместо `localhost` для доступа из интернета
2. **Включите HTTPS** с Let's Encrypt для безопасности
3. **Настройте federation** для общения с другими Matrix серверами
4. **Отключите открытую регистрацию** после создания учетных записей
5. **Настройте регулярные бэкапы** с помощью Task Scheduler
6. **Изучите администрирование** через Matrix Admin API

### Полезные ссылки:

- Официальная документация Synapse: https://matrix-org.github.io/synapse/latest/
- Element клиенты: https://element.io/download
- Matrix сообщество: https://matrix.to/#/#matrix:matrix.org

---

## Краткая памятка команд

```powershell
# Перейти в папку Matrix
Set-Location C:\Matrix

# Запустить сервер
docker-compose up -d

# Остановить сервер
docker-compose stop

# Перезапустить сервер
docker-compose restart

# Просмотр логов
docker-compose logs -f synapse

# Статус контейнеров
docker-compose ps

# Создать пользователя
docker exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml

# Создать администратора
docker exec -it matrix-synapse register_new_matrix_user http://localhost:8008 -c /data/homeserver.yaml -a
```

---

**Готово! Ваш Matrix homeserver работает на Windows с Docker! 🚀**

Если возникли проблемы, проверьте логи командой `docker-compose logs synapse` и раздел "Решение типичных проблем".
