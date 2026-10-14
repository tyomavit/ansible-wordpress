# ansible-wordpress-stack: WordPress на трёх VPS (Nginx → Apache → MySQL)

Учебный проект по Ansible. Он поднимает шаблон сайта на WordPress с нуля на трёх чистых серверах Ubuntu:

```
Браузер ──HTTPS :443──▶ [proxy] Nginx ──HTTP :8080──▶ [web] Apache + PHP + WordPress ──:3306──▶ [db] MySQL
                          └─ сертификат Let's Encrypt (certbot)
```

Раньше то же самое делали три bash-скрипта. Ansible заменяет их и добавляет главное: **идемпотентность** (повторный запуск ничего не ломает и не меняет без причины).

## Что делает каждый плейбук

| Файл | Хосты | Что делает |
|---|---|---|
| `baseline.yml` | все три | Ставит fail2ban, unattended-upgrades, ufw. Отключает вход по SSH по паролю. Задаёт базовые правила UFW: входящие запрещены, исходящие разрешены, SSH открыт |
| `db.yml` | db | Ставит MySQL, применяет свой конфиг, создаёт базу и двух пользователей, открывает порт 3306 только для web |
| `web.yml` | web | Переводит Apache на порт 8080, ставит PHP, скачивает WordPress, собирает `wp-config.php` из шаблона, открывает порт 8080 только для proxy |
| `proxy.yml` | proxy | Ставит Nginx и certbot, получает сертификат Let's Encrypt, настраивает HTTPS и проксирование на Apache, настраивает автопродление |
| `site.yml` | — | Запускает все четыре по порядку: baseline → db → web → proxy |

**Почему такой порядок.** WordPress на web нужна готовая база, а proxy нужен готовый web. Baseline идёт первым, потому что фаервол и SSH настраиваются до всего остального.

## Структура проекта

```
ansible-wordpress-stack/
├── ansible.cfg              # inventory и файл пароля vault
├── inventory.ini            # реальные адреса серверов (в .gitignore, в Git не попадает)
├── inventory.ini.example    # то же самое с адресами-заглушками (в Git есть)
├── requirements.yml         # нужные коллекции Ansible
├── group_vars/all/
│   ├── all.yml              # открытые переменные
│   └── vault.yml            # секреты (ЗАШИФРОВАН ansible-vault)
├── templates/               # шаблоны конфигов (.j2)
│   ├── proxy_sites_conf.j2  # Nginx: HTTPS и reverse proxy
│   ├── web_sites_conf.j2    # Apache: виртуальный хост
│   ├── web_ports_conf.j2    # Apache: слушаем только 8080
│   └── web_php.j2           # wp-config.php
├── baseline.yml  db.yml  web.yml  proxy.yml  site.yml
├── docs/                    # памятки (см. раздел «Памятки» ниже)
├── .gitignore               # .vault_pass, inventory.ini, *.tar.gz, *.retry
└── .vault_pass              # пароль от vault (в .gitignore, в Git не попадает)
```

## Требования

**Управляющая машина** (откуда запускается Ansible, Linux на Ubuntu 26.04):

```bash
sudo apt update && sudo apt install -y pipx
pipx ensurepath
pipx install --include-deps ansible
ansible-galaxy collection install -r requirements.yml
```

Ansible ставится из терминала, а коллекции из `requirements.yml`. Коллекции должны быть на управляющей машине: модули ищутся до начала выполнения задач. Проверить: `ansible-galaxy collection list | grep -E "mysql|general|posix"`.

**Три VPS** с чистой Ubuntu (проверено на 26.04 с MySQL 8.4), доступ по SSH-ключу под root.

**Домен**: A-записи `домен` и `www.домен` должны указывать на IP сервера proxy **до** запуска `proxy.yml`. Иначе certbot не пройдёт проверку.

## Первый запуск

1. **Ключ на серверы.** Baseline отключает вход по паролю, поэтому сначала нужно убедиться, что вход по ключу работает: `ssh -o PasswordAuthentication=no root@IP "echo ok"` (для каждого сервера). Под рукой лучше держать открытые сессии в веб-консоли хостинга или  терминале как запасной вариант.
2. **Адреса серверов.** Нужно скопировать шаблон и вписать свои IP:
   ```bash
   cp inventory.ini.example inventory.ini
   ```
3. **Секреты.** Нужно создать файл пароля и сам vault:
   ```bash
   echo "придумай_пароль" > .vault_pass && chmod 600 .vault_pass
   ansible-vault create group_vars/all/vault.yml
   ```
   В vault должны быть переменные (их использует `all.yml`):
   ```yaml
   vault_web_ip_address: "..."            # IP сервера web
   vault_db_ip_address: "..."             # IP сервера db
   vault_proxy_ip_address: "..."          # IP сервера proxy
   vault_db_user_password: "..."          # пароль локального пользователя MySQL
   vault_db_remote_user_password: "..."   # пароль пользователя для WordPress
   vault_ssl_email: "..."                 # e-mail для Let's Encrypt
   vault_db_password_salt: "..."          # ровно 20 символов: openssl rand -hex 10
   vault_wp_salts:                        # ключи и соли WordPress
     AUTH_KEY: "..."                      # по 64 символа: openssl rand -hex 32
     SECURE_AUTH_KEY: "..."
     LOGGED_IN_KEY: "..."
     NONCE_KEY: "..."
     AUTH_SALT: "..."
     SECURE_AUTH_SALT: "..."
     LOGGED_IN_SALT: "..."
     NONCE_SALT: "..."
   ```
   Как править vault: `EDITOR=nano ansible-vault edit group_vars/all/vault.yml`. Пароль vault лучше хранить в менеджере паролей: без него `vault.yml` не расшифровать.
4. **Имя домена** нужно вписать в `group_vars/all/all.yml` (`fqdn_domain`, `domain_name`).
5. **Проверка синтаксиса и запуск:**
   ```bash
   ansible-playbook site.yml --syntax-check
   ansible-playbook site.yml
   ```
6. Запуск `site.yml` ещё раз. Ожидаем `changed=0` на всех хостах: это проверка идемпотентности.
7. Открываем сайт в браузере и пройдим мастер установки WordPress (плейбуки готовят сервер, но саму установку WordPress не делают).

> **Тест на чистых серверах и лимит Let's Encrypt.** На настоящие сертификаты действует лимит (5 штук в неделю на один набор доменов). Для тестовых прогонов временно лучше добавлять `--staging` в задачу certbot в `proxy.yml`. Потом нужно убрать флаг и удалить тестовый сертификат: `certbot delete --cert-name ДОМЕН`.

## Запуск по частям

```bash
ansible-playbook site.yml --limit db        # только сервер БД
ansible-playbook web.yml --syntax-check     # проверка одного плейбука
ansible-playbook proxy.yml -v               # подробный вывод
```

## Как проверить результат

| Где | Команда | Ожидаем |
|---|---|---|
| db | `ss -tlnp \| grep -E '3306\|33060'` | 3306 на `0.0.0.0`, 33060 на `127.0.0.1` |
| db | `mysql -e "SELECT user,host,plugin FROM mysql.user"` | пользователи с `caching_sha2_password` |
| web | `ss -tlnp \| grep apache2` | только порт 8080 |
| web | `mysql -h IP_DB -u пользователь -p'пароль от db_remote_user' база -e "SELECT 1;"` | таблица с единицей (связь web → db) |
| proxy | `curl -sI https://ДОМЕН \| head -5` | ответ WordPress, заголовок `Strict-Transport-Security` |
| proxy | `certbot renew --dry-run` | все симулированные продления успешны, Nginx после теста `active` |
| proxy | `systemctl list-timers \| grep certbot` | таймер автопродления включён |

## Почему сделано именно так

- **Коллекция `ansible.mysql`.** Коллекция `community.mysql` переименована в `ansible.mysql`, в новых плейбуках используем новое имя. Для связи с MySQL на сервере нужна библиотека `python3-pymysql` (старый `mysqlclient` признан устаревшим).
- **Пароли MySQL через `plugin` + `salt`.** В MySQL 8.4 плагин `mysql_native_password` отключён по умолчанию, а параметр `password` в модуле работает только с ним (ошибка `Plugin 'mysql_native_password' is not loaded`). Поэтому пароль задаётся связкой `plugin: caching_sha2_password`, `plugin_auth_string` и `salt`.
- **`lock_timeout: 300` у всех задач apt.** Сразу после создания VPS `cloud-init` и `unattended-upgrades` держат блокировку apt, и задача падает. Параметр заставляет подождать.
- **Конфиг MySQL отдельным файлом.** `mysqld_custom.cnf` накладывается поверх системного, не заменяя его. Файлы читаются по алфавиту, последний побеждает, поэтому имя выбрано так, чтобы идти после `mysqld.cnf`.
- **Apache слушает только 8080 без привязки к IP.** Доступ ограничивает UFW (порт открыт только для IP прокси). Привязка к публичному IP ломается, если он не висит на интерфейсе сервера.
- **UFW собран в baseline.** Базовые правила заданы один раз. В остальных плейбуках только правила своего сервиса. Сначала правило для SSH, потом включение файрвола, иначе можно отрезать себе доступ.
- **Сертификат в `block` с `always`.** Certbot в режиме standalone требует свободный порт 80, поэтому Nginx на время останавливается. Секция `always` включает его обратно при любом исходе, и сайт не остаётся лежать при ошибке certbot.
- **Проверка сертификата с `follow: true`.** Иначе битая символическая ссылка считалась бы существующим сертификатом.
- **HSTS с `max-age=86400`.** Браузер запоминает HSTS надолго. Лучше начать с суток и увеличивать до года, когда всё стабильно.
- **Проверки внутри плейбука.** `apache2ctl configtest`, ответ Apache на `127.0.0.1:8080`, `nginx -t`. Ошибка видна сразу, а не при обращении к сайту.

### Автопродление сертификата: как это работает

Отдельной задачи «включить автопродление» в плейбуке нет, оно держится на трёх вещах:

1. Пакет `certbot` из apt сам ставит и включает **таймер** `certbot.timer`. Он дважды в сутки запускает `certbot renew`.
2. Команда `certbot certonly` при выпуске сохраняет **конфиг продления** в `/etc/letsencrypt/renewal/`.
3. **Хуки** из `proxy.yml` (`/etc/letsencrypt/renewal-hooks/pre` и `post`) освобождают порт 80: перед продлением Nginx останавливается, после попытки запускается (хук `post` срабатывает даже при неудаче, поэтому сайт не остаётся лежать).

Certbot продлевает сертификат, когда до окончания остаётся меньше 30 дней. Если продление не требуется, хуки не запускаются вообще. Команда `certbot renew --dry-run` прогоняет всю цепочку для проверки.

### Две разные «соли»

В проекте есть две соли, и они для разного:

| | `db_password_salt` | `wp_salts` |
|---|---|---|
| Для чего | Хеширование пароля пользователя MySQL | Подпись cookie и токенов WordPress |
| Кто использует | Модуль `ansible.mysql.mysql_user` | Сам WordPress (`wp-config.php`) |
| Сколько | Одна строка, ровно 20 символов | Восемь ключей по 64 символа |
| Зачем | Чтобы Ansible мог сравнить пароль и не менял его при каждом запуске | Чтобы сессии были надёжно защищены |

Ключи WordPress должны быть **уникальными для каждого сайта**. Тот же сайт на новых серверах можно разворачивать с теми же ключами, для другого сайта нужно генерировать новые.

## Бэкап сертификатов

Чтобы при пересоздании сервера не тратить лимит Let's Encrypt, лучше сохранить копию `/etc/letsencrypt`. Команды я выполнял в PowerShell:

```powershell
ssh root@IP_PROXY "tar czf /root/le-backup.tar.gz /etc/letsencrypt"
scp root@IP_PROXY:/root/le-backup.tar.gz .
tar tzf le-backup.tar.gz | Select-Object -First 5
ssh root@IP_PROXY "rm /root/le-backup.tar.gz"
```

Хранить архив лучше вне облачных папок и не класть его в Git: в нём приватный ключ.

## Памятки

- [`docs/memo-git-push.md`](docs/memo-git-push.md): как на новом VPS залить проект в GitHub (ключи, репозиторий, первый коммит).
- [`docs/memo-deploy.md`](docs/memo-deploy.md): как на новом VPS получить проект из репозитория и запустить `site.yml`.

## Безопасность: что нельзя коммитить

- `.vault_pass` (в `.gitignore`);
- `inventory.ini` с реальными адресами (в `.gitignore`, в репозитории только `inventory.ini.example`);
- `group_vars/all/vault.yml` только в зашифрованном виде (первая строка файла должна быть `$ANSIBLE_VAULT;1.1;AES256`);
- архивы `/etc/letsencrypt` и любые приватные ключи.

Если секрет попал в коммит, он остаётся в истории Git. Его нужно считать скомпрометированным и сменить.

## Частые проблемы

| Симптом | Причина и решение |
|---|---|
| `Could not get lock` у apt | Сервер только создан, фоновое обновление. Решает `lock_timeout` |
| Certbot: `Connection refused` или ошибка проверки | DNS ещё не указывает на proxy, либо закрыт порт 80 |
| Nginx не стартует после certbot | Нет файлов сертификата. Проверить `certbot certificates` и `nginx -t` |
| `Plugin 'mysql_native_password' is not loaded` | Пароль задан через `password`. Использовать `plugin` + `plugin_auth_string` + `salt` |
| `couldn't resolve module` | Не установлены коллекции: `ansible-galaxy collection install -r requirements.yml` |
| `Host key verification failed` | Ключ сервера изменился после пересоздания VPS: удалить старую запись из `~/.ssh/known_hosts` |
| `Permission denied (publickey)` при `git push` | Ключ не добавлен на GitHub или не указан в `~/.ssh/config`. См. `docs/memo-git-push.md` |

## Источники

- [Ansible tutorial: first playbook on VPS (SSD Nodes)](https://www.ssdnodes.com/learn/lang/ru/ansible-tutorial-first-playbook-vps): первоначальная настройка проекта.
- [How to Set Up a High-Performance WordPress Site on Cloud (Atlantic.Net)](https://www.atlantic.net/vps-hosting/how-to-set-up-a-high-performance-wordpress-site-on-cloud/): та же схема вручную (Apache на порту 8080, Nginx как reverse proxy, MySQL, WordPress).
- [Let's Encrypt hooks use cases (Advanced Web Machinery)](https://advancedweb.hu/lets-encrypt-hooks-use-cases/): как работают `pre-hook` и `post-hook` certbot, режим standalone и продление.
- [Ansible: General tips (раздел «Keep vaulted variables safely visible»)](https://docs.ansible.com/ansible/latest/playbooks_best_practices.html): схема `vars` + `vault` с префиксом `vault_`, на которой построен `group_vars/all/`.
- [ansible.mysql.mysql_user](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mysql/mysql_user_module.html): документация модуля (параметры `plugin`, `plugin_auth_string`, `salt`).
