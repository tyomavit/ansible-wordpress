# Памятка: как залить проект с нового VPS в GitHub

Когда нужна: проект лежит на сервере (в папке `~/[папка проекта]`), а на GitHub его ещё нет.

**Обозначения.** Всё в квадратных скобках это подстановка: `[имя аккаунта github]`, `[название репозитория]`. Сами скобки не вводить. Пометка **[сервер]** означает терминал на VPS (например, терминал VS Code по SSH), **[браузер]** означает сайт GitHub.

## Идея в одну минуту

Чтобы сервер мог отправлять код на GitHub, он должен доказать, что это вы. Для этого на сервере создаётся пара SSH-ключей: **приватный** остаётся на сервере и никому не показывается, **публичный** добавляется в свой аккаунт GitHub. После этого `git push` работает без пароля.

## Шаг 1. [браузер] Создать пустой репозиторий

1. github.com → `+` → **New repository**.
2. **Repository name**: `[название репозитория]`.
3. Выбрать **Private** (приватный).
4. 4. Галочки *Add a README*, *Add .gitignore*, *Choose a license* **не ставить**: иначе GitHub создаст в репозитории свой первый коммит, а на сервере уже есть свой. Это две несвязанные истории, и первый `git push` будет отклонён.
5. **Create repository**.

## Шаг 2. [сервер] Установить Git и представиться

```bash
git --version                         # если команды нет: apt install -y git
git config --global user.name "[Ваше имя]"
git config --global user.email "[ID]+[имя аккаунта github]@users.noreply.github.com"
```

Адрес `noreply` показан на GitHub: Settings → Emails (включить *Keep my email addresses private*). Он нужен, чтобы настоящая почта не попала в историю коммитов.

## Шаг 3. [сервер] Создать SSH-ключ для GitHub

```bash
ssh-keygen -t ed25519 -C "[метка, например control-node]" -f ~/.ssh/github_ed25519
cat ~/.ssh/github_ed25519.pub
```

На вопрос про passphrase можно нажать Enter (пусто). Вторая команда выводит **публичный** ключ (строка, начинающаяся с `ssh-ed25519`). Скопировать её целиком.

## Шаг 4. [браузер] Добавить публичный ключ в GitHub

GitHub → иконка профиля → **Settings** → **SSH and GPG keys** → **New SSH key**. Title: `[метка]`, в поле Key вставить скопированную строку, **Add SSH key**.

## Шаг 5. [сервер] Указать SSH, какой ключ использовать для GitHub

```bash
cat >> ~/.ssh/config <<'EOF'
Host github.com
  IdentityFile ~/.ssh/github_ed25519
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
ssh -T git@github.com
```

На вопрос про отпечаток (fingerprint) ответить `yes`. Ожидается сообщение `Hi [имя аккаунта github]! You've successfully authenticated`. Фраза о том, что shell-доступа нет, нормальна.

## Шаг 6. [сервер] Подготовить проект к безопасной загрузке

```bash
cd ~/[папка проекта]
```

Проверить файл `.gitignore`: в нём должны быть строки с секретами и лишним (для Ansible-проекта):

```
.vault_pass
inventory.ini
*.tar.gz
*.retry
```

Для файлов с реальными адресами сделать безопасную копию-образец, которая уйдёт в репозиторий:

```bash
cp inventory.ini inventory.ini.example
```

Открыть `inventory.ini.example` и заменить реальные IP на заглушки (`203.0.113.10`, `.11`, `.12`: диапазон, зарезервированный для примеров).

Проверка, что секреты не уедут:

```bash
git check-ignore -v .vault_pass inventory.ini    # покажет, какие правила их скрывают
head -1 group_vars/all/vault.yml                 # должно быть: $ANSIBLE_VAULT;1.1;AES256
```

## Шаг 7. [сервер] Первый коммит

```bash
git init -b main          # если папка ещё не репозиторий (git status ругается: not a git repository)
git add .
git status                # ВАЖНО: просмотреть список файлов
git commit -m "[что это за проект и что делает]"
```

`git status` после `git add .` показывает, что попадёт в коммит. Если в списке есть `.vault_pass`, `inventory.ini`, архивы или любые пароли, остановиться и исправить `.gitignore` (секрет, попавший в историю Git, остаётся там навсегда).

## Шаг 8. [сервер] Привязать репозиторий и отправить

```bash
git remote add origin git@github.com:[имя аккаунта github]/[название репозитория].git
git push -u origin main
```

`remote add origin` запоминает адрес репозитория под именем `origin`. Флаг `-u` связывает локальную ветку `main` с удалённой, дальше хватит просто `git push`.

## Шаг 9. [браузер] Проверить результат

Обновить страницу репозитория: должны быть файлы и README. Убедиться, что нет `.vault_pass` и `inventory.ini`, а `vault.yml` внутри показывает только зашифрованные строки.

Доступ для другого человека: **Settings → Collaborators → Add people**.

## Каждый день: рабочий цикл

```bash
git pull                                  # в начале: забрать изменения
git status                                # что изменилось
git add .
git commit -m "что и зачем изменил"       # смысл, а не «правки»
git push                                  # в конце: отправить
```

## Перед удалением VPS

1. `git status` чистый и `git push` сделан.
2. Нужные бэкапы (например, сертификатов) скачаны.
3. **Удалить ключ этого сервера на GitHub** (Settings → SSH and GPG keys → Delete). Пока он там, любой, кто получит доступ к серверу, получит доступ к репозиториям.

## Если что-то не работает

| Ошибка | Что делать |
|---|---|
| `Permission denied (publickey)` | Проверить, что публичный ключ добавлен на GitHub (шаг 4) и что `~/.ssh/config` написан верно (шаг 5). Диагностика: `ssh -vT git@github.com` |
| `remote origin already exists` | Адрес уже задан. Заменить: `git remote set-url origin git@github.com:[имя аккаунта github]/[название репозитория].git` |
| `rejected ... fetch first` или `unrelated histories` | В репозитории на GitHub уже есть файлы (например, README, созданный галочкой). Чтобы объединить: `git pull --rebase origin main`, потом `git push` |
| `fatal: not a git repository` | Я не в папке проекта (`cd`) или не выполнил `git init -b main` |
| `Author identity unknown` | Не выполнен шаг 2 (`git config --global user.name / user.email`) |
