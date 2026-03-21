# MySQL Backup Verification & Archive Script

Автоматическая проверка целостности MySQL бэкапов и создание резервных копий на NFS сервере.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Bash](https://img.shields.io/badge/bash-4.0+-green.svg)](https://www.gnu.org/software/bash/)
[![Platform](https://img.shields.io/badge/platform-linux-lightgrey.svg)](https://www.kernel.org/)

## 📋 Содержание

- [О проекте](#о-проекте)
- [Возможности](#возможности)
- [Как работает проверка целостности](#как-работает-проверка-целостности)
- [Требования](#требования)
- [Быстрый старт](#быстрый-старт)
- [Установка](#установка)
- [Настройка](#настройка)
- [Использование](#использование)
- [Мониторинг](#мониторинг)
- [Устранение проблем](#устранение-проблем)
- [FAQ](#faq)
- [Участие в проекте](#участие-в-проекте)
- [Лицензия](#лицензия)

---

## 🎯 О проекте

Скрипт решает проблему **неполных или поврежденных MySQL бэкапов** при записи по сети (NFS).

### Проблема

```
┌─────────────────┐                    ┌──────────────────┐
│   MySQL VM      │                    │   NFS Server     │
│   (CentOS 8)    │                    │   (Debian 10)    │
│                 │                    │                  │
│  Webmin backup ├────NFS write───────►│ backup.sql       │
│                 │    ⚠️ ОБРЫВ!       │ ❌ НЕПОЛНЫЙ!     │
└─────────────────┘                    └──────────────────┘
        │
        │ VM может упасть!
        ▼
    💥 Crash
```

**Последствия:**
- ❌ Бэкап записан не полностью
- ❌ Обнаруживается только при попытке восстановления
- ❌ Потеря критичных данных

### Решение

```
┌─────────────────┐                    ┌──────────────────────────────┐
│   MySQL VM      │                    │      NFS Server              │
│   (CentOS 8)    │                    │      (Debian 10)             │
│                 │                    │                              │
│  Webmin backup ├────NFS write───────►│ backup.sql                   │
└─────────────────┘                    │                              │
                                       │  ⏰ Скрипт запускается        │
                                       │                              │
                                       │  ✅ 7 проверок целостности   │
                                       │  ✅ Создание проверенной копии│
                                       │                              │
                                       │  archive/                    │
                                       │  └─ backup_verified.sql ✓    │
                                       │  └─ backup_verified.sql.md5  │
                                       └──────────────────────────────┘
```

---

## ✨ Возможности

- 🔍 **7 проверок целостности SQL файлов**
  - Размер файла
  - SQL заголовок и синтаксис
  - Завершающие маркеры
  - Проверка транзакций
  - GZIP целостность (для сжатых файлов)
  
- 📦 **Автоматическое архивирование**
  - Копирование проверенных бэкапов с timestamp
  - MD5 контрольные суммы
  - Автоочистка старых копий
  
- 🔔 **Уведомления**
  - Email алерты при ошибках
  - Подробное логирование
  - Интеграция с Zabbix/Telegram
  
- 🛡️ **Надежность**
  - Работает на стабильном NFS сервере
  - Не зависит от состояния виртуальных машин
  - Проверка монтирования NFS

---

## 🔍 Как работает проверка целостности

Скрипт выполняет **многоуровневую проверку**:

### Уровень 1: Быстрые проверки (мгновенно)

```bash
✓ Проверка 1: Файл не пустой (> 0 байт)
✓ Проверка 2: Минимальный размер (> 100KB)
✓ Проверка 3: Валидный SQL заголовок
```

### Уровень 2: Проверки содержимого (секунды)

```bash
✓ Проверка 4: Завершающие маркеры дампа
✓ Проверка 5: Отсутствие ROLLBACK команд
✓ Проверка 6: Парные кавычки (синтаксис SQL)
```

### Уровень 3: Специальные проверки

```bash
✓ Проверка 7: GZIP целостность (если файл сжат)
```

### Опционально: Тест восстановления (медленно, 100% надежность)

```bash
○ Создание тестовой БД
○ Восстановление дампа
○ Удаление тестовой БД
```

**Подробнее:** См. документацию [SQL_INTEGRITY_CHECKS.md](docs/SQL_INTEGRITY_CHECKS.md)

---

## 📦 Требования

### Операционная система

- **NFS Server:** Debian 10/11/12, Ubuntu 20.04+, или любой Linux с NFS
- **MySQL Server:** CentOS 7/8, Rocky Linux 8/9, или любой Linux

### Программное обеспечение

```bash
# Обязательно
- bash >= 4.0
- coreutils (stat, find, grep, wc)
- NFS server/client

# Опционально
- mailx (для email уведомлений)
- MySQL client (для теста восстановления)
- md5sum (для контрольных сумм)
```

### Архитектура

```
NFS Server (где запускается скрипт)
├── /data/nfs/backup/mysql-vm/           ← Webmin пишет сюда
│   ├── mysql_backup_20260206.sql        ← Оригинальный бэкап
│   └── archive/                          ← Скрипт создает копии тут
│       ├── mysql_backup_*_verified.sql
│       └── mysql_backup_*_verified.sql.md5
```

---

## 🚀 Быстрый старт

### 1. Клонирование репозитория

```bash
# На NFS сервере (Debian)
cd /opt
git clone https://github.com/yourusername/mysql-backup-verify.git
cd mysql-backup-verify
```

### 2. Установка скрипта

```bash
# Копирование скрипта
sudo cp mysql_backup_verify.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/mysql_backup_verify.sh
```

### 3. Настройка

```bash
# Редактирование конфигурации
sudo nano /usr/local/bin/mysql_backup_verify.sh

# Измените эти параметры:
BACKUP_DIR="/data/nfs/backup/mysql-vm"           # Путь к бэкапам
BACKUP_PATTERN="mysql_backup_*.sql"              # Маска файлов
ARCHIVE_DIR="/data/nfs/backup/mysql-vm/archive"  # Путь к архиву
EMAIL_ALERT="admin@example.com"                  # Email (опционально)
```

### 4. Тестовый запуск

```bash
# Запуск вручную
sudo /usr/local/bin/mysql_backup_verify.sh

# Проверка логов
sudo tail -50 /var/log/mysql_backup_verify.log
```

### 5. Настройка автозапуска

```bash
# Добавление в cron
sudo crontab -e

# Добавьте строку (если Webmin создает бэкап в 02:00):
0 3 * * * /usr/local/bin/mysql_backup_verify.sh >> /var/log/mysql_backup_verify.log 2>&1
```

---

## 📥 Установка

### Вариант 1: Установка из Git (рекомендуется)

```bash
# Клонирование
git clone https://github.com/yourusername/mysql-backup-verify.git
cd mysql-backup-verify

# Запуск установщика
sudo ./install.sh

# Следуйте инструкциям установщика
```

### Вариант 2: Ручная установка

```bash
# Скачивание скрипта
wget https://raw.githubusercontent.com/yourusername/mysql-backup-verify/main/mysql_backup_verify.sh

# Перемещение и установка прав
sudo mv mysql_backup_verify.sh /usr/local/bin/
sudo chmod +x /usr/local/bin/mysql_backup_verify.sh

# Создание директорий
sudo mkdir -p /var/log
sudo mkdir -p /data/nfs/backup/mysql-vm/archive
```

### Вариант 3: Установка через package manager (планируется)

```bash
# Debian/Ubuntu (в разработке)
sudo apt install mysql-backup-verify

# RHEL/CentOS (в разработке)
sudo yum install mysql-backup-verify
```

---

## ⚙️ Настройка

### Конфигурационный файл

Откройте скрипт для редактирования:

```bash
sudo nano /usr/local/bin/mysql_backup_verify.sh
```

### Основные параметры

```bash
# ==========================================
# ОБЯЗАТЕЛЬНЫЕ НАСТРОЙКИ
# ==========================================

# Директория где Webmin сохраняет бэкапы (на NFS сервере - ЛОКАЛЬНЫЙ путь!)
BACKUP_DIR="/data/nfs/backup/mysql-vm"

# Маска имени файла бэкапа
# Если Webmin создает файлы с датой:
BACKUP_PATTERN="mysql_backup_*.sql"
# Если Webmin перезаписывает один файл:
# BACKUP_PATTERN="mysql_backup.sql"
# Если файлы сжаты:
# BACKUP_PATTERN="mysql_backup_*.sql.gz"

# Директория для проверенных копий
ARCHIVE_DIR="/data/nfs/backup/mysql-vm/archive"
```

### Дополнительные параметры

```bash
# ==========================================
# ОПЦИОНАЛЬНЫЕ НАСТРОЙКИ
# ==========================================

# Лог файл
LOG_FILE="/var/log/mysql_backup_verify.log"

# Email для уведомлений (оставьте пустым если не нужно)
EMAIL_ALERT="admin@example.com"

# Максимальный возраст файла для проверки (в минутах)
# 60 = проверять только файлы созданные в последний час
# 180 = проверять файлы за последние 3 часа (рекомендуется для NFS)
MAX_FILE_AGE=180

# Срок хранения проверенных копий (в днях)
RETENTION_DAYS=30
```

### Настройка email уведомлений

```bash
# Установка mailx
sudo apt install mailx -y   # Debian/Ubuntu
sudo yum install mailx -y   # CentOS/RHEL

# Настройка SMTP (опционально)
sudo nano /etc/mail.rc
```

Добавьте:
```
set smtp=smtp://your-smtp-server:587
set smtp-use-starttls
set smtp-auth=login
set smtp-auth-user=your-email@example.com
set smtp-auth-password=your-password
set from=your-email@example.com
```

### Настройка cron

```bash
# Редактирование crontab
sudo crontab -e
```

**Примеры расписаний:**

```bash
# Ежедневно в 03:00 (если Webmin создает бэкап в 02:00)
0 3 * * * /usr/local/bin/mysql_backup_verify.sh >> /var/log/mysql_backup_verify.log 2>&1

# Каждый час в 15 минут
15 * * * * /usr/local/bin/mysql_backup_verify.sh >> /var/log/mysql_backup_verify.log 2>&1

# Несколько раз в день (03:00, 09:00, 15:00, 21:00)
0 3,9,15,21 * * * /usr/local/bin/mysql_backup_verify.sh >> /var/log/mysql_backup_verify.log 2>&1
```

---

## 💻 Использование

### Ручной запуск

```bash
# Запуск скрипта
sudo /usr/local/bin/mysql_backup_verify.sh

# Запуск с отладкой
sudo bash -x /usr/local/bin/mysql_backup_verify.sh

# Запуск и сохранение вывода
sudo /usr/local/bin/mysql_backup_verify.sh | tee /tmp/backup_verify_output.log
```

### Просмотр логов

```bash
# Последние 50 строк
sudo tail -50 /var/log/mysql_backup_verify.log

# В реальном времени
sudo tail -f /var/log/mysql_backup_verify.log

# Только ошибки
sudo grep "ОШИБКА" /var/log/mysql_backup_verify.log

# Только успешные проверки
sudo grep "успешно" /var/log/mysql_backup_verify.log

# Статистика за сегодня
sudo grep "$(date +%Y-%m-%d)" /var/log/mysql_backup_verify.log
```

### Проверка созданных копий

```bash
# Список всех проверенных копий
ls -lh /data/nfs/backup/mysql-vm/archive/

# Сортировка по дате
ls -lht /data/nfs/backup/mysql-vm/archive/ | head -10

# Проверка MD5 контрольной суммы
md5sum -c /data/nfs/backup/mysql-vm/archive/mysql_backup_*_verified.sql.md5

# Размер архива
du -sh /data/nfs/backup/mysql-vm/archive/

# Количество копий
ls -1 /data/nfs/backup/mysql-vm/archive/*.sql | wc -l
```

### Восстановление из проверенной копии

```bash
# Список доступных копий
ls -lh /data/nfs/backup/mysql-vm/archive/

# Восстановление
mysql -u root -p zabbix < /data/nfs/backup/mysql-vm/archive/mysql_backup_20260206_030015_verified.sql

# Если файл сжат
gunzip -c /data/nfs/backup/mysql-vm/archive/backup.sql.gz | mysql -u root -p zabbix
```

---

## 📊 Мониторинг

### Проверка статуса через cron

```bash
# Просмотр запланированных задач
sudo crontab -l

# Логи cron
sudo grep mysql_backup_verify /var/log/cron   # CentOS/RHEL
sudo grep mysql_backup_verify /var/log/syslog # Debian/Ubuntu

# Последний запуск
sudo ls -lt /var/log/mysql_backup_verify.log
```

### Интеграция с Zabbix

Создайте UserParameter:

```bash
sudo nano /etc/zabbix/zabbix_agentd.d/mysql_backup.conf
```

```ini
# Время последней успешной проверки
UserParameter=mysql.backup.last_success,grep "успешно" /var/log/mysql_backup_verify.log | tail -1 | awk '{print $1, $2}'

# Количество успешных проверок за сутки
UserParameter=mysql.backup.success_count,grep "$(date +%Y-%m-%d)" /var/log/mysql_backup_verify.log | grep "успешно" | wc -l

# Количество ошибок за сутки
UserParameter=mysql.backup.error_count,grep "$(date +%Y-%m-%d)" /var/log/mysql_backup_verify.log | grep "ОШИБКА" | wc -l

# Размер последней проверенной копии
UserParameter=mysql.backup.last_size,ls -lt /data/nfs/backup/mysql-vm/archive/*.sql 2>/dev/null | head -1 | awk '{print $5}'
```

Перезапустите агента:
```bash
sudo systemctl restart zabbix-agent
```

### Интеграция с Telegram

Добавьте в скрипт функцию:

```bash
send_telegram() {
    local BOT_TOKEN="your_bot_token"
    local CHAT_ID="your_chat_id"
    local message="$1"
    
    curl -s -X POST "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        -d chat_id="${CHAT_ID}" \
        -d text="${message}" > /dev/null
}
```

И вызовы в нужных местах:
```bash
send_telegram "✅ Бэкап MySQL проверен успешно на $(hostname)"
send_telegram "❌ ОШИБКА: Бэкап MySQL поврежден на $(hostname)"
```

---

## 🔧 Устранение проблем

### Проблема: "Не найдены свежие бэкапы"

**Причина:** Файлы старше `MAX_FILE_AGE` минут

**Решение:**
```bash
# Проверьте наличие файлов
ls -la /data/nfs/backup/mysql-vm/

# Увеличьте MAX_FILE_AGE в скрипте
MAX_FILE_AGE=300  # 5 часов

# Или используйте точное имя файла
BACKUP_PATTERN="mysql_backup.sql"
```

### Проблема: "NFS не примонтирован"

**Причина:** NFS директория не доступна

**Решение:**
```bash
# Проверьте монтирование (если скрипт на клиенте)
mountpoint /mnt/nfs-backup

# Проверьте существование директории (если на сервере)
ls -la /data/nfs/backup/mysql-vm/

# Проверьте NFS экспорты
sudo exportfs -v
sudo showmount -e localhost
```

### Проблема: "Файл слишком маленький"

**Причина:** Бэкап действительно неполный или порог слишком высокий

**Решение:**
```bash
# Проверьте размер оригинала
ls -lh /data/nfs/backup/mysql-vm/mysql_backup*

# Если ваша БД маленькая, измените порог в скрипте
if [ "$file_size" -lt 10240 ]; then  # 10KB вместо 100KB
```

### Проблема: Email не отправляются

**Причина:** mailx не установлен или не настроен

**Решение:**
```bash
# Установите mailx
sudo apt install mailx -y

# Проверьте отправку
echo "Test" | mail -s "Test Subject" admin@example.com

# Проверьте логи
sudo tail -f /var/log/mail.log
```

### Проблема: "Нечетное количество кавычек"

**Причина:** Файл оборван в середине или содержит экранированные кавычки

**Решение:**
```bash
# Проверьте конец файла
tail -50 /data/nfs/backup/mysql-vm/mysql_backup.sql

# Если файл целый но проверка ложная - это нормально
# Это дополнительная проверка, не критичная
```

---

## ❓ FAQ

### Можно ли запускать скрипт на MySQL сервере вместо NFS?

Да, но НЕ рекомендуется:
- ✅ Работает, если изменить `BACKUP_DIR` на примонтированный NFS путь
- ❌ Скрипт остановится если VM упадет
- ❌ Проверка не выполнится если MySQL недоступен

**Рекомендация:** Запускайте на стабильном NFS сервере.

### Как часто запускать скрипт?

**Рекомендации:**
- Если Webmin создает бэкап раз в день → запускайте через 1-2 часа после
- Если несколько бэкапов в день → запускайте после каждого
- Минимальный интервал: 1 час

### Сколько места займут копии?

```bash
Размер копий ≈ Размер оригинала × Количество дней хранения

Пример:
- Бэкап: 1GB
- Хранение: 30 дней
- Итого: ~30GB
```

**Настройка:** Измените `RETENTION_DAYS` в скрипте

### Можно ли проверять несколько баз данных?

Да! Создайте wrapper скрипт:

```bash
#!/bin/bash

# БД 1
BACKUP_DIR="/data/nfs/backup/db1"
ARCHIVE_DIR="/data/nfs/backup/db1/archive"
/usr/local/bin/mysql_backup_verify.sh

# БД 2
BACKUP_DIR="/data/nfs/backup/db2"
ARCHIVE_DIR="/data/nfs/backup/db2/archive"
/usr/local/bin/mysql_backup_verify.sh
```

### Нужно ли включать тест восстановления?

**Нет** для ежедневных проверок (медленно).

**Да** для критичных production бэкапов раз в неделю.

Раскомментируйте в скрипте:
```bash
if verify_sql_restore "$latest_backup"; then
    log_message "✓ Тест восстановления пройден"
fi
```

---

## 🤝 Участие в проекте

Мы приветствуем вклад в проект! 

### Как помочь

1. 🐛 **Сообщайте об ошибках** через [Issues](https://github.com/yourusername/mysql-backup-verify/issues)
2. 💡 **Предлагайте улучшения** через [Discussions](https://github.com/yourusername/mysql-backup-verify/discussions)
3. 🔧 **Отправляйте Pull Requests** с исправлениями
4. 📖 **Улучшайте документацию**
5. ⭐ **Ставьте звезды** проекту

### Процесс разработки

```bash
# 1. Fork репозитория
# 2. Клонирование вашего fork
git clone https://github.com/your-username/mysql-backup-verify.git
cd mysql-backup-verify

# 3. Создание ветки для фичи
git checkout -b feature/amazing-feature

# 4. Внесение изменений и коммит
git commit -m "Add amazing feature"

# 5. Push в ваш fork
git push origin feature/amazing-feature

# 6. Создание Pull Request
```

### Стиль кода

- Используйте `bash` (не `sh`)
- Отступы: 4 пробела
- Комментарии на русском или английском
- Тестируйте на CentOS 8 и Debian 10+

---

## 📄 Лицензия

Этот проект распространяется под лицензией MIT. См. файл [LICENSE](LICENSE) для деталей.

```
MIT License

Copyright (c) 2026 Your Name

Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction...
```

---

## 👥 Авторы

- **Your Name** - *Первоначальная разработка* - [@yourusername](https://github.com/yourusername)

См. также список [участников](https://github.com/yourusername/mysql-backup-verify/contributors) проекта.

---

## 🙏 Благодарности

- Проект вдохновлен необходимостью надежного резервного копирования в production
- Спасибо сообществу за тестирование и feedback
- Webmin за отличный инструмент управления MySQL

---

## 📞 Поддержка

- 📧 Email: support@example.com
- 💬 Telegram: [@your_telegram](https://t.me/your_telegram)
- 🐛 Issues: [GitHub Issues](https://github.com/yourusername/mysql-backup-verify/issues)
- 📖 Wiki: [GitHub Wiki](https://github.com/yourusername/mysql-backup-verify/wiki)

---

## 🔗 Связанные проекты

- [mysql-backup-auto-restore](https://github.com/yourusername/mysql-backup-auto-restore) - Автовосстановление MySQL при сбое
- [proxmox-mysql-guide](https://github.com/yourusername/proxmox-mysql-guide) - MySQL на Proxmox VE

---

**⭐ Если проект полезен - поставьте звезду на GitHub!**
