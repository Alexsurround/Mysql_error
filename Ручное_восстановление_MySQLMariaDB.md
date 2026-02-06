# Ручное восстановление MySQL/MariaDB

Пошаговое руководство по восстановлению MySQL/MariaDB при сбое базы данных.

## 📋 Содержание

- [Типичные ошибки](#типичные-ошибки)
- [Диагностика проблемы](#диагностика-проблемы)
- [Процедура восстановления](#процедура-восстановления)
- [Восстановление из backup](#восстановление-из-backup)
- [Установка пароля root](#установка-пароля-root)
- [Проверка работоспособности](#проверка-работоспособности)

---

## 🔍 Типичные ошибки

### Ошибка 111 - Connection refused
```
Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (111)
```
**Причина:** MySQL не запущен

---

### Ошибка 2 - No such file or directory
```
Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)
```
**Причина:** MySQL не запущен или сокет-файл отсутствует

---

### Ошибка InnoDB corruption
```
InnoDB: Aborting because of a corrupt database page
Missing MLOG_CHECKPOINT
```
**Причина:** Поврежденные файлы InnoDB

---

## 🔬 Диагностика проблемы

### Шаг 1: Проверка статуса службы

```bash
# Проверка статуса MySQL/MariaDB
sudo systemctl status mariadb

# Или для MySQL
sudo systemctl status mysqld
```

**Возможные результаты:**

**A) Служба остановлена:**
```
Active: inactive (dead)
```
→ Переходите к [Запуск MySQL](#запуск-mysql)

**B) Служба падает при запуске:**
```
Active: failed (Result: exit-code)
```
→ Переходите к [Проверка логов](#шаг-2-проверка-логов)

**C) Служба перезапускается постоянно:**
```
Active: activating (auto-restart)
```
→ Переходите к [Проверка логов](#шаг-2-проверка-логов)

---

### Шаг 2: Проверка логов

```bash
# Просмотр последних ошибок MariaDB
sudo tail -50 /var/log/mariadb/mariadb.log

# Или через journalctl
sudo journalctl -u mariadb -n 100 --no-pager

# Для MySQL
sudo tail -50 /var/log/mysqld.log
```

**Что искать в логах:**

**Поврежденная БД:**
```
InnoDB: Aborting because of a corrupt database page
Missing MLOG_CHECKPOINT
```
→ Нужна [полная переинициализация](#полная-переинициализация)

**Проблемы с правами:**
```
Can't create/write to file
Permission denied
```
→ Исправьте права: `sudo chown -R mysql:mysql /var/lib/mysql`

**Нехватка места:**
```
No space left on device
```
→ Освободите место: `df -h`

---

### Шаг 3: Проверка файлов данных

```bash
# Проверка содержимого директории MySQL
ls -la /var/lib/mysql/

# Проверка прав доступа
ls -ld /var/lib/mysql/

# Должно быть:
# drwxr-xr-x. mysql mysql /var/lib/mysql/
```

---

## 🔧 Процедура восстановления

### Вариант 1: Простой перезапуск (если нет повреждений)

#### Запуск MySQL

```bash
# Остановка (если запущен)
sudo systemctl stop mariadb

# Запуск
sudo systemctl start mariadb

# Проверка статуса
sudo systemctl status mariadb

# Включение автозапуска
sudo systemctl enable mariadb
```

**Проверка сокета:**
```bash
ls -la /var/lib/mysql/mysql.sock

# Должно показать:
# srwxrwxrwx. 1 mysql mysql 0 ... /var/lib/mysql/mysql.sock
```

**Если запустилось успешно** → Переходите к [Проверке работоспособности](#проверка-работоспособности)

---

### Вариант 2: Полная переинициализация

**⚠️ ВНИМАНИЕ: Это удалит все данные! Убедитесь, что у вас есть backup!**

#### Шаг 1: Остановка MySQL

```bash
sudo systemctl stop mariadb

# Убедитесь что процесс остановлен
ps aux | grep mysql
```

#### Шаг 2: Резервное копирование старых данных

```bash
# Создаем backup старой директории (на всякий случай)
sudo mv /var/lib/mysql /var/lib/mysql.broken_$(date +%Y%m%d_%H%M%S)

# Проверяем что директория перемещена
ls -la /var/lib/ | grep mysql
```

#### Шаг 3: Создание чистой директории

```bash
# Создаем новую директорию
sudo mkdir -p /var/lib/mysql

# Устанавливаем правильные права
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 755 /var/lib/mysql

# Проверяем права
ls -ld /var/lib/mysql/
# Должно быть: drwxr-xr-x. mysql mysql
```

#### Шаг 4: Инициализация новой базы данных

```bash
# Инициализация
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql

# Проверка успешности (в конце вывода должно быть):
# To start mysqld at boot time you have to copy
# support-files/mysql.server to the right place for your system
```

**Если ошибка при инициализации:**
```bash
# Удалите всё и попробуйте снова
sudo rm -rf /var/lib/mysql/*
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql
```

#### Шаг 5: Запуск MySQL

```bash
# Запуск службы
sudo systemctl start mariadb

# Проверка статуса
sudo systemctl status mariadb

# Должно показать:
# Active: active (running)
```

#### Шаг 6: Проверка файлов

```bash
# Проверяем что создались файлы
ls -la /var/lib/mysql/

# Должны быть файлы:
# mysql/ (системная БД)
# ib_logfile0, ib_logfile1 (логи InnoDB)
# ibdata1 (данные InnoDB)
# mysql.sock (сокет)
```

---

### Вариант 3: Восстановление с режимом InnoDB recovery

**Если MySQL не запускается из-за InnoDB ошибок, но вы хотите попробовать извлечь данные:**

#### Шаг 1: Остановка MySQL

```bash
sudo systemctl stop mariadb
```

#### Шаг 2: Редактирование конфигурации

```bash
sudo nano /etc/my.cnf.d/mariadb-server.cnf

# Или для MySQL:
sudo nano /etc/my.cnf
```

#### Шаг 3: Добавление режима восстановления

Добавьте в секцию `[mysqld]`:
```ini
[mysqld]
innodb_force_recovery = 1
```

**Уровни восстановления:**
- `1` - игнорировать поврежденные страницы (начните с этого)
- `2` - предотвратить запуск фоновых потоков
- `3` - не запускать откат транзакций
- `4` - не вычислять статистику
- `5` - не использовать undo логи
- `6` - не выполнять откат транзакций при запуске

#### Шаг 4: Попытка запуска

```bash
sudo systemctl start mariadb
sudo systemctl status mariadb
```

#### Шаг 5: Если запустилось - создайте дамп

```bash
# Быстро создайте бэкап ВСЕХ баз
mysqldump -u root -p --all-databases > /tmp/all_databases_recovery.sql

# Или конкретной базы
mysqldump -u root -p your_database > /tmp/your_database_recovery.sql
```

#### Шаг 6: Полная переинициализация

```bash
# Остановите MySQL
sudo systemctl stop mariadb

# Удалите строку innodb_force_recovery из конфига
sudo nano /etc/my.cnf.d/mariadb-server.cnf

# Выполните полную переинициализацию (Вариант 2)
sudo mv /var/lib/mysql /var/lib/mysql.backup
sudo mkdir -p /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql
sudo systemctl start mariadb
```

#### Шаг 7: Восстановление из дампа

```bash
mysql -u root -p < /tmp/all_databases_recovery.sql
```

---

## 💾 Восстановление из backup

### После успешного запуска MySQL

#### Шаг 1: Установка пароля root

См. раздел [Установка пароля root](#установка-пароля-root)

#### Шаг 2: Создание базы данных

```bash
# Войдите в MySQL
mysql -u root -p

# Создайте базу данных
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# Проверьте
SHOW DATABASES;

# Выход
EXIT;
```

#### Шаг 3: Восстановление из SQL файла

```bash
# Восстановление конкретной базы
mysql -u root -p zabbix < /path/to/backup.sql

# Если backup сжат (gzip)
gunzip < /path/to/backup.sql.gz | mysql -u root -p zabbix

# Если backup всех баз данных
mysql -u root -p < /path/to/all_databases_backup.sql
```

#### Шаг 4: Проверка восстановленных данных

```bash
mysql -u root -p

# Выберите базу
USE zabbix;

# Посмотрите таблицы
SHOW TABLES;

# Проверьте количество записей в важных таблицах
SELECT COUNT(*) FROM table_name;

EXIT;
```

---

## 🔐 Установка пароля root

### Метод 1: Через mysql_secure_installation (Рекомендуется)

```bash
sudo mysql_secure_installation
```

**Ответы:**
```
Enter current password for root: [Enter - пароля нет]

Switch to unix_socket authentication [Y/n] n
Set root password? [Y/n] Y
New password: [ваш_надежный_пароль]
Re-enter new password: [повторите_пароль]

Remove anonymous users? [Y/n] Y
Disallow root login remotely? [Y/n] Y
Remove test database? [Y/n] Y
Reload privilege tables now? [Y/n] Y
```

---

### Метод 2: Вручную через SQL

```bash
# Войдите в MySQL без пароля
sudo mysql -u root

# Установите пароль
ALTER USER 'root'@'localhost' IDENTIFIED BY 'ВашНадежныйПароль123!';

# Примените изменения
FLUSH PRIVILEGES;

# Выход
EXIT;
```

---

### Метод 3: Через mysqladmin

```bash
# Установка пароля (если пароля нет)
sudo mysqladmin -u root password 'НовыйПароль123!'

# Смена существующего пароля
sudo mysqladmin -u root -p'СтарыйПароль' password 'НовыйПароль123!'
```

---

### Сброс забытого пароля

```bash
# 1. Остановите MySQL
sudo systemctl stop mariadb

# 2. Запустите без проверки паролей
sudo mysqld_safe --skip-grant-tables &

# 3. Подключитесь без пароля (в другом терминале)
mysql -u root

# 4. Смените пароль
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'НовыйПароль123!';
EXIT;

# 5. Остановите mysqld_safe
sudo pkill mysqld

# 6. Запустите MySQL нормально
sudo systemctl start mariadb
```

---

## ✅ Проверка работоспособности

### Шаг 1: Проверка подключения

```bash
# Подключение с паролем
mysql -u root -p

# Должны увидеть:
# Welcome to the MariaDB monitor...
# MariaDB [(none)]>
```

### Шаг 2: Базовые проверки

```sql
-- Проверка версии
SELECT VERSION();

-- Список баз данных
SHOW DATABASES;

-- Проверка пользователей
SELECT User, Host FROM mysql.user;

-- Проверка переменных
SHOW VARIABLES LIKE 'datadir';
SHOW VARIABLES LIKE 'socket';

-- Выход
EXIT;
```

### Шаг 3: Проверка статуса службы

```bash
# Статус службы
sudo systemctl status mariadb

# Проверка автозапуска
sudo systemctl is-enabled mariadb

# Если не включен автозапуск
sudo systemctl enable mariadb
```

### Шаг 4: Проверка сокета и портов

```bash
# Проверка сокета
ls -la /var/lib/mysql/mysql.sock

# Проверка что MySQL слушает порт 3306
sudo netstat -tlnp | grep 3306

# Или через ss
sudo ss -tlnp | grep 3306

# Должно показать:
# tcp  0  0  127.0.0.1:3306  0.0.0.0:*  LISTEN  12345/mysqld
```

### Шаг 5: Проверка производительности

```bash
mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'Uptime';"
mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'Questions';"
mysql -u root -p -e "SHOW GLOBAL STATUS LIKE 'Threads_connected';"
```

---

## 🚨 Скрипт быстрого восстановления

### Скрипт "одной кнопкой"

Сохраните как `/root/mysql_emergency_restore.sh`:

```bash
#!/bin/bash

echo "=========================================="
echo "MySQL/MariaDB Emergency Restore Script"
echo "=========================================="
echo ""
echo "WARNING: This will DELETE all current MySQL data!"
read -p "Do you have a backup? (yes/no): " confirm

if [ "$confirm" != "yes" ]; then
    echo "Operation cancelled. Create a backup first!"
    exit 1
fi

echo ""
echo "[1/7] Stopping MySQL..."
sudo systemctl stop mariadb

echo "[2/7] Backing up old data..."
if [ -d "/var/lib/mysql" ]; then
    sudo mv /var/lib/mysql /var/lib/mysql.broken_$(date +%Y%m%d_%H%M%S)
    echo "Old data moved to: /var/lib/mysql.broken_*"
fi

echo "[3/7] Creating clean directory..."
sudo mkdir -p /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 755 /var/lib/mysql

echo "[4/7] Initializing new database..."
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql

if [ $? -ne 0 ]; then
    echo "ERROR: Database initialization failed!"
    exit 1
fi

echo "[5/7] Starting MySQL..."
sudo systemctl start mariadb
sleep 3

echo "[6/7] Checking status..."
sudo systemctl status mariadb --no-pager

echo "[7/7] Checking socket..."
if [ -S "/var/lib/mysql/mysql.sock" ]; then
    echo "✓ Socket created successfully"
else
    echo "✗ Socket NOT created - check logs!"
    exit 1
fi

echo ""
echo "=========================================="
echo "MySQL restored successfully!"
echo "=========================================="
echo ""
echo "Next steps:"
echo "1. Set root password: sudo mysql_secure_installation"
echo "2. Create database: mysql -u root -p"
echo "3. Restore backup: mysql -u root -p database < backup.sql"
echo ""
```

**Использование:**
```bash
# Сделайте скрипт исполняемым
chmod +x /root/mysql_emergency_restore.sh

# Запустите
sudo /root/mysql_emergency_restore.sh
```

---

## 📝 Чек-лист восстановления

- [ ] Проверили статус MySQL: `systemctl status mariadb`
- [ ] Проверили логи: `tail -50 /var/log/mariadb/mariadb.log`
- [ ] Сохранили старые данные: `mv /var/lib/mysql /var/lib/mysql.backup`
- [ ] Создали чистую директорию с правами mysql:mysql
- [ ] Инициализировали новую БД: `mysql_install_db`
- [ ] Запустили MySQL: `systemctl start mariadb`
- [ ] Проверили сокет: `ls -la /var/lib/mysql/mysql.sock`
- [ ] Установили пароль root: `mysql_secure_installation`
- [ ] Создали базу данных: `CREATE DATABASE ...`
- [ ] Восстановили backup: `mysql < backup.sql`
- [ ] Проверили данные: `SELECT COUNT(*) FROM ...`
- [ ] Включили автозапуск: `systemctl enable mariadb`

---

## 🔧 Полезные команды

### Управление службой
```bash
sudo systemctl start mariadb      # Запуск
sudo systemctl stop mariadb       # Остановка
sudo systemctl restart mariadb    # Перезапуск
sudo systemctl status mariadb     # Статус
sudo systemctl enable mariadb     # Автозапуск
```

### Проверка логов
```bash
sudo tail -f /var/log/mariadb/mariadb.log    # Реальное время
sudo journalctl -u mariadb -f                # Через systemd
sudo grep -i error /var/log/mariadb/mariadb.log  # Только ошибки
```

### Работа с базами
```bash
mysql -u root -p                              # Подключение
mysqldump -u root -p database > backup.sql   # Создание бэкапа
mysql -u root -p database < backup.sql       # Восстановление
mysqlcheck -u root -p --all-databases         # Проверка всех БД
```

---

## 📞 Поддержка

Если проблема не решается:

1. Сохраните полный лог: `sudo journalctl -u mariadb > mysql_error.log`
2. Создайте issue на GitHub с логом
3. Укажите версию ОС: `cat /etc/os-release`
4. Укажите версию MySQL: `mysql --version`

---

## 📄 Лицензия

MIT License - свободное использование

---

**Последнее обновление:** 2026-02-06
