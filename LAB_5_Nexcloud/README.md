# Лабораторная работа по DevOps: Nextcloud в Docker

Полное руководство по развёртыванию Nextcloud в контейнеризированной среде на Ubuntu.

---

## 1. Подготовка виртуальных машин

Создайте **2 виртуальные машины** с Ubuntu (клиент и сервер):

- **Сервер** — на этой машине будет установлен Docker и развёрнут Nextcloud
- **Клиент** — будет использоваться для доступа к Nextcloud через браузер

Рекомендуемые параметры ВМ:
- **ОС:** Ubuntu 22.04 LTS или 24.04 LTS
- **RAM:** минимум 2 ГБ (лучше 4 ГБ для Nextcloud)
- **Диск:** минимум 20 ГБ
- **Сеть:** bridge или NAT для доступа к интернету

---

## 2. Установка Docker

Все команды выполняйте **на сервере.** Клиент служить исключительно для открытия web-панелей.

### 2.1 Обновление списка пакетов
```bash
sudo apt-get update
```
**Пояснение:** Обновляет индекс доступных пакетов из репозиториев Ubuntu.

### 2.2 Перезагрузка (после первого обновления)
```bash
reboot
```
**Пояснение:** Перезагрузка применяет последние обновления ядра и драйверов. После перезагрузки снова подключитесь к машине.
### 2.3 Установка необходимых пакетов
```bash
sudo apt-get install ca-certificates curl
```
**Пояснение:** Устанавливает сертификаты для HTTPS и утилиту curl для загрузки файлов. Требуется для добавления репозитория Docker.

### 2.4 Создание директории для ключей
```bash
sudo install -m 0755 -d /etc/apt/keyrings
```
**Пояснение:** Создаёт каталог `/etc/apt/keyrings` с правами 755. Здесь хранятся GPG-ключи репозиториев для проверки подлинности пакетов.

### 2.5 Загрузка GPG-ключа Docker
```bash
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
```
**Пояснение:** Скачивает официальный GPG-ключ Docker и сохраняет его в `docker.asc`. Ключ используется для проверки подписи пакетов Docker.

### 2.6 Установка прав на ключ
```bash
sudo chmod a+r /etc/apt/keyrings/docker.asc
```
**Пояснение:** Даёт права на чтение ключа всем пользователям (all read), что необходимо для работы apt.

### 2.7 Добавление репозитория Docker в источники Apt
```bash
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
**Пояснение:** Добавляет официальный репозиторий Docker в систему. `dpkg --print-architecture` определяет архитектуру (amd64, arm64 и т.д.), а `UBUNTU_CODENAME` — кодовое имя релиза (jammy, noble и т.д.).

### 2.8 Обновление списка пакетов
```bash
sudo apt-get update
```
**Пояснение:** Обновляет индекс пакетов с учётом нового репозитория Docker.

### 2.9 Установка Docker и плагинов
```bash
sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
**Пояснение:** Устанавливает Docker Engine, CLI, containerd (runtime для контейнеров), buildx (сборка образов) и плагин Docker Compose.

### 2.10 Установка Docker Compose (standalone)
```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
**Пояснение:** Скачивает бинарный файл Docker Compose для вашей ОС и архитектуры. `uname -s` — тип ОС (Linux), `uname -m` — архитектура (x86_64).

### 2.11 Права на выполнение Docker Compose
```bash
sudo chmod +x /usr/local/bin/docker-compose
```
**Пояснение:** Делает файл исполняемым, чтобы его можно было запускать как команду.

### 2.12 Создание симлинка (опционально)
```bash
sudo ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
**Пояснение:** Создаёт символическую ссылку в `/usr/bin`, чтобы команда `docker-compose` была доступна из любой директории.

### 2.13 Проверка версии Docker Compose
```bash
docker-compose --version
```
**Пояснение:** Проверяет, что Docker Compose установлен и выводит версию.

### 2.14 Добавление пользователя в группу docker
```bash
sudo usermod -aG docker $USER
```
**Пояснение:** Добавляет текущего пользователя в группу `docker`, чтобы запускать Docker без `sudo`. `$USER` — переменная с именем текущего пользователя.

### 2.15 Перезагрузка
```bash
reboot
```
**Пояснение:** Перезагрузка нужна, чтобы вступили в силу права группы `docker`. После перезагрузки Docker будет работать без sudo.

---

## 3. Развёртывание Nextcloud

Все шаги выполняются **на сервере**.

### 3.1 Создание файла docker-compose.yaml
```bash
nano docker-compose.yaml
```
**Пояснение:** Открывает текстовый редактор nano для создания файла конфигурации Docker Compose.

### 3.2 Содержимое docker-compose.yaml

Скопируйте и вставьте следующий конфиг (для nano: Ctrl+Shift+V, сохраните Ctrl+O, Enter, выход Ctrl+X):

```yaml
version: '3.8'  # версия формата Docker Compose

services:
  # ── PostgreSQL (база данных) ──
  db:
    image: postgres:15-alpine          # образ: PostgreSQL 15 на базе Alpine Linux
    restart: always                    # автоматический перезапуск при падении
    environment:                       # переменные окружения для настройки БД
      POSTGRES_DB: nextcloud           # имя создаваемой базы данных
      POSTGRES_USER: nextcloud         # имя пользователя БД
      POSTGRES_PASSWORD: nextcloud_password_123  # пароль пользователя
    volumes:                           # монтирование данных на хост
      - ./postgres:/var/lib/postgresql/data  # папка ./postgres на хосте → данные PostgreSQL
    networks:
      - nc-net                         # подключение к сети nc-net

  # ── Redis (кэш и очереди) ──
  redis:
    image: redis:7-alpine              # образ: Redis 7 на Alpine
    restart: always
    command: redis-server --appendonly yes   # включение persistence (сохранение на диск)
    volumes:
      - ./redis:/data                 # папка ./redis → данные Redis
    networks:
      - nc-net

  # ── Nextcloud (основное приложение) ──
  nextcloud:
    image: nextcloud:latest            # образ Nextcloud (последняя версия)
    restart: always
    ports:
      - "8080:80"                     # порт 8080 на хосте → порт 80 в контейнере
    environment:
      POSTGRES_HOST: db               # хост базы данных (имя сервиса db)
      POSTGRES_DB: nextcloud          # имя базы данных
      POSTGRES_USER: nextcloud        # пользователь БД
      POSTGRES_PASSWORD: nextcloud_password_123
      REDIS_HOST: redis               # хост Redis для кэширования
      NEXTCLOUD_ADMIN_USER: admin     # логин администратора (создаётся при первом запуске)
      NEXTCLOUD_ADMIN_PASSWORD: admin_password_123  # пароль администратора
    volumes:
      - ./nextcloud:/var/www/html     # папка ./nextcloud → веб-корень Nextcloud
    depends_on:                       # порядок запуска: сначала db и redis
      - db
      - redis
    networks:
      - nc-net

networks:
  nc-net:                             # изолированная сеть для контейнеров
    driver: bridge                    # драйвер bridge — контейнеры видят друг друга по имени
```

### 3.3 Запуск контейнеров
```bash
docker-compose up --build -d
```
**Пояснение:**
- `up` — поднимает все сервисы из docker-compose.yaml
- `--build` — пересобирает образы при необходимости
- `-d` — запуск в фоновом режиме (без вывода логов)

### 3.4 Ожидание запуска сервисов

Подождите 1–3 минуты, пока загрузятся все образы и запустятся контейнеры. Проверить статус можно командой:

```bash
docker-compose ps
```
**Пояснение:** Показывает состояние всех контейнеров. Все должны быть в статусе `Up`.

---

## 4. Открытие Nextcloud

1. Откройте браузер на **клиентской** машине.
2. Перейдите по адресу: `http://localhost:8080`.

---

## Полезные команды

```bash
# Просмотр логов
docker-compose logs -f

# Остановка сервисов
docker-compose down

# Запуск уже созданных контейнеров
docker-compose up -d

# Проверка работы Docker
docker run hello-world
```
