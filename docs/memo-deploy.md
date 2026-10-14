# Памятка: как на новом VPS получить проект из репозитория и запустить site.yml

Когда нужна: сменился управляющий VPS (или новая машина), а проект лежит на GitHub. Ниже весь путь: от пустого сервера до работающего сайта.

**Обозначения.** Всё в квадратных скобках это подстановка: `[имя аккаунта github]`, `[название репозитория]`, `[IP proxy]`. Сами скобки не вводить.

**Три роли серверов, не путайте:**

- **управляющий VPS**: сюда ставится Ansible, здесь лежит проект, отсюда всё запускается;
- **целевые VPS** (proxy, web, db): серверы, которые Ansible настраивает по SSH;
- **GitHub**: хранит код (главная копия).

## Что понадобится заранее

- три целевых VPS с чистой Ubuntu, их IP и доступ по SSH-ключу под root;
- домен с A-записями (`домен` и `www.домен`) на IP сервера proxy;
- пароль от vault (от файла `.vault_pass`; без него `vault.yml` не расшифровать, хранить лучше в менеджере паролей);
- доступ к GitHub-репозиторию (приватный репозиторий требует SSH-ключа, см. шаг 2).

## Шаг 1. Подготовить управляющий VPS

```bash
apt update && apt install -y git pipx
pipx ensurepath
```

После `pipx ensurepath` нужно закрыть терминал и открыть заново (или выполнить `source ~/.bashrc`), иначе команды `pipx` и `ansible` не найдутся.

## Шаг 2. Дать серверу доступ к GitHub

Нужен SSH-ключ этого сервера. Выполнить **шаги 2-5** из памятки `memo-git-push.md` (Git, ключ `github_ed25519`, добавление на GitHub, `~/.ssh/config`) и убедиться, что `ssh -T git@github.com` отвечает `Hi [имя аккаунта github]!`.

## Шаг 3. Скачать проект

```bash
cd ~
git clone git@github.com:[имя аккаунта github]/[название репозитория].git
cd [название репозитория]
ls -a
```

Должны быть `ansible.cfg`, плейбуки, `group_vars`, `templates`, `requirements.yml`, `inventory.ini.example`. Файлов `inventory.ini` и `.vault_pass` в репозитории нет: они в `.gitignore`, их нужно создать вручную (шаги 6 и 7).

## Шаг 4. Установить Ansible и коллекции

```bash
pipx install --include-deps ansible
ansible --version
ansible-galaxy collection install -r requirements.yml
ansible-galaxy collection list | grep -E "mysql|general|posix"
```

Должны отобразиться `ansible.mysql`, `ansible.posix`, `community.general`. Сообщение `Nothing to do. All requested collections are already installed` это нормально: коллекции пришли вместе с пакетом `ansible`.

## Шаг 5. Доступ с управляющего VPS на целевые

Управляющий сервер должен заходить на три целевых по SSH-ключу (это отдельный ключ, не тот, что для GitHub).

```bash
ls ~/.ssh/id_ed25519.pub
# создать, если ключа ещё нет
ssh-keygen -t ed25519 -C "[любая метка, например control-node]" -f ~/.ssh/github_ed25519
ssh-copy-id root@[IP proxy]
ssh-copy-id root@[IP web]
ssh-copy-id root@[IP db]
```

`ssh-copy-id` работает, пока на целевом сервере разрешён вход по паролю (на свежем VPS так обычно и есть). Если ключ был добавлен при создании VPS в панели хостинга, этот шаг не нужен.

**Обязательная проверка** (baseline потом отключит вход по паролю):

```bash
for ip in [IP proxy] [IP web] [IP db]; do
  ssh -o PasswordAuthentication=no root@$ip "echo ok $ip"
done
```

Должно быть три строки `ok ...`. Если где-то запрашивают пароль или отказывают, дальше не идите. Запасной вход: веб-консоль хостинга.

## Шаг 6. Файл inventory.ini с реальными адресами

```bash
cp inventory.ini.example inventory.ini
nano inventory.ini
```

Нужно вписать реальные IP трёх серверов вместо заглушек (`[IP proxy]`, `[IP web]`, `[IP db]`). Проверка:

```bash
ansible-inventory --list
```

## Шаг 7. Пароль vault и переменные

```bash
echo "[пароль vault]" > .vault_pass
chmod 600 .vault_pass
ansible-vault view group_vars/all/vault.yml
```

Если `view` показал переменные, пароль верный. Теперь нужно поправить то, что изменилось с прошлого раза:

```bash
EDITOR=nano ansible-vault edit group_vars/all/vault.yml
```

В vault лежат IP серверов (`vault_proxy_ip_address`, `vault_web_ip_address`, `vault_db_ip_address`). Если серверы новые, **обновить их здесь**. Остальные секреты (пароли, соли, ключи WordPress) не трогать, если разворачивается тот же сайт заново. Если домен новый, поменяйть `fqdn_domain` и `domain_name` в `group_vars/all/all.yml`, а ключи WordPress сгенерировать новые.

## Шаг 8. Проверки перед запуском

```bash
ansible all -m ping
ansible-playbook site.yml --syntax-check
nslookup [домен]
```

- `ansible all -m ping`: три ответа `pong` (связь с серверами есть).
- `--syntax-check`: без ошибок.
- `nslookup`: домен (и `www.[домен]`) указывает на `[IP proxy]`.

**Если это тестовый прогон** (например, проверка на чистых серверах), временно добавить `--staging` в задачу certbot в `proxy.yml`, чтобы не тратить лимит настоящих сертификатов (5 в неделю на один набор доменов).

## Шаг 9. Запуск

```bash
ansible-playbook site.yml
```

Плейбуки лучше выполнить по порядку: baseline → db → web → proxy. Это займёт несколько минут. Для отладки можно запускать по частям:

```bash
ansible-playbook baseline.yml
ansible-playbook site.yml --limit db
```

Потом запустить `site.yml` **ещё раз**: на втором прогоне должно быть `changed=0` и `failed=0` у всех хостов (идемпотентность).

## Шаг 10. Проверить результат

| Где | Команда | Ожидаем |
|---|---|---|
| web | `ss -tlnp \| grep apache2` | только порт 8080 |
| db | `ss -tlnp \| grep -E '3306\|33060'` | 3306 на `0.0.0.0`, 33060 на `127.0.0.1` |
| proxy | `curl -sI https://[домен] \| head -5` | ответ WordPress (`302` на страницу установки или `200`) |
| proxy | `certbot certificates` | сертификат на оба домена |
| proxy | `certbot renew --dry-run` | симулированное продление успешно |

Открыть `https://[домен]` в браузере и пройти мастер установки WordPress.

## Шаг 11. После тестового прогона

Если использовался `--staging`:

1. Убрать `--staging` из задачи certbot в `proxy.yml`.
2. На сервере proxy удалить тестовый сертификат: `certbot delete --cert-name [домен]`.
3. Запустить `ansible-playbook proxy.yml`, затем ещё раз (ожидаем `changed=0`).
4. Проверить: `certbot certificates` (вместо `TEST_CERT` должен быть настоящий срок), сайт открывается в браузере без предупреждений.

Сохранить копию сертификатов на будущее (см. раздел «Бэкап сертификатов» в README).

## Если что-то не работает

| Симптом | Причина и решение |
|---|---|
| `couldn't resolve module` | Не установлены коллекции (шаг 4) |
| `Vault password: ... Decryption failed` | Неверный пароль в `.vault_pass` (шаг 7) |
| `UNREACHABLE` или `Permission denied` у хоста | Нет доступа по ключу (шаг 5) или неверный IP в `inventory.ini` |
| `Host key verification failed` | Сервер пересоздан, ключ хоста изменился: `ssh-keygen -R [IP]` и повторить |
| `Could not get lock` у apt | Сервер ещё обновляется, подождите (в плейбуках стоит `lock_timeout: 300`) |
| Certbot не получает сертификат | DNS ещё не указывает на proxy или закрыт порт 80 |
| Nginx не стартует | `nginx -t` покажет причину; проверьте `certbot certificates` |
