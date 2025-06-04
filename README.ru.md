[English version](./README.md)

Это руководство содержит инструкции по запуску Ansible-плейбуков для настройки VPS и установки VMess.

### Примечания

- Сервер VMess разворачивается с использованием официального Docker-образа: [`v2fly/v2fly-core`](https://hub.docker.com/r/v2fly/v2fly-core).

## Требования к инфраструктуре

Перед началом убедитесь, что у вас есть следующее:

- **Ansible-хост:** Машина, на которой будут выполняться Ansible-плейбуки. На ней должны быть установлены Ansible и Python 3.x.
- **VPS-хост:** Целевой сервер, который будет настраиваться (VMess, Docker, Nginx и т.д.).
- **Домен или поддомен:** Необходим для выпуска SSL-сертификата через Certbot и настройки Nginx/VMess.
- **Prometheus-хост (опционально):** Сервер для сбора метрик с VPS через Node Exporter.
- **Инвентори-файл:** Файл инвентаря Ansible, описывающий вашу инфраструктуру и хосты. У вас должен быть доступ к этому файлу и необходимым учетным данным.

## Структура проекта

```
.
├── add_user.yml
├── delete_user.yml
├── generate_config.yml
├── README.md
├── vmess_init_and_configure.yml
└── roles
    ├── add_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── create_bootstrap_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── delete_user
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   └── tasks
    │       └── main.yml
    ├── generate_config
    │   ├── defaults
    │   │   ├── main_template.yml
    │   │   └── main.yml
    │   ├── tasks
    │   │   └── main.yml
    │   └── templates
    │       └── config_template.j2
    └── vmess_init_and_configure
        ├── defaults
        │   ├── main_template.yml
        │   └── main.yml
        ├── handlers
        │   └── main.yml
        ├── tasks
        │   ├── install_base_vmess.yml
        │   ├── install_docker.yml
        │   ├── install_fail2ban.yml
        │   ├── install_nodeexporter.yml
        │   ├── install_ufw.yml
        │   └── main.yml
        └── templates
            ├── base_config.json.j2
            ├── docker-compose.yml.j2
            ├── jail.local.j2
            ├── nginx_base.conf.j2
            ├── nginx.conf.j2
            ├── nginx_temp.conf.j2
            ├── node_exporter.service.j2
            └── prometheus_job.j2
```

## Основные роли

- **create_bootstrap_user:** Создание bootstrap-пользователя (будет использоваться для дальнейшей работы с VPS).
- **vmess_init_and_configure:** Установка и настройка:
    - UFW
    - Node Exporter
    - Fail2Ban
    - Docker
    - Nginx с certbot
    - VMess
- **add_user:** Роль для добавления клиента VMess.
- **delete_user:** Роль для удаления клиента VMess.
- **generate_config:** Роль для генерации клиентской конфигурации VMess.

## Как запустить плейбук

1. **Перейдите в директорию проекта:**
   ```bash
   cd ./VMESS
   ```

2. **Запустите плейбук:**
   ```bash
   ansible-playbook -i inventory/your_inventory your_playbook.yml
   ```
   - Замените `your_inventory` на ваш файл инвентаря.
   - Замените `your_playbook.yml` на нужный плейбук.

3. **Опционально: передайте переменные**
   ```bash
   ansible-playbook -i inventory/your_inventory your_playbook.yml -e '{"key": "value"}'
   ```
   Переменные можно задать двумя способами:
   - Описать их в `roles/.../defaults/main.yml` (будут использоваться по умолчанию).
   - Передать при запуске через опцию `-e` (строкой `-e "key=value"` или JSON `-e '{"key": "value"}'`). Переменные, переданные через `-e`, имеют приоритет над значениями из defaults.

## Как настроить VPS

> **Важно:** Плейбук `vmess_init_and_configure.yml` нужно запускать только для одного хоста из инвентаря за раз (используйте `-l target_host`). Не запускайте его одновременно для нескольких хостов.

1. **Запустите плейбук:**
   ```bash
   ansible-playbook -i inventory/your_inventory vmess_init_and_configure.yml -l target_host --ask-pas
   ```
   - Замените `your_inventory` на ваш файл инвентаря.
   - Замените `target_host` на нужный хост.

2. **Опционально: передайте переменные**
   ```bash
   ansible-playbook -i inventory/your_inventory vmess_init_and_configure.yml -e '{"key": "value"}' --ask-pas
   ```
   Переменные можете задать в `roles/vmess_init_and_configure/defaults/main.yml` или передать их через `-e` при запуске.

### Основные переменные

#### VM с VMess

- **ssh_port**: SSH-порт на VM. Должен совпадать с `ansible_port` в инвентаре.
- **server_name**: Домен или поддомен для VMess и SSL-сертификата (используется Nginx и Certbot).
- **certbot_email**: Email для Certbot (регистрация и продление сертификатов).
- **nginx_conf_path**: Путь к конфигу Nginx для VMess на VM.
- **docker_dir**: Директория на VM для конфигов Docker Compose и VMess.
- **fail2ban_ignoreip**: Список IP, которые игнорируются Fail2Ban для защиты SSH (через пробел).
- **node_exporter_allowed_src_ip**: IP, с которого разрешён доступ к метрикам Node Exporter (обычно Prometheus).

#### VM/Prometheus-хост

- **prometheus_conf_paths**: Путь к конфигу Prometheus на Prometheus-хосте.
- **prometheus_host**: Имя Prometheus-хоста из инвентаря Ansible.

#### Переменные для опциональных установок

- **install_fail2ban**: Устанавливаем/Настраиваем  Fail2Ban (`true` или `false`).
- **install_nodeexporter**: Устанавливаем/Настраиваем  Node Exporter (`true` или `false`).
- **install_ufw**: Устанавливаем/Настраиваем UFW (`true` или `false`).
- **install_docker**: Устанавливаем/Настраиваем Docker и Docker Compose (`true` или `false`).
- **setup_prometheus**: Добавляем scrape-конфиг в Prometheus на внешнем сервере (`true` или `false`).

---

## Как добавить клиента vmess на VPS
> **Примечание:** Плейбук `add_user.yml` можно запускать сразу для нескольких VMess-хостов, указав группу или несколько хостов в инвентаре.  
> **Важно:** Все целевые хосты должны иметь одинаковые пути к конфигурационным файлам VMess (`vmess_config_file_path`, `docker_compose_dir`), чтобы плейбук работал корректно.

1. **Запустите плейбук:**
   ```bash
   ansible-playbook -i inventory/your_inventory add_user.yml -l target_host
   ```
   - Замените `your_inventory` на ваш файл инвентаря.
   - Замените `your_playbook.yml` на нужный плейбук.
   - Замените `target_host` на нужный хост или группу хостов.

2. **Опционально: передайте переменные**
   ```bash
   ansible-playbook -i inventory/your_inventory add_user.yml -e '{"key": "value"}'
   ```
    Переменные можно задать двумя способами:
    - Описать их в `roles/add_user/defaults/main.yml` (будут использоваться по умолчанию).
    - Передать при запуске через опцию `-e` (строкой `-e "key=value"` или JSON `-e '{"key": "value"}'`). Переменные, переданные через `-e`, имеют приоритет над значениями из defaults.

    Ниже приведены основные переменные для настройки сервера VMess. По умолчанию они определены в `roles/add_user/defaults/main.yml`, но вы можете переопределить их при запуске.

### Переменные для роли add_user

- **vmess_config_file_path**: Путь к конфигу VMess-сервера, в который будет добавлен пользователь.
- **docker_compose_dir**: Путь к директории с docker-compose и конфигами VMess.
- **email**: Имя или идентификатор нового пользователя VMess (используется как метка или для идентификации клиента).
---

## Как удалить клиента vmess на VPS
> **Примечание:** Плейбук `delete_user.yml` можно запускать сразу для нескольких VMess-хостов, указав группу или несколько хостов в инвентаре.  
> **Важно:** Все целевые хосты должны иметь одинаковые пути к конфигурационным файлам VMess (`vmess_config_file_path`, `docker_compose_dir`), чтобы плейбук работал корректно.

1. **Запустите плейбук:**
   ```bash
   ansible-playbook -i inventory/your_inventory delete_user.yml -l target_host
   ```
   - Замените `your_inventory` на ваш файл инвентаря.
   - Замените `your_playbook.yml` на нужный плейбук.
   - Замените `target_host` на нужный хост или группу хостов.

2. **Опционально: передайте переменные**
   ```bash
   ansible-playbook -i inventory/your_inventory delete_user.yml -e '{"key": "value"}'
   ```
    Переменные можно задать двумя способами:
    - Описать их в `roles/delete_user/defaults/main.yml` (будут использоваться по умолчанию).
    - Передать при запуске через опцию `-e` (строкой `-e "key=value"` или JSON `-e '{"key": "value"}'`). Переменные, переданные через `-e`, имеют приоритет над значениями из defaults.

    Ниже приведены основные переменные для настройки сервера VMess. По умолчанию они определены в `roles/delete_user/defaults/main.yml`, но вы можете переопределить их при запуске.

### Переменные для роли delete_user

- **vmess_config_file_path**: Путь к конфигу VMess-сервера, из которого будет удалён пользователь.
- **docker_compose_dir**: Путь к директории с docker-compose и конфигами VMess.
- **email**: Имя или идентификатор пользователя VMess (используется как метка или для идентификации клиента).
---

## Как сгенерировать конфиг клиента vmess
> **Примечание:** Плейбук `generate_config.yml` можно запускать сразу для нескольких VMess-хостов, указав группу или несколько хостов в инвентаре.  
> **Важно:** Все целевые хосты должны иметь одинаковые пути к конфигурационным файлам VMess (`vmess_config_file_path`, `docker_compose_dir`), чтобы плейбук работал корректно.

1. **Запустите плейбук:**
   ```bash
   ansible-playbook -i inventory/your_inventory generate_config.yml -l target_host
   ```
   - Замените `your_inventory` на ваш файл инвентаря.
   - Замените `your_playbook.yml` на нужный плейбук.
   - Замените `target_host` на нужный хост или группу хостов.

2. **Опционально: передайте переменные**
   ```bash
   ansible-playbook -i inventory/your_inventory generate_config.yml -e '{"key": "value"}'
   ```
    Переменные можно задать двумя способами:
    - Описать их в `roles/generate_config/defaults/main.yml` (будут использоваться по умолчанию).
    - Передать при запуске через опцию `-e` (строкой `-e "key=value"` или JSON `-e '{"key": "value"}'`). Переменные, переданные через `-e`, имеют приоритет над значениями из defaults.

    Ниже приведены основные переменные для настройки сервера VMess. По умолчанию они определены в `roles/generate_config/defaults/main.yml`, но вы можете переопределить их при запуске.

### Переменные для роли generate_config

- **vmess_config_file_path**: Путь к конфигу VMess-сервера, из которого будет сгенерирован клиентский конфиг.
- **email**: Имя или идентификатор пользователя VMess.

#### Интеграция с Password Push

Если вы хотите автоматически опубликовать сгенерированный конфиг и получить ссылку через [Password Pusher](https://pwpush.com/):

- **pwd_push**: Установите в `true`, чтобы включить публикацию через Password Pusher.
- **pwpush_url**: URL вашего сервиса Password Pusher.
- **pwpush_email**: Email для аутентификации в Password Pusher (user/admin).
- **pwpush_api_token**: API-токен для аутентификации в Password Pusher.

Если `pwd_push`- `true`, плейбук отправит сгенерированный конфиг в указанный сервис Password Pusher и вернёт ссылку для передачи.  
Если `pwd_push` - `false`, сгенерированный клиентский конфиг будет выведен прямо в консоль.

---