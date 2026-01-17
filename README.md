# Инструкция по установке системы автоматического восстановления MySQL

## Шаг 1: Создайте скрипт восстановления

```bash
sudo nano /usr/local/bin/mysql_auto_restore.sh
```

Вставьте содержимое из первого артефакта и **ОБЯЗАТЕЛЬНО измените** следующие параметры:

```bash
BACKUP_FILE="/path/to/your/backup.sql"          # Ваш путь к бэкапу!
DATABASE_NAME="your_database_name"               # Имя вашей базы!
MYSQL_ROOT_PASSWORD="your_root_password"         # Пароль root MySQL!
EMAIL_ALERT="admin@example.com"                  # Ваш email (опционально)
```

Сохраните файл (Ctrl+O, Enter, Ctrl+X).

## Шаг 2: Сделайте скрипт исполняемым

```bash
sudo chmod +x /usr/local/bin/mysql_auto_restore.sh
```

## Шаг 3: Создайте systemd service

```bash
sudo nano /etc/systemd/system/mysql-monitor.service
```

Вставьте:

```ini
[Unit]
Description=MySQL Auto-Restore Monitor Service
After=mariadb.service
Wants=mariadb.service

[Service]
Type=oneshot
ExecStart=/usr/local/bin/mysql_auto_restore.sh
StandardOutput=journal
StandardError=journal

[Install]
WantedBy=multi-user.target
```

## Шаг 4: Создайте systemd timer

```bash
sudo nano /etc/systemd/system/mysql-monitor.timer
```

Вставьте:

```ini
[Unit]
Description=MySQL Auto-Restore Monitor Timer
Requires=mysql-monitor.service

[Timer]
OnBootSec=2min
OnUnitActiveSec=5min

[Install]
WantedBy=timers.target
```

## Шаг 5: Активируйте мониторинг

```bash
# Перезагрузите конфигурацию systemd
sudo systemctl daemon-reload

# Включите таймер
sudo systemctl enable mysql-monitor.timer

# Запустите таймер
sudo systemctl start mysql-monitor.timer

# Проверьте статус
sudo systemctl status mysql-monitor.timer
```

## Шаг 6: Проверка работы

```bash
# Проверьте, когда следующая проверка
sudo systemctl list-timers mysql-monitor.timer

# Посмотрите логи
sudo journalctl -u mysql-monitor.service -f

# Или в файле лога
sudo tail -f /var/log/mysql_auto_restore.log
```

## Тестирование системы восстановления

⚠️ **ВНИМАНИЕ: Это остановит MySQL!**

```bash
# Остановите MySQL для теста
sudo systemctl stop mariadb

# Вручную запустите скрипт восстановления
sudo /usr/local/bin/mysql_auto_restore.sh

# Проверьте, что всё восстановилось
sudo systemctl status mariadb
mysql -u root -p -e "SHOW DATABASES;"
```

## Настройка частоты проверки

По умолчанию проверка каждые 5 минут. Чтобы изменить:

```bash
sudo nano /etc/systemd/system/mysql-monitor.timer
```

Измените строку `OnUnitActiveSec=5min` на нужное значение:
- `OnUnitActiveSec=1min` - каждую минуту
- `OnUnitActiveSec=10min` - каждые 10 минут
- `OnUnitActiveSec=1h` - каждый час

После изменения:

```bash
sudo systemctl daemon-reload
sudo systemctl restart mysql-monitor.timer
```

## Отключение автовосстановления

```bash
sudo systemctl stop mysql-monitor.timer
sudo systemctl disable mysql-monitor.timer
```

## Просмотр истории восстановлений

```bash
# В логе скрипта
sudo cat /var/log/mysql_auto_restore.log

# В системных логах
sudo journalctl -u mysql-monitor.service --since "1 day ago"
```

## Email уведомления (опционально)

Для работы email-уведомлений установите mailx:

```bash
sudo dnf install mailx -y
```

И настройте отправку почты через SMTP в `/etc/mail.rc`.

## Резервные копии поврежденных данных

Скрипт автоматически сохраняет поврежденные данные в:
```
/var/lib/mysql.broken.YYYYMMDD_HHMMSS/
```

Можно удалять старые копии вручную или настроить автоочистку через cron.

## Рекомендации

1. **Регулярно обновляйте бэкап** - скрипт восстанавливает данные из указанного файла
2. **Тестируйте бэкапы** - периодически проверяйте, что бэкап можно восстановить
3. **Мониторьте логи** - проверяйте `/var/log/mysql_auto_restore.log`
4. **Настройте алерты** - используйте email или интеграцию с системой мониторинга

## Интеграция с Zabbix (бонус)

Добавьте UserParameter в Zabbix для мониторинга восстановлений:

```bash
sudo nano /etc/zabbix/zabbix_agentd.d/mysql_monitor.conf
```

```ini
UserParameter=mysql.last_restore_status,grep "завершено успешно" /var/log/mysql_auto_restore.log | tail -1 | wc -l
UserParameter=mysql.restore_count,grep "Запуск скрипта" /var/log/mysql_auto_restore.log | wc -l
```

Перезапустите агента:
```bash
sudo systemctl restart zabbix-agent
```
