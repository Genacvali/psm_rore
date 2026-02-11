# `psmongodb` – Ansible роль для установки и настройки Percona Server for MongoDB

Эта роль устанавливает, настраивает и управляет **Percona Server for MongoDB** в режиме ReplicaSet, включая:

- Подготовку ОС (отключение THP, создание директорий)
- Настройку репозиториев и установку нужной версии MongoDB
- Генерацию и загрузку `keyFile` для аутентификации между членами ReplicaSet
- Настройку `mongod.conf`
- Инициализацию и расширение ReplicaSet
- Создание администратора (root) MongoDB
- Удаление MongoDB при необходимости

---

## 📦 Переменные

### Обязательные (обычно в `group_vars` или `inventory`):

```yaml
mongo_desired_action: install  # или 'update_conf', 'wipe'

mongo_rs_members:
  - host: hostname1
    role: primary
  - host: hostname2
    role: secondary
  - host: hostname3
    role: arbiter

mongo_pkg_version: "6.0"       # Версия MongoDB (без patch-номера)
```

### Опциональные:

| Переменная               | Описание                                             | Значение по умолчанию                |
|--------------------------|------------------------------------------------------|--------------------------------------|
| `mongo_port`             | Порт MongoDB                                         | `27017`                              |
| `mongo_db_path`          | Путь до данных                                       | `/data/data`                         |
| `mongo_log_path`         | Путь до логов                                        | `/data/logs`                         |
| `mongo_keyfile_dir`      | Каталог для keyFile                                  | `/data/ssl`                          |
| `mongo_keyfile_name`     | Имя файла keyFile                                    | `mongo.key`                          |
| `mongo_replset`          | Имя ReplicaSet                                       | Автоматически из `ansible_hostname` |
| `mongo_admin_pwd`        | Пароль для пользователя `admin`                      | Генерируется                         |
| `pymongo_version`        | Версия PyMongo для модулей community.mongodb         | `4.13.2`                             |

---

## 🔧 Доступные действия (`mongo_desired_action`)

- `install` – Полная установка MongoDB + ReplicaSet + пользователь
- `update_conf` – Только перерендерить `mongod.conf` и перезапустить
- `wipe` – Полное удаление MongoDB и связанных данных

---

## 🔒 Примеры использования

```yaml
- name: Установка MongoDB через роль
  hosts: mongodb
  become: true
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: install
        mongo_pkg_version: "6.0"
        mongo_rs_members:
          - host: hostname1
            role: primary
          - host: hostname2
            role: secondary
          - host: hostname3
            role: arbiter
```

---

## 🧹 Удаление MongoDB

```yaml
mongo_desired_action: wipe
```

Удаляет:
- Все RPM пакеты MongoDB
- Каталоги с данными, логами и keyfile
- Конфигурацию `mongod.conf`
- systemd unit для отключения THP

---

## ⚙️ Теги

Можно выполнять только части роли с помощью `--tags`:

- `mongodb_prepare_os`
- `mongodb_configure`
- `mongodb_replicaset`
- `mongodb_users`
- `mongodb_keyfile`
- `mongodb_wipe`
- `mongodb_install`

---

## 📝 Замечания

- Репликация и создание пользователя происходит **только на `primary`**.
- Версия MongoDB должна быть указана в `mongo_pkg_version` как `6.0`, `7.0`, `8.0` и т.д. — без patch версии.
- Репозиторий формируется автоматически из шаблона и использует **внутренний Nexus**.

---

## 📁 Структура роли

```bash
roles/psmongodb/
├── defaults/
│   └── main.yml
├── tasks/
│   ├── configure.yml
│   ├── derive_replname.yml
│   ├── install.yml
│   ├── keyfile.yml
│   ├── main.yml
│   ├── prepare_os.yml
│   ├── replicaset.yml
│   ├── update_conf.yml
│   └── users.yml
```
