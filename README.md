# `psmongodb` – Ansible роль для установки и настройки Percona Server for MongoDB

Эта роль устанавливает, настраивает и управляет **Percona Server for MongoDB** в режиме ReplicaSet, включая:

- Подготовку ОС (отключение THP, создание директорий)
- Настройку репозиториев и установку нужной версии MongoDB
- Генерацию и загрузку `keyFile` для аутентификации между членами ReplicaSet
- Настройку `mongod.conf`
- Инициализацию и расширение ReplicaSet
- Создание администратора (root) MongoDB
- **Установку и настройку Percona Backup for MongoDB (PBM)** для резервного копирования
- Удаление MongoDB при необходимости

---

## 📦 Переменные

### Обязательные (обычно в `group_vars` или `inventory`):

```yaml
mongo_desired_action: install  # или 'update_conf', 'upgrade', 'wipe'

mongo_rs_members:
  - host: hostname1
    role: primary
  - host: hostname2
    role: secondary
  - host: hostname3
    role: arbiter

mongo_pkg_version: "8.0"       # Версия MongoDB: major.minor (8.0) или точная (8.0.16)
```

### 📌 Указание версии MongoDB:

Переменная `mongo_pkg_version` поддерживает два формата:

**Формат 1: Major.Minor (автоматически выбирается последняя patch-версия)**
```yaml
mongo_pkg_version: "8.0"   # Установит последнюю доступную 8.0.x
```

**Формат 2: Major.Minor.Patch (точная версия)**
```yaml
mongo_pkg_version: "8.0.16"   # Установит точно 8.0.16
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
| `mongo_install_pbm`      | Установить PBM при действии `install`                | `true`                               |
| `mongo_pbm_skip`         | Пропустить установку PBM при действии `pbm_install`  | `false`                              |
| `pymongo_version`        | Версия PyMongo для модулей community.mongodb         | `4.13.2`                             |

---

## 🔧 Доступные действия (`mongo_desired_action`)

- `install` – Полная установка MongoDB + ReplicaSet + пользователь + PBM (можно отключить PBM через `mongo_install_pbm: false`)
- `update_conf` – Только перерендерить `mongod.conf` и перезапустить
- `upgrade` – Обновление MongoDB до новой версии с проверками совместимости
- `wipe` – Полное удаление MongoDB и связанных данных
- `pbm_install` – Только установка и настройка Percona Backup for MongoDB (для добавления на существующий кластер)

---

## 🔒 Примеры использования

### Установка MongoDB + PBM (по умолчанию)

```yaml
- name: Установка MongoDB с PBM
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
        # Настройки S3 для PBM (обязательны для бэкапов)
        pbm_s3_bucket: "my-backup-bucket"
        pbm_s3_region: "us-east-1"
        pbm_s3_access_key: "your_access_key"
        pbm_s3_secret_key: "your_secret_key"
        pbm_s3_endpoint: "https://obs.ru-moscow-1.hc.sbercloud.ru"
```

### Установка только MongoDB (без PBM)

Если вам не нужен PBM, отключите его:

```yaml
- name: Установка только MongoDB
  hosts: mongodb
  become: true
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: install
        mongo_install_pbm: false    # Отключить установку PBM
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

## 💾 Установка Percona Backup for MongoDB (PBM)

### Автоматическая установка с MongoDB

По умолчанию PBM устанавливается автоматически при `mongo_desired_action: install`. Просто укажите настройки S3 в переменных.

### Отдельная установка PBM на существующий кластер

Если MongoDB уже установлена, и вы хотите добавить только PBM, используйте действие `pbm_install`:

```yaml
- name: Установка PBM
  hosts: mongodb
  become: true
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: pbm_install
        # Обязательные переменные для работы с MongoDB
        mongo_admin_pwd: "your_admin_password"
        # Настройки S3 хранилища (опционально)
        pbm_s3_bucket: "my-backup-bucket"
        pbm_s3_region: "us-east-1"
        pbm_s3_prefix: "mongodb-backups/PROD"
        pbm_s3_access_key: "your_access_key"
        pbm_s3_secret_key: "your_secret_key"
        pbm_s3_endpoint: "https://s3.example.com"  # опционально
        pbm_s3_force_path_style: false
```

**Пример для SberCloud Object Storage:**

```yaml
- name: Установка PBM с SberCloud OBS
  hosts: mongodb
  become: true
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: pbm_install
        mongo_admin_pwd: "your_admin_password"
        # SberCloud OBS настройки
        pbm_s3_bucket: "p-ihub-bareos"
        pbm_s3_prefix: "p-ihub-bareos/PROD"
        pbm_s3_region: "ru-moscow-1"
        pbm_s3_endpoint: "https://obs.ru-moscow-1.hc.sbercloud.ru"
        pbm_s3_force_path_style: false
        pbm_s3_access_key: "your_access_key"
        pbm_s3_secret_key: "your_secret_key"
        # Настройки restore для оптимизации нагрузки
        pbm_restore_batch_size: 100
        pbm_restore_num_insertion_workers: 1
        pbm_restore_num_download_workers: 1
        pbm_restore_max_download_buffer_mb: 256
        pbm_restore_download_chunk_mb: 16
```

### Переменные для PBM:

| Переменная                           | Описание                                    | Значение по умолчанию                                      |
|--------------------------------------|---------------------------------------------|------------------------------------------------------------|
| `pbm_version`                        | Версия PBM                                  | `2.8`                                                      |
| `pbm_user`                           | Имя пользователя MongoDB для PBM            | `pbmuser`                                                  |
| `pbm_user_password`                  | Пароль пользователя PBM                    | Генерируется автоматически                                 |
| `pbm_backup_dir`                     | Локальная директория для бэкапов           | `/data/backup`                                             |
| `pbm_s3_bucket`                      | S3 bucket для хранения бэкапов             | Не задано (обязательно для S3)                             |
| `pbm_s3_region`                      | Регион S3                                   | `us-east-1`                                                |
| `pbm_s3_prefix`                      | Префикс для объектов в S3                  | Не задано                                                  |
| `pbm_s3_endpoint`                    | URL эндпоинта S3 (для совместимых сервисов)| Не задано                                                  |
| `pbm_s3_access_key`                  | Access Key для S3                           | Не задано (обязательно для S3)                             |
| `pbm_s3_secret_key`                  | Secret Key для S3                           | Не задано (обязательно для S3)                             |
| `pbm_s3_force_path_style`            | Использовать path-style URLs для S3         | `false`                                                    |
| `pbm_compression_level`              | Уровень сжатия (0-6)                        | `3`                                                        |
| `pbm_restore_batch_size`             | Количество документов в буфере при restore  | `100`                                                      |
| `pbm_restore_num_insertion_workers`  | Количество потоков вставки при restore      | `1`                                                        |
| `pbm_restore_num_download_workers`   | Количество потоков загрузки при restore     | `1`                                                        |
| `pbm_restore_max_download_buffer_mb` | Максимальный размер буфера загрузки (MB)    | `256`                                                      |
| `pbm_restore_download_chunk_mb`      | Размер загружаемого блока (MB)              | `16`                                                       |

### Что делает `pbm_install`:

1. Добавляет репозиторий Percona PBM
2. Устанавливает пакеты `percona-backup-mongodb`
3. Создает специальную роль `pbmAnyAction` в MongoDB
4. Создает пользователя `pbmuser` с необходимыми правами
5. Настраивает конфигурацию S3 (если указаны параметры)
6. Создает директорию для локальных бэкапов
7. Настраивает и запускает службу `pbm-agent`
8. Сохраняет учетные данные в `/root/pbm_credentials.txt`

---

## ⬆️ Обновление MongoDB (Upgrade)

Роль поддерживает безопасное обновление MongoDB до новой версии с автоматическими проверками.

### Использование:

```yaml
- name: Upgrade MongoDB
  hosts: mongodb
  become: true
  serial: 1  # ВАЖНО: обновлять по одному узлу за раз!
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: upgrade
        mongo_pkg_version: "8.0"  # Целевая версия
```

⚠️ **ВАЖНО:** Используйте `serial: 1` для последовательного обновления узлов по одному!

### Что делает upgrade:

1. ✅ Проверяет текущую версию MongoDB (через `mongod --version`)
2. ✅ Проверяет, что целевая версия новее текущей
3. ✅ **Предотвращает пропуск версий** (нельзя с 6.0 сразу на 8.0!)
4. ✅ Останавливает mongod на текущем узле
5. ✅ Обновляет пакеты MongoDB
6. ✅ Запускает mongod и ждет готовности
7. ✅ Делает паузу для стабилизации ReplicaSet
8. ✅ Переходит к следующему узлу
9. ⚠️ Выводит инструкцию для `setFeatureCompatibilityVersion`

### ⚠️ Важные ограничения:

**MongoDB НЕ поддерживает пропуск major версий!**

❌ **Нельзя:**
- 6.0 → 8.0 напрямую
- 5.0 → 7.0 напрямую

✅ **Правильный путь обновления:**

```
6.0 → 7.0 → 8.0
```

**Пример пошагового обновления:**

```yaml
# Шаг 1: Обновление с 6.0 на 7.0
- hosts: mongodb
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: upgrade
        mongo_pkg_version: "7.0"
```

После завершения выполните вручную:
```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "7.0" })
```

```yaml
# Шаг 2: Обновление с 7.0 на 8.0
- hosts: mongodb
  roles:
    - role: psmongodb
      vars:
        mongo_desired_action: upgrade
        mongo_pkg_version: "8.0"
```

После завершения выполните вручную:
```javascript
db.adminCommand({ setFeatureCompatibilityVersion: "8.0" })
```

### 🔴 Ошибка при попытке пропустить версию:

Если попытаться обновиться с 6.0 на 8.0, вы получите ошибку:

```
===== UPGRADE ERROR: Version Skip Detected =====

You are trying to upgrade from 6.0 to 8.0.

MongoDB does NOT support skipping major versions!

Required upgrade path:
  1. First upgrade to MongoDB 7.0
  2. Run setFeatureCompatibilityVersion("7.0")
  3. Then upgrade to MongoDB 8.0

Please set mongo_pkg_version to the intermediate version first.
================================================
```

### 📋 Post-Upgrade: Feature Compatibility Version

После успешного обновления **обязательно** выполните вручную:

```bash
# 1. Подключитесь к PRIMARY
mongosh --host <primary-host> -u admin -p <password> --authenticationDatabase admin

# 2. Проверьте текущий FCV
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })

# 3. Обновите FCV
db.adminCommand({ setFeatureCompatibilityVersion: "8.0" })

# 4. Проверьте обновление
db.adminCommand({ getParameter: 1, featureCompatibilityVersion: 1 })
```

⚡ **До выполнения `setFeatureCompatibilityVersion` новые возможности MongoDB 8.0 будут недоступны!**

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

### Основные действия:
- `mongodb_install` - полная установка MongoDB + ReplicaSet + пользователи + PBM
- `mongodb_update_conf` - только обновление конфигурации
- `mongodb_upgrade` - обновление MongoDB до новой версии
- `mongodb_wipe` - полное удаление MongoDB
- `mongodb_pbm_install` - только установка PBM (отдельное действие)

### Части установки (внутри `install`):
- `mongodb_prepare_os` - подготовка ОС (THP, репозитории, пакеты)
- `mongodb_configure` - настройка mongod.conf
- `mongodb_replicaset` - инициализация ReplicaSet
- `mongodb_users` - создание пользователя admin
- `mongodb_keyfile` - настройка аутентификации
- `mongodb_pbm` - установка Percona Backup for MongoDB

### Примеры использования тегов:

```bash
# Установить только PBM (MongoDB уже должна быть установлена)
ansible-playbook playbook.yml --tags mongodb_pbm

# Выполнить полную установку
ansible-playbook playbook.yml --tags mongodb_install

# Только подготовка ОС и установка пакетов
ansible-playbook playbook.yml --tags mongodb_prepare_os
```

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
│   ├── pbm_install.yml
│   ├── prepare_os.yml
│   ├── replicaset.yml
│   ├── update_conf.yml
│   ├── users.yml
│   └── wipe.yml
├── templates/
│   ├── mongod.conf.j2
│   └── pbm_config_s3.yaml.j2
```
