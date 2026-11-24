# Ansible Deploy

Деплой Docker Compose-проекта из Git-репозитория на Linux-сервер.

## Что делает

- устанавливает Git;
- устанавливает Docker через [`get.docker.com`](https://get.docker.com/), если его нет;
- добавляет SSH-пользователя в группу `docker`;
- клонирует или обновляет репозиторий в `$HOME` пользователя;
- создаёт `.env` из `.env.example`, не затирая существующий;
- запускает `docker compose`.

## Настройка

Укажи сервер и SSH-ключ в `inventory.ini`:

```ini
[app_servers]
app-01 ansible_host=192.168.1.50 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519
```

Укажи репозиторий и каталог в `group_vars/all.yml`:

```yaml
repo_url: "git@github.com:company/project.git"
repo_version: "main"
repo_directory: "project"
repo_key_file: "{{ ansible_facts['env']['HOME'] }}/.ssh/id_ed25519"
```

> `ansible_ssh_private_key_file` — локальный ключ для входа на сервер.  
> `repo_key_file` — приватный ключ на сервере для доступа к Git-репозиторию.

Подготовь локальные файлы, которые будут скопированы на сервер:

```bash
cp playbooks/install_monitoring/export/.env.example \
  playbooks/install_monitoring/export/.env

cp playbooks/install_monitoring/export/allowlist.conf.example \
  playbooks/install_monitoring/export/allowlist.conf
```

Заполни их своими значениями. Реальные файлы исключены из Git.

## Запуск

```bash
ansible-galaxy collection install -r requirements.yml
ansible app_servers -m ansible.builtin.ping
ansible-playbook deploy.yml
```

Если `sudo` требует пароль:

```bash
ansible-playbook deploy.yml --ask-become-pass
```

Готовое приложение будет находиться в `$HOME/<repo_directory>` на сервере.

Playbook запуска стека можно выполнять отдельно, если репозиторий на серверах уже
клонирован:

```bash
ansible-playbook playbooks/install_monitoring/start.yaml
```

Он проверяет локальные `export/.env` и `export/allowlist.conf`, копирует их на
серверы, запускает Docker Compose и показывает фактически развёрнутый Git commit
через `git rev-parse HEAD`.

## Генерация Prometheus targets

Playbook `playbooks/install_monitoring/add-workers.yaml` создаёт два файла в формате
Prometheus file-based service discovery:

- `node-exporter.json`;
- `cadvisor.json`.

Список targets берётся из группы `[app_servers]` в `inventory.ini`. Для каждого хоста
используются `ansible_host` и имя хоста из inventory:

```ini
[app_servers]
crocus ansible_host=10.20.0.3 ansible_user=ubuntu ansible_ssh_private_key_file=~/.ssh/id_ed25519_crocus
```

Порты читаются на Ansible control node из
`playbooks/install_monitoring/export/.env`:

```dotenv
NODE_EXPORTER_PORT=8080
CADVISOR_PORT=9100
```

Оба значения должны состоять только из цифр. Создай рабочий файл из примера перед
первым запуском:

```bash
cp playbooks/install_monitoring/export/.env.example \
  playbooks/install_monitoring/export/.env
```

Рабочий `.env` и сгенерированные JSON исключены из Git.

### Куда записываются JSON

Каталог задаётся в `group_vars/all.yml`:

```yaml
monitoring_config_dir: "{{ playbook_dir }}/../../prometheus/file_sd"
```

При текущем локальном запуске это каталог `prometheus/file_sd` в корне репозитория.
Playbook создаёт его автоматически. Полные пути собираются внутри
playbook как `<monitoring_config_dir>/node-exporter.json` и
`<monitoring_config_dir>/cadvisor.json`.

### Как работает дополнение файлов

Playbook не затирает уже записанные вручную targets:

1. `stat` проверяет наличие каждого JSON на хосте выполнения playbook.
2. `slurp` читает существующий файл и возвращает его в Base64.
3. `b64decode` декодирует содержимое, а `from_json` превращает его в Ansible-список.
4. Jinja2 добавляет targets из текущей группы `app_servers`.
5. `unique(attribute='labels.instance')` убирает дубли по имени instance. Значение из
   текущего inventory имеет приоритет над старым значением.

Хосты, которых больше нет в `app_servers`, автоматически не удаляются: они остаются
в JSON, потому что задача работает в режиме дополнения. Для удаления target удали его
из JSON вручную.

Существующий файл должен содержать валидный JSON-массив. Если JSON повреждён,
`from_json` остановит playbook вместо перезаписи файла.

### Локальный запуск

Сейчас playbook настроен на:

```yaml
hosts: localhost
```

Запуск:

```bash
ansible-playbook playbooks/install_monitoring/add-workers.yaml
```

Повторный запуск идемпотентен: если inventory, порты и существующие targets не
изменились, обе template-задачи возвращают `ok`, а не `changed`.

### Запись конфигов на сервер

Чтобы генерировать файлы непосредственно на сервере, измени хост выполнения:

```yaml
hosts: main
```

И задай серверный каталог в `group_vars/all.yml`, например:

```yaml
monitoring_config_dir: /root/mon/prometheus/file_sd
```

Замени `/opt/monitoring` на реальный каталог, в котором на сервере лежит
`compose.yaml`. Задачи `stat`, `slurp` и `template` тогда будут работать с файлами на `main`.
Файл `export/.env` всё равно читается с control node через Ansible lookup. SSH-пользователь
или `become` должны давать права на чтение и запись `monitoring_config_dir`.


