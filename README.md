# ansible_learning: WordPress на трёх VPS (Nginx → Apache → MySQL)

Учебный проект по Ansible. Он поднимает сайт на WordPress с нуля на трёх чистых серверах Ubuntu:

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
| `proxy.yml` | proxy | Ставит Nginx и certbot, получает сертификат Let's Encrypt, настраивает HTTPS и проксирование на Apache, включает автопродление |
| `site.yml` | — | Запускает все четыре по порядку: baseline → db → web → proxy |

**Почему такой порядок.** WordPress на web нужна готовая база, а proxy нужен готовый web. Baseline идёт первым, потому что фаервол и SSH настраиваются до всего остального.

## Структура проекта

```
ansible_learning/
├── ansible.cfg              # inventory и файл пароля vault
├── inventory.ini            # три сервера: proxy, web, db
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
└── .vault_pass              # пароль от vault (В .gitignore, в Git не попадает!)
```

## Требования

**Управляющая машина** (откуда запускается Ansible, Linux):

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

1. **Ключ на серверы.** Baseline отключает вход по паролю, поэтому сначала убедитесь, что вход по ключу работает: `ssh -o PasswordAuthentication=no root@IP "echo ok"` (для каждого сервера). Под рукой держите веб-консоль хостинга как запасной вход.
2. **Адреса серверов** впишите в `inventory.ini`.
3. **Секреты.** Создайте файл пароля и сам vault:
   ```bash
   echo "придумайте_пароль" > .vault_pass && chmod 600 .vault_pass
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
   Править vault: `ansible-vault edit group_vars/all/vault.yml`.
4. **Имя домена** впишите в `group_vars/all/all.yml` (`fqdn_domain`, `domain_name`).
5. **Проверка синтаксиса и запуск:**
   ```bash
   ansible-playbook site.yml --syntax-check
   ansible-playbook site.yml
   ```
6. Запустите `site.yml` ещё раз. Ожидаем `changed=0` на всех хостах: это проверка идемпотентности.

> **Тест на чистых серверах и лимит Let's Encrypt.** На настоящие сертификаты действует лимит (5 штук в неделю на один набор доменов). Для тестовых прогонов временно добавьте `--staging` в задачу certbot в `proxy.yml`. Потом уберите флаг и удалите тестовый сертификат: `certbot delete --cert-name ДОМЕН`. Копию рабочих сертификатов можно сохранить: `tar czf le-backup.tar.gz /etc/letsencrypt` (в Git её класть нельзя, там приватный ключ).

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
| web | `mysql -h IP_DB -u пользователь -p база -e "SELECT 1;"` | таблица с единицей (связь web → db) |
| proxy | `curl -sI https://ДОМЕН \| head -5` | ответ WordPress, заголовок `Strict-Transport-Security` |
| proxy | `certbot renew --dry-run` | все симулированные продления успешны, Nginx после теста `active` |
| proxy | `systemctl list-timers \| grep certbot` | таймер автопродления включён |

## Почему сделано именно так

- **Коллекция `ansible.mysql`.** Коллекция `community.mysql` переименована в `ansible.mysql`, в новых плейбуках используем новое имя. Для связи с MySQL на сервере нужна библиотека `python3-pymysql` (старый `mysqlclient` признан устаревшим).
- **Пароли MySQL через `plugin` + `salt`.** В MySQL 8.4 плагин `mysql_native_password` отключён по умолчанию, а параметр `password` в модуле работает только с ним (ошибка `Plugin 'mysql_native_password' is not loaded`). Поэтому пароль задаётся связкой `plugin: caching_sha2_password`, `plugin_auth_string` и `salt` (ровно 20 символов, нужна для идемпотентности).
- **`lock_timeout: 300` у всех задач apt.** Сразу после создания VPS `cloud-init` и `unattended-upgrades` держат блокировку apt, и задача падает. Параметр заставляет подождать. Это причина эффекта «работает через раз».
- **Конфиг MySQL отдельным файлом.** `mysqld_custom.cnf` накладывается поверх системного, не заменяя его. Файлы читаются по алфавиту, последний побеждает, поэтому имя выбрано так, чтобы идти после `mysqld.cnf`.
- **Apache слушает только 8080 без привязки к IP.** Доступ ограничивает UFW (порт открыт только для IP прокси). Привязка к публичному IP ломается, если он не висит на интерфейсе сервера.
- **UFW собран в baseline.** Базовые правила заданы один раз. В остальных плейбуках только правила своего сервиса. Сначала правило для SSH, потом включение фаервола, иначе можно отрезать себе доступ.
- **Сертификат в `block` с `always`.** Certbot в режиме standalone требует свободный порт 80, поэтому Nginx на время останавливается. Секция `always` включает его обратно при любом исходе, и сайт не остаётся лежать при ошибке certbot.
- **Проверка сертификата с `follow: true`.** Иначе битая символическая ссылка считалась бы существующим сертификатом.
- **Хуки автопродления** (`/etc/letsencrypt/renewal-hooks/pre` и `post`). Автопродление в режиме standalone тоже занимает порт 80, поэтому перед ним Nginx останавливается, после запускается.
- **HSTS с `max-age=86400`.** Браузер запоминает HSTS надолго. Начинаем с суток и увеличиваем до года, когда всё стабильно.
- **Ключи и соли WordPress в vault**, `wp-config.php` с правами `0640` (в нём пароль базы).
- **Проверки внутри плейбука.** `apache2ctl configtest`, ответ Apache на `127.0.0.1:8080`, `nginx -t`. Ошибка видна сразу, а не при обращении к сайту.

## Частые проблемы

| Симптом | Причина и решение |
|---|---|
| `Could not get lock` у apt | Сервер только создан, фоновое обновление. Решает `lock_timeout` |
| Certbot: `Connection refused` или ошибка проверки | DNS ещё не указывает на proxy, либо закрыт порт 80 |
| Nginx не стартует после certbot | Нет файлов сертификата. Проверить `certbot certificates` и `nginx -t` |
| `Plugin 'mysql_native_password' is not loaded` | Пароль задан через `password`. Использовать `plugin` + `plugin_auth_string` + `salt` |
| `couldn't resolve module` | Не установлены коллекции: `ansible-galaxy collection install -r requirements.yml` |
| `Host key verification failed` | Ключ сервера изменился после пересоздания VPS: удалить старую запись из `~/.ssh/known_hosts` |

## Безопасность: что нельзя коммитить

- `.vault_pass` (в `.gitignore`);
- `group_vars/all/vault.yml` только в зашифрованном виде (первая строка файла должна быть `$ANSIBLE_VAULT;1.1;AES256`);
- архивы `/etc/letsencrypt` и любые приватные ключи.

Если секрет попал в коммит, он остаётся в истории Git. Его нужно считать скомпрометированным и сменить.

## Источники

- Первоначальная настройка по статье: [Ansible tutorial: first playbook on VPS (SSD Nodes)](https://www.ssdnodes.com/learn/lang/ru/ansible-tutorial-first-playbook-vps)
- Документация модуля: [ansible.mysql.mysql_user](https://docs.ansible.com/projects/ansible/latest/collections/ansible/mysql/mysql_user_module.html)