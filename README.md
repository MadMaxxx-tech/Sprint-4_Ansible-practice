# Ansible Redis

Проект Ansible для установки и настройки сервиса Redis на серверах с OS семейства Debian.

Playbook устанавливает Redis, создаёт каталог для данных, формирует конфигурацию из Jinja2-шаблона, настраивает systemd override и запускает сервис с автозагрузкой.

## Структура проекта

```text
Sprint-4_Ansible-practice/
├── redis-playbook.yaml
├── README.md
├── inventories/
│   ├── dev/
│   │   ├── hosts
│   │   └── group_vars/
│   │       ├── all.yml
│   │       └── vault.yml
│   └── prod/
│       ├── hosts
│       └── group_vars/
│           ├── all.yml
│           └── vault.yml
└── roles/
    └── redis/
        ├── defaults/
        │   └── main.yml
        ├── tasks/
        │   └── main.yml
        ├── handlers/
        │   └── main.yml
        └── templates/
            ├── redis.conf.j2
            └── redis-dir.conf.j2
```

## Инвентари

Проект использует два независимых inventory:

- `inventories/dev/hosts` — окружение разработки;
- `inventories/prod/hosts` — производственное окружение.

Переключение окружения выполняется выбором соответствующего inventory при запуске playbook. Сам playbook при этом не изменяется.

## Переменные окружений

| Переменная          | Назначение                                         |
|---------------------|----------------------------------------------------|
| `redis_port`        | Порт, на котором работает Redis                    |
| `redis_dir`         | Каталог для хранения данных Redis                  |
| `redis_databases`   | Количество баз данных Redis                        |
| `redis_appendonly`  | Включение или отключение AOF                       |
| `redis_password`    | Пароль Redis; хранится в зашифрованном Vault-файле |

### Переменные dev

```yaml
redis_port: 6380
redis_dir: /var/lib/redis
redis_databases: 4
redis_appendonly: "no"
```

### Переменные prod

```yaml
redis_port: 6379
redis_dir: /var/lib/redis
redis_databases: 16
redis_appendonly: "yes"
```

## Ansible Vault

Пароль Redis хранится в отдельных зашифрованных файлах:

- `inventories/dev/group_vars/vault.yml`;
- `inventories/prod/group_vars/vault.yml`.

Файлы подключаются в playbook через `vars_files`:

```yaml
vars_files:
  - "{{ inventory_dir }}/group_vars/vault.yml"
```

Vault-пароль хранится в файле:

```text
/home/ansibler/.vault_pass
```

## Запуск playbook

### Запуск для dev

```bash
sudo -u ansibler -H ansible-playbook \
  -i inventories/dev/hosts \
  --vault-password-file=/home/ansibler/.vault_pass \
  redis-playbook.yaml
```

### Запуск для prod

```bash
sudo -u ansibler -H ansible-playbook \
  -i inventories/prod/hosts \
  --vault-password-file=/home/ansibler/.vault_pass \
  redis-playbook.yaml
```

## Проверка идемпотентности

После первого запуска необходимо выполнить playbook повторно.

Ожидаемый результат:

```text
changed=0
failed=0
```

Это подтверждает, что повторный запуск playbook не изменяет систему и не вызывает излишний перезапуск сервиса Redis.

## Проверка сервиса

```bash
sudo systemctl status redis-server --no-pager
```

Ожидаемый результат:

```text
Active: active (running)
```

## Безопасность

- Пароль Redis хранится только в зашифрованном Vault-файле.
- Пароль не должен попадать в `all.yml`, playbook, Git или документацию.
- Vault password file не должен добавляться в Git.
- Не используйте `shell` для установки или настройки Redis.
- Все параметры окружений задаются через inventory.
