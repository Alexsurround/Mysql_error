# Восстановление MariaDB/MySQL при сбое

Руководство по диагностике и восстановлению MariaDB при повреждении InnoDB таблиц и проблемах с доступом.

## 📋 Содержание

- [Симптомы проблемы](#симптомы-проблемы)
- [Диагностика](#диагностика)
- [Сброс пароля root](#сброс-пароля-root)
- [Восстановление InnoDB](#восстановление-innodb)
- [Восстановление из дампа](#восстановление-из-дампа)
- [Типичные ошибки и решения](#типичные-ошибки-и-решения)

---

## 🚨 Симптомы проблемы

### MariaDB падает в цикле каждые 3-5 секунд:
```
InnoDB: Assertion failure in btr0cur.cc line 340
InnoDB: Failing assertion: btr_page_get_prev(get_block->frame)
InnoDB: corruption in the InnoDB tablespace
260607 17:32:14 [ERROR] mysqld got signal 6
```

### Ошибка сокета при запуске:
```
Socket file /var/lib/mysql/mysql.sock exists
Is another MySQL daemon already running?
```

### Ошибка доступа:
```
ERROR 1045 (28000): Access denied for user 'root'@'localhost'
```

---

## 🔍 Диагностика

### Проверка логов MariaDB
```bash
# Последние ошибки
sudo tail -100 /var/log/mariadb/mariadb.log

# Через systemd
sudo journalctl -u mariadb -n 100 --no-pager

# Поиск критических ошибок
sudo grep -i "error\|crash\|assertion\|corruption" /var/log/mariadb/mariadb.log | tail -50
```

### Определение поврежденной таблицы
```bash
# В логе ищите строку Query перед CRASH:
grep -A 2 "got signal 6" /var/log/mariadb/mariadb.log

# Пример из лога:
# Query: insert into history_uint (itemid,clock,ns,value) values (...)
# Значит повреждена таблица: history_uint
```

### Проверка статуса файловой системы
```bash
# Где хранится MySQL
df -h /var/lib/mysql
mount | grep mysql

# Ошибки диска
dmesg | grep -i "error\|i/o" | tail -20

# Статус файловой системы
sudo dumpe2fs /dev/YOUR_DEVICE | grep -i "state\|error"
```

---

## 🔐 Сброс пароля root

### Метод 1: mysqld_safe (рекомендуется)

```bash
# 1. Остановите MariaDB
sudo systemctl stop mariadb

# 2. Удалите старый сокет если есть
sudo rm -f /var/lib/mysql/mysql.sock
sudo rm -f /var/lib/mysql/mysql.sock.lock

# 3. Запустите без авторизации
sudo mysqld_safe --skip-grant-tables --skip-networking &
sleep 5

# 4. Подключитесь без пароля
mysql -u root

# 5. Смените пароль
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'НовыйПароль123!';
FLUSH PRIVILEGES;
EXIT;

# 6. Перезапустите нормально
sudo pkill mysqld_safe
sudo pkill mysqld
sleep 3
sudo systemctl start mariadb

# 7. Проверьте
mysql -u root -p'НовыйПароль123!'
```

### Метод 2: init-file

```bash
# 1. Создайте файл с командой
sudo tee /tmp/resetpass.sql << 'EOF'
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY 'НовыйПароль123!';
FLUSH PRIVILEGES;
EOF

# 2. Остановите MariaDB
sudo systemctl stop mariadb

# 3. Запустите с файлом инициализации
sudo mysqld --user=mysql \
            --skip-grant-tables \
            --init-file=/tmp/resetpass.sql &
sleep 5

# 4. Остановите и запустите нормально
sudo pkill mysqld
sleep 3
sudo systemctl start mariadb

# 5. Проверьте новый пароль
mysql -u root -p'НовыйПароль123!'

# 6. ВАЖНО: Удалите файл с паролем!
sudo rm -f /tmp/resetpass.sql
```

### Метод 3: через sudo (быстрый)

```bash
# Иногда работает без пароля через unix socket
sudo mysql -u root

# Внутри MySQL:
ALTER USER 'root'@'localhost' IDENTIFIED BY 'НовыйПароль123!';
FLUSH PRIVILEGES;
EXIT;
```

---

## 🔧 Восстановление InnoDB

### Шаг 1: Определите уровень повреждения

```bash
sudo tail -50 /var/log/mariadb/mariadb.log | grep -i "corruption\|assertion\|error"
```

### Шаг 2: Включите режим восстановления

```bash
sudo nano /etc/my.cnf.d/mariadb-server.cnf
```

Добавьте в секцию `[mysqld]`:
```ini
[mysqld]
innodb_force_recovery = 1
```

**Уровни восстановления (пробуйте по порядку):**

| Уровень | Описание | Когда использовать |
|---------|----------|-------------------|
| 1 | Игнорировать поврежденные страницы | Первая попытка |
| 2 | Отключить фоновые потоки | Если 1 не помог |
| 3 | Не запускать откат транзакций | Если 2 не помог |
| 4 | Не вычислять статистику | Если 3 не помог |
| 5 | Не использовать undo логи | Крайний случай |
| 6 | Не выполнять откат при запуске | Последний шанс |

### Шаг 3: Запустите MariaDB

```bash
sudo systemctl start mariadb
sudo systemctl status mariadb
```

### Шаг 4: Если запустилась - СРАЗУ создайте дамп!

```bash
# Дамп всех баз
mysqldump -u root -p --all-databases > /tmp/full_backup_$(date +%Y%m%d_%H%M%S).sql

# Дамп только нужной базы
mysqldump -u root -p zabbix > /tmp/zabbix_$(date +%Y%m%d_%H%M%S).sql
```

### Шаг 5: Починка конкретной таблицы

```bash
mysql -u root -p zabbix
```

```sql
-- Посмотрите структуру поврежденной таблицы
SHOW CREATE TABLE history_uint;

-- Переименуйте поврежденную
RENAME TABLE history_uint TO history_uint_broken;

-- Создайте новую чистую
CREATE TABLE history_uint LIKE history_uint_broken;

-- Проверьте
SHOW TABLES LIKE 'history%';

EXIT;
```

### Шаг 6: Уберите innodb_force_recovery

```bash
sudo nano /etc/my.cnf.d/mariadb-server.cnf
# Удалите или закомментируйте строку:
# innodb_force_recovery = 1

sudo systemctl restart mariadb
```

---

## 💾 Восстановление из дампа

### Вариант 1: Восстановление только базы Zabbix (рекомендуется)

```bash
# 1. Остановите Zabbix
sudo systemctl stop zabbix-server

# 2. Подключитесь к MySQL
mysql -u root -p

# 3. Пересоздайте базу
DROP DATABASE zabbix;
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EXIT;

# 4. Восстановите из дампа
mysql -u root -p zabbix < /path/to/zabbix_backup.sql

# Если дамп сжат:
gunzip -c /path/to/zabbix_backup.sql.gz | mysql -u root -p zabbix

# 5. Запустите Zabbix
sudo systemctl start zabbix-server
sudo systemctl status zabbix-server
```

### Вариант 2: Полная переинициализация MySQL

```bash
# 1. Остановите всё
sudo systemctl stop zabbix-server
sudo systemctl stop mariadb

# 2. Сохраните поврежденные данные
sudo mv /var/lib/mysql /var/lib/mysql.broken_$(date +%Y%m%d_%H%M%S)

# 3. Создайте чистую директорию
sudo mkdir -p /var/lib/mysql
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 755 /var/lib/mysql

# 4. Инициализируйте новую базу
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql

# 5. Запустите MariaDB
sudo systemctl start mariadb

# 6. Настройка безопасности
sudo mysql_secure_installation

# 7. Создайте базу и пользователя Zabbix
mysql -u root -p << EOF
CREATE DATABASE zabbix CHARACTER SET utf8mb4 COLLATE utf8mb4_bin;
CREATE USER 'zabbix'@'localhost' IDENTIFIED BY 'zabbix_password';
GRANT ALL PRIVILEGES ON zabbix.* TO 'zabbix'@'localhost';
FLUSH PRIVILEGES;
EOF

# 8. Восстановите из дампа
mysql -u root -p zabbix < /path/to/zabbix_backup.sql

# 9. Запустите Zabbix
sudo systemctl start zabbix-server
sudo systemctl status zabbix-server
```

---

## ⚠️ Типичные ошибки и решения

### Ошибка: Socket file exists

```
Socket file /var/lib/mysql/mysql.sock exists
Is another MySQL daemon already running?
```

**Решение:**
```bash
# Убить все процессы MySQL
sudo pkill -9 mysqld
sudo pkill -9 mysqld_safe

# Удалить старый сокет
sudo rm -f /var/lib/mysql/mysql.sock
sudo rm -f /var/lib/mysql/mysql.sock.lock

# Запустить заново
sudo systemctl start mariadb
```

---

### Ошибка: Access denied for root

```
ERROR 1045 (28000): Access denied for user 'root'@'localhost'
```

**Решение:** См. раздел [Сброс пароля root](#сброс-пароля-root)

---

### Ошибка: InnoDB Assertion failure

```
InnoDB: Assertion failure in btr0cur.cc
InnoDB: corruption in the InnoDB tablespace
```

**Решение:** См. раздел [Восстановление InnoDB](#восстановление-innodb)

---

### Ошибка: Can't connect to socket

```
Can't connect to local MySQL server through socket '/var/lib/mysql/mysql.sock' (2)
```

**Решение:**
```bash
# Проверьте статус
sudo systemctl status mariadb

# Если не запущен - запустите
sudo systemctl start mariadb

# Если не запускается - проверьте логи
sudo journalctl -u mariadb -n 50
```

---

### Ошибка: mysql_install_db падает

```
Installation of system tables failed!
```

**Решение:**
```bash
# Полностью очистите директорию
sudo rm -rf /var/lib/mysql/*

# Проверьте права
sudo chown mysql:mysql /var/lib/mysql
sudo chmod 755 /var/lib/mysql

# Попробуйте снова
sudo mysql_install_db --user=mysql --datadir=/var/lib/mysql
```

---

## 📝 Чек-лист восстановления

- [ ] Проверили логи: `journalctl -u mariadb -n 100`
- [ ] Определили тип ошибки (InnoDB/сокет/пароль)
- [ ] Остановили MariaDB: `systemctl stop mariadb`
- [ ] Убили все процессы: `pkill -9 mysqld`
- [ ] Удалили старый сокет: `rm -f /var/lib/mysql/mysql.sock`
- [ ] Восстановили доступ (сбросили пароль если нужно)
- [ ] Включили `innodb_force_recovery` если InnoDB повреждена
- [ ] Создали дамп существующих данных
- [ ] Восстановили из резервного дампа
- [ ] Убрали `innodb_force_recovery` из конфига
- [ ] Перезапустили MariaDB и проверили статус
- [ ] Запустили зависимые сервисы (Zabbix и т.д.)

---

## 🔗 Полезные ссылки

- [MariaDB InnoDB Recovery Modes](https://mariadb.com/kb/en/innodb-recovery-modes/)
- [MariaDB Crash Recovery](https://mariadb.com/kb/en/xtrabackup-overview/)
- [Reporting MariaDB Bugs](https://jira.mariadb.org/)

---

## 📄 Лицензия

MIT License - свободное использование

---

**Последнее обновление:** 2026-06-07
