# AmneziaWG Web — Ivan UI

Русифицированная версия `amneziawg-web` на основе проекта `wiresock/amneziawg-install`.

Репозиторий форка:

`https://github.com/ivan12000user/amneziawg-install`

Рабочая ветка:

`ivan-ui`

Проверенная production-версия:

`ivan-ui-0.1.20-production`

Версия веб-панели:

`0.1.20`

---

## Что добавлено в Ivan UI

По сравнению с upstream `amneziawg-web 0.1.20`:

- полностью русифицирован основной веб-интерфейс;
- светлая и тёмная темы;
- выбранная тема сохраняется в браузере;
- VPN IP клиента показывается отдельной колонкой;
- ручная проверка доступности клиента по ICMP ping;
- кнопка `Проверить` для отдельного клиента;
- кнопка `Проверить все`;
- сохранены штатные статусы Connection/handshake;
- сохранён учёт RX/TX;
- сохранено управление пользователями, сроками действия, QR-кодами и конфигурациями;
- сохранены штатные механизмы авторизации, базы SQLite и systemd.

Проверка ping запускается только вручную. Она не создаёт постоянный фоновый VPN-трафик и не подменяет штатный статус соединения по handshake.

---

# 1. Требования

Поддерживается Linux на `x86_64` и `aarch64`.

Для установки нужны:

- доступ `root` или `sudo`;
- `git`;
- рабочий интернет на время установки пакетов и Rust, если они ещё не установлены;
- AmneziaWG для работы VPN;
- Rust нужен только при сборке веб-панели из исходников;
- для публичного доступа к панели рекомендуется nginx или Caddy с HTTPS.

Отдельный сервер БД, Redis или Docker для `amneziawg-web` не требуются.

Если Rust отсутствует, установщик панели может установить его автоматически параметром `--install-rust`.

---

# 2. Установка на новый сервер

## 2.1. Установить Git

Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y git ca-certificates curl
```

## 2.2. Клонировать Ivan UI

```bash
git clone -b ivan-ui https://github.com/ivan12000user/amneziawg-install.git
cd amneziawg-install
```

Если нужна именно проверенная production-версия, а не последняя версия ветки:

```bash
git fetch --tags
git checkout ivan-ui-0.1.20-production
```

## 2.3. Установить AmneziaWG

Если AmneziaWG на сервере ещё не установлен:

```bash
sudo ./amneziawg-install.sh
```

Следуйте интерактивному мастеру.

После установки проверьте:

```bash
systemctl status awg-quick@awg0.service --no-pager
/usr/bin/awg show
```

Сервис `awg-quick@awg0.service` должен быть активен.

## 2.4. Установить веб-панель

Если Rust на сервере может отсутствовать:

```bash
sudo ./amneziawg-web.sh install --install-rust
```

Если Rust уже установлен:

```bash
sudo ./amneziawg-web.sh install
```

Установщик собирает панель из исходников текущего checkout репозитория и создаёт необходимые каталоги, пользователя службы, конфигурацию и systemd service.

Во время интерактивной установки задайте логин и пароль администратора.

---

# 3. Установка Ivan UI на сервер, где AmneziaWG уже работает

Не переустанавливайте VPN без необходимости.

```bash
git clone -b ivan-ui https://github.com/ivan12000user/amneziawg-install.git
cd amneziawg-install
git fetch --tags
git checkout ivan-ui-0.1.20-production
sudo ./amneziawg-web.sh install --install-rust
```

После установки:

```bash
systemctl is-active amneziawg-web.service
systemctl is-active awg-quick@awg0.service
/usr/local/bin/amneziawg-web --version
```

Ожидается:

```text
active
active
amneziawg-web 0.1.20
```

---

# 4. Где находится веб-панель

По умолчанию панель слушает только localhost:

```text
127.0.0.1:8080
```

Проверка:

```bash
ss -lntp | grep ':8080'
curl -I http://127.0.0.1:8080/
```

При включённой авторизации ответ может быть редиректом на страницу входа. Это нормально.

Не рекомендуется открывать порт `8080` напрямую в Интернет.

---

# 5. Доступ к панели через SSH-туннель

На Windows, Linux или macOS:

```bash
ssh -L 8080:127.0.0.1:8080 root@SERVER_IP
```

Окно SSH нужно оставить открытым.

После этого в браузере:

```text
http://127.0.0.1:8080/
```

Если локальный порт `8080` занят:

```bash
ssh -L 18080:127.0.0.1:8080 root@SERVER_IP
```

Тогда открыть:

```text
http://127.0.0.1:18080/
```

---

# 6. Публичный доступ через HTTPS

Для постоянного удалённого доступа рекомендуется оставлять `amneziawg-web` на `127.0.0.1:8080`, а наружу публиковать только reverse proxy с TLS.

Пример nginx:

```nginx
server {
    listen 443 ssl;
    server_name vpn.example.com;

    ssl_certificate     /etc/letsencrypt/live/vpn.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/vpn.example.com/privkey.pem;

    location / {
        proxy_pass http://127.0.0.1:8080;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
}
```

После изменения nginx:

```bash
sudo nginx -t
sudo systemctl reload nginx
```

Для production с HTTPS в конфигурации панели должна использоваться защищённая cookie-сессия (`AUTH_SECURE_COOKIE=true`).

---

# 7. Основные файлы

Бинарник панели:

```text
/usr/local/bin/amneziawg-web
```

Привилегированный helper:

```text
/usr/local/libexec/amneziawg-web-privileged
```

Настройки панели:

```text
/etc/amneziawg-web/
```

Файл переменных окружения:

```text
/etc/amneziawg-web/env.conf
```

База данных:

```text
/var/lib/amneziawg-web/awg-web.db
```

Клиентские конфигурации AmneziaWG:

```text
/etc/amnezia/amneziawg/clients/
```

Systemd service панели:

```text
amneziawg-web.service
```

VPN service:

```text
awg-quick@awg0.service
```

---

# 8. Полезные команды

Статус панели:

```bash
systemctl status amneziawg-web.service --no-pager
```

Статус VPN:

```bash
systemctl status awg-quick@awg0.service --no-pager
```

Версия панели:

```bash
/usr/local/bin/amneziawg-web --version
```

Журнал панели:

```bash
journalctl -u amneziawg-web.service -n 100 --no-pager
```

Журнал в реальном времени:

```bash
journalctl -fu amneziawg-web.service
```

Перезапуск только панели:

```bash
sudo systemctl restart amneziawg-web.service
```

Проверка AWG:

```bash
sudo /usr/bin/awg show
```

---

# 9. Обновление Ivan UI

Перед обновлением обязательно сделайте резервную копию.

Перейдите в репозиторий:

```bash
cd /path/to/amneziawg-install
```

Перейдите на рабочую ветку:

```bash
git checkout ivan-ui
```

Получите изменения:

```bash
git pull --ff-only origin ivan-ui
```

Проверьте код:

```bash
cd amneziawg-web
cargo test --locked -- --test-threads=1
cd ..
```

Обновите панель:

```bash
sudo ./amneziawg-web.sh upgrade
```

После обновления:

```bash
systemctl is-active amneziawg-web.service
systemctl is-active awg-quick@awg0.service
/usr/local/bin/amneziawg-web --version
```

---

# 10. Обновление строго до проверенного production-тега

```bash
cd /path/to/amneziawg-install
git fetch --tags origin
git checkout ivan-ui-0.1.20-production
sudo ./amneziawg-web.sh upgrade
```

Это предпочтительный вариант, если требуется воспроизводимая стабильная версия.

Чтобы вернуться к разработческой ветке:

```bash
git checkout ivan-ui
git pull --ff-only origin ivan-ui
```

---

# 11. Резервная копия перед обновлением

Пример:

```bash
BACKUP=/root/awg-web-backup_$(date +%Y%m%d_%H%M%S)
sudo mkdir -p "$BACKUP"

sudo cp -a /usr/local/bin/amneziawg-web "$BACKUP/"
sudo cp -a /usr/local/libexec/amneziawg-web-privileged "$BACKUP/" 2>/dev/null || true
sudo cp -a /etc/amneziawg-web "$BACKUP/" 2>/dev/null || true
```

Перед копированием БД остановите только веб-панель. Сам VPN останавливать не требуется:

```bash
sudo systemctl stop amneziawg-web.service

sudo cp -a /var/lib/amneziawg-web/awg-web.db "$BACKUP/"
sudo cp -a /var/lib/amneziawg-web/awg-web.db-wal "$BACKUP/" 2>/dev/null || true
sudo cp -a /var/lib/amneziawg-web/awg-web.db-shm "$BACKUP/" 2>/dev/null || true

sudo systemctl start amneziawg-web.service
```

Проверьте:

```bash
systemctl is-active amneziawg-web.service
systemctl is-active awg-quick@awg0.service
```

---

# 12. Ручной откат панели

Если новая версия панели не запускается, VPN `awg0` без необходимости не трогайте.

Предположим, резервная копия находится в `$BACKUP`:

```bash
sudo systemctl stop amneziawg-web.service

sudo cp -a "$BACKUP/amneziawg-web" /usr/local/bin/amneziawg-web

if [ -f "$BACKUP/awg-web.db" ]; then
    sudo cp -a "$BACKUP/awg-web.db" /var/lib/amneziawg-web/awg-web.db
    sudo rm -f /var/lib/amneziawg-web/awg-web.db-wal
    sudo rm -f /var/lib/amneziawg-web/awg-web.db-shm
fi

sudo systemctl start amneziawg-web.service
```

Проверка:

```bash
systemctl status amneziawg-web.service --no-pager
journalctl -u amneziawg-web.service -n 50 --no-pager
```

---

# 13. Ручная сборка из исходников

```bash
cd amneziawg-web

cargo fmt --check
cargo check --locked
cargo test --locked -- --test-threads=1
cargo build --release --locked
```

Готовый бинарник:

```text
amneziawg-web/target/release/amneziawg-web
```

Проверка:

```bash
./target/release/amneziawg-web --version
sha256sum ./target/release/amneziawg-web
```

---

# 14. Проверка после установки или обновления

```bash
echo '===== VERSION ====='
/usr/local/bin/amneziawg-web --version

echo '===== SERVICES ====='
systemctl is-active amneziawg-web.service
systemctl is-active awg-quick@awg0.service

echo '===== LISTEN ====='
ss -lntp | grep ':8080'

echo '===== HTTP ====='
curl -sS -o /dev/null -w 'HTTP=%{http_code}\n' http://127.0.0.1:8080/

echo '===== AWG ====='
sudo /usr/bin/awg show

echo '===== LOG ====='
journalctl -u amneziawg-web.service -n 40 --no-pager
```

Нормально, если локальный HTTP при включённой авторизации возвращает redirect `302` или `303`.

---

# 15. Безопасность

Никогда не добавляйте в Git:

- приватные ключи AmneziaWG;
- клиентские `.conf`;
- пароли;
- `AUTH_PASSWORD_HASH`, если не хотите публиковать его;
- API token;
- содержимое `/etc/amnezia/amneziawg/clients/`;
- базу `/var/lib/amneziawg-web/awg-web.db`;
- резервные копии production.

Панель рекомендуется держать на `127.0.0.1:8080`.

Для доступа из Интернета используйте HTTPS reverse proxy и включённую авторизацию.

---

# 16. Git: рекомендуемая схема

Для этого форка:

```text
origin   -> https://github.com/ivan12000user/amneziawg-install.git
upstream -> https://github.com/wiresock/amneziawg-install.git
```

Рабочая ветка:

```text
ivan-ui
```

Получить изменения upstream без риска случайного push:

```bash
git fetch upstream
```

Рекомендуется отключить push в upstream:

```bash
git remote set-url --push upstream DISABLED
```

Проверка:

```bash
git remote -v
```

---

# 17. Production tag

Текущая проверенная точка:

```text
ivan-ui-0.1.20-production
```

Для сервера, где важна стабильность, рекомендуется ставить именно tag, а не автоматически следовать за последним commit ветки.

---

# 18. Upstream

Оригинальный проект:

`https://github.com/wiresock/amneziawg-install`

Документация upstream по веб-панели находится в:

```text
amneziawg-web/docs/INSTALL.md
amneziawg-web/docs/DEPLOYMENT.md
```

Ivan UI сохраняет upstream как основу и добавляет русификацию, тёмную тему, отдельный VPN IP и ручную проверку доступности клиентов.
