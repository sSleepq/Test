
# Руководство по настройке веб-сервера Nginx на Ubuntu Server

## Оглавление
1. [Обновление системы](#обновление-системы)
2. [Установка Nginx](#установка-nginx)
3. [Управление сервисом Nginx](#управление-сервисом-nginx)
4. [Настройка брандмауэра (UFW)](#настройка-брандмауэра-ufw)
5. [Структура конфигурационных файлов](#структура-конфигурационных-файлов)
6. [Создание и настройка серверного блока (виртуального хоста)](#создание-и-настройка-серверного-блока-виртуального-хоста)
7. [Настройка статического контента](#настройка-статического-контента)
8. [Настройка PHP с PHP-FPM](#настройка-php-с-php-fpm)
9. [Настройка SSL/TLS с помощью Let's Encrypt](#настройка-ssltls-с-помощью-lets-encrypt)
10. [Полезные команды и диагностика](#полезные-команды-и-диагностика)

---

## Обновление системы

Перед началом работы обновите индекс пакетов и установите последние обновления:

```bash
sudo apt update
sudo apt upgrade -y
```

## Установка Nginx

Nginx доступен в стандартных репозиториях Ubuntu. Установите его командой:

```bash
sudo apt install nginx -y
```

## Управление сервисом Nginx

После установки Nginx будет автоматически запущен. Вот основные команды для управления сервисом:

* **Проверить статус:**
  ```bash
  sudo systemctl status nginx
  ```

* **Запустить Nginx:**
  ```bash
  sudo systemctl start nginx
  ```

* **Остановить Nginx:**
  ```bash
  sudo systemctl stop nginx
  ```

* **Перезапустить Nginx** (применяет изменения конфигурации):
  ```bash
  sudo systemctl restart nginx
  ```

* **Перезагрузить Nginx** (более мягкий способ, без разрыва соединений):
  ```bash
  sudo systemctl reload nginx
  ```

* **Настроить автозапуск при загрузке системы:**
  ```bash
  sudo systemctl enable nginx
  ```

## Настройка брандмауэра (UFW)

Если у вас включен брандмауэр `ufw`, необходимо разрешить трафик на HTTP (порт 80) и HTTPS (порт 443).

1. Посмотрите список доступных профилей:
   ```bash
   sudo ufw app list
   ```
   Вы должны увидеть `Nginx Full`, `Nginx HTTP` и `Nginx HTTPS`.

2. Разрешите подключения. Рекомендуется сразу настроить `Nginx Full` для работы с HTTP и HTTPS:
   ```bash
   sudo ufw allow 'Nginx Full'
   ```

3. Если вам нужен только HTTP на данном этапе:
   ```bash
   sudo ufw allow 'Nginx HTTP'
   ```

4. Проверьте статус UFW:
   ```bash
   sudo ufw status
   ```

## Структура конфигурационных файлов

* **Основной конфигурационный файл:** `/etc/nginx/nginx.conf`
* **Файлы конфигурации серверных блоков (виртуальных хостов):** `/etc/nginx/sites-available/`
* **Активные серверные блоки (симлинки на `sites-available`):** `/etc/nginx/sites-enabled/`
* **Корневая директория веб-документов по умолчанию:** `/var/www/html`
* **Файлы логов:**
  * Access log: `/var/log/nginx/access.log`
  * Error log: `/var/log/nginx/error.log`

## Создание и настройка серверного блока (виртуального хоста)

Серверные блоки позволяют размещать несколько сайтов на одном сервере.

1. **Создайте директорию для вашего сайта.** Замените `your_domain` на ваше доменное имя или название проекта.
   ```bash
   sudo mkdir -p /var/www/your_domain/html
   ```

2. **Назначьте права владельца.**
   ```bash
   sudo chown -R $USER:$USER /var/www/your_domain/html
   sudo chmod -R 755 /var/www/your_domain
   ```

3. **Создайте тестовую HTML-страницу.**
   ```bash
   nano /var/www/your_domain/html/index.html
   ```
   Добавьте следующий примерный код:
   ```html
   <html>
       <head>
           <title>Добро пожаловать на your_domain!</title>
       </head>
       <body>
           <h1>Успех! Сервер your_domain работает правильно.</h1>
       </body>
   </html>
   ```

4. **Создайте файл конфигурации серверного блока в `sites-available`.**
   ```bash
   sudo nano /etc/nginx/sites-available/your_domain
   ```

5. **Добавьте базовую конфигурацию в файл.** Замените `your_domain` на ваше доменное имя и, при необходимости, `127.0.0.1` на IP-адрес вашего сервера.
   ```nginx
   server {
       listen 80;
       listen [::]:80; # IPv6

       server_name your_domain www.your_domain;

       root /var/www/your_domain/html;
       index index.html index.htm index.nginx-debian.html;

       location / {
           try_files $uri $uri/ =404;
       }
   }
   ```

6. **Активируйте серверный блок,** создав символическую ссылку в директории `sites-enabled`.
   ```bash
   sudo ln -s /etc/nginx/sites-available/your_domain /etc/nginx/sites-enabled/
   ```

7. **Проверьте конфигурацию Nginx на наличие ошибок.**
   ```bash
   sudo nginx -t
   ```
   Если вывод показывает `successful`, конфигурация корректна.

8. **Перезагрузите Nginx для применения изменений.**
   ```bash
   sudo systemctl reload nginx
   ```

Теперь ваш сайт должен быть доступен по доменному имени или IP-адресу сервера.

## Настройка PHP с PHP-FPM

Если ваш сайт использует PHP, необходимо установить и настроить PHP-FPM.

1. **Установите PHP и PHP-FPM:**
   ```bash
   sudo apt install php-fpm php-mysql -y
   ```

2. **В файле конфигурации вашего серверного блока (`/etc/nginx/sites-available/your_domain`) отредактируйте блок `location` для обработки PHP-файлов.**

   Замените стандартный блок `location / { ... }` или добавьте новый блок `location ~ \.php$`:
   ```nginx
   server {
       ... # предыдущие настройки (listen, server_name, root)

       index index.php index.html index.htm; # Добавьте index.php в начало

       location / {
           try_files $uri $uri/ =404;
       }

       location ~ \.php$ {
           include snippets/fastcgi-php.conf;
           fastcgi_pass unix:/var/run/php/php8.1-fpm.sock; # Убедитесь, что версия PHP совпадает!
       }

       location ~ /\.ht {
           deny all;
       }
   }
   ```
   *Важно:* Укажите правильную версию PHP в `fastcgi_pass` (например, `php8.1-fpm`, `php8.2-fpm`). Узнать версию можно командой `php -v`.

3. **Создайте тестовый PHP-файл для проверки:**
   ```bash
   nano /var/www/your_domain/html/info.php
   ```
   Добавьте код:
   ```php
   <?php
   phpinfo();
   ?>
   ```

4. **Проверьте конфигурацию и перезагрузите Nginx:**
   ```bash
   sudo nginx -t
   sudo systemctl reload nginx
   ```

5. **Перейдите в браузере на `http://your_domain/info.php`.** Вы должны увидеть страницу с информацией о PHP. **Удалите этот файл после проверки из соображений безопасности:**
   ```bash
   sudo rm /var/www/your_domain/html/info.php
   ```

## Настройка SSL/TLS с помощью Let's Encrypt

Для получения бесплатного SSL-сертификата используйте Certbot.

1. **Установите Certbot и плагин для Nginx:**
   ```bash
   sudo apt install certbot python3-certbot-nginx -y
   ```

2. **Убедитесь, что в конфигурации Nginx правильно указан `server_name`** (ваш домен).

3. **Получите и установите SSL-сертификат.** Certbot автоматически изменит конфигурацию Nginx.
   ```bash
   sudo certbot --nginx -d your_domain -d www.your_domain
   ```

4. **Сертификат будет автоматически обновляться.** Вы можете проверить автоматическое обновление командой:
   ```bash
   sudo certbot renew --dry-run
   ```

После успешной настройки ваш сайт будет доступен по защищенному протоколу HTTPS.

## Полезные команды и диагностика

* **Проверка синтаксиса конфигурации:**
  ```bash
  sudo nginx -t
  ```

* **Просмотр логов в реальном времени:**
  ```bash
  sudo tail -f /var/log/nginx/error.log
  sudo tail -f /var/log/nginx/access.log
  ```

* **Проверка, какие процессы слушают порты:**
  ```bash
  sudo ss -tulpn | grep nginx
  ```

* **Перечитать конфигурацию без остановки сервера:**
  ```bash
  sudo nginx -s reload
  ```

Теперь у вас есть полностью функционирующий веб-сервер на Nginx!
