### **Общая цель**

Настроить два сервера с панелью управления прокси-соединениями 3X-UI, защитить их с помощью SSL-сертификатов от Let's Encrypt и настроить проксирование трафика.

---

### **Часть 1: Настройка первого сервера (Санкт-Петербург)**

На этом этапе все действия производятся на первом сервере с тестовым доменом `test.experced.ru`.

**Шаг 1: Подключение к серверу и обновление системы**

1.  В SSH-клиенте Termius выберите сохранённый хост "СПб-Сервер" и подключитесь к нему.

2.  Обновите список пакетов:
    ```bash
    apt update
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Get:1 http://archive.ubuntu.com/ubuntu noble InRelease [256 kB]
    [...]
    Fetched 40.9 MB in 5s (9,367 kB/s)
    Reading package lists... Done
    242 packages can be upgraded. Run 'apt list --upgradable' to see them.
    ```

3.  Обновите установленные пакеты (это может занять несколько минут):
    ```bash
    apt upgrade -y
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Reading package lists... Done
    [...]
    Setting up linux-firmware (20240318.git3b246942-0ubuntu2.10) ...
    Processing triggers for systemd (255.4-1ubuntu8.1) ...
    root@kingsplants:~#
    ```

**Шаг 2: Проверка IP-адреса и установка Nginx и Certbot**

1.  Узнайте IP-адрес сервера (замените `test.experced.ru` на ваш домен).
    ```bash
    nslookup test.experced.ru
    ```
    *   **Ожидаемый вывод:**
    ```
    Server:		127.0.0.53
    Address:	127.0.0.53#53

    Non-authoritative answer:
    Name:	test.experced.ru
    Address: 103.71.22.37  <-- Это IP-адрес вашего сервера
    ```

2.  Установите Nginx и Certbot.
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Reading package lists... Done
    [...]
    Setting up nginx-common (1.24.0-2ubuntu7.1) ...
    Setting up certbot (2.8.0-1) ...
    root@kingsplants:~#
    ```

**Шаг 3: Настройка Nginx и получение SSL-сертификата**

1.  Удалите стандартный конфигурационный файл Nginx.
    ```bash
    rm /etc/nginx/sites-enabled/default
    ```
    *   **Ожидаемый вывод:** Ничего.

2.  Создайте директорию для файлов сайта.
    ```bash
    mkdir /var/www/html/site
    ```
    *   **Ожидаемый вывод:** Ничего.

3.  Создайте и разместите файл `index.html` в директории `/var/www/html/site`.

4.  Создайте новый конфигурационный файл для Nginx.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    Вставьте конфигурацию для перенаправления с HTTP на HTTPS (замените `test.experced.ru` на ваш домен):
    ```nginx
    server {
        listen 80;
        server_name test.experced.ru;

        if ($host = test.experced.ru) {
            return 301 https://$host$request_uri;
        }

        return 404;
    }
    ```
    Сохраните файл (`Ctrl+X`, `Y`, `Enter`).

5.  Активируйте конфигурацию.
    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```
    *   **Ожидаемый вывод:** Ничего.

6.  Запустите Certbot для получения SSL-сертификата.
    ```bash
    certbot --nginx -d test.experced.ru
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/test.experced.ru/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/test.experced.ru/privkey.pem
    [...]
    Congratulations! You have successfully enabled HTTPS on https://test.experced.ru
    ```

7.  Откройте конфигурационный файл для редактирования.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    **Удалите всё содержимое** и вставьте финальную конфигурацию:
    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test.experced.ru/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;

        # Настройки Proxy Protocol
        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
        set_real_ip_from [::1];

        root /var/www/html/site;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
    Сохраните и выйдите.

8.  Проверьте синтаксис Nginx и перезапустите сервис.
    ```bash
    nginx -t
    ```
    *   **Ожидаемый вывод:**
    ```
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```
    ```bash
    systemctl restart nginx
    ```
    *   **Ожидаемый вывод:** Ничего.

**Шаг 4: Установка и настройка панели 3X-UI**

1.  Выполните команду установки.
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Would you like to customize the Panel Port settings? [y/n]: y
    Please set up the panel port: 8080
    [...]
    Username: aBcDeFgH
    Password: xYz12345
    Access URL: http://SERVER_IP:8080/path/
    ```
    **Сохраните имя пользователя и пароль!**

2.  Откройте URL в браузере и войдите в панель.

3.  В панели перейдите в **Inbounds** -> **Add Inbound** и настройте:
    *   **Port:** `443`
    *   **Security:** `reality`
    *   **Client:** Создайте нового клиента (email: `Test`, subscription: `Test`)
    *   **Flow:** `xtls-rprx-vision`
    *   **Target:** `127.0.0.1:8443`
    *   **SNI:** `test.experced.ru`
    *   Нажмите **Get New Keys**, затем **Create**.

4.  Перейдите в **Routing Rules** -> **Add Rule**:
    *   **Outbound Tag:** `Test`
    *   **Inbound Tags:** `inbound-443`
    *   Нажмите **Add Rule**.

5.  Нажмите **Save**, а затем **Restart Panel**.

**Шаг 5: Проверка работы через клиент**

1.  Скопируйте ссылку на подключение из панели и импортируйте в клиент.
2.  Запустите тест скорости и убедитесь, что IP-адрес соответствует серверу в Санкт-Петербурге.

---

### **Часть 2: Настройка второго сервера (Германия)**

**Шаг 6: Подключение и обновление**

1.  Подключитесь ко второму серверу ("Франкфурт сервер").
2.  Обновите систему:
    ```bash
    apt update
    ```
    ```bash
    apt upgrade -y
    ```

**Шаг 7: Настройка Nginx и SSL для второго домена**

1.  Установите Nginx и Certbot.
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
2.  Удалите стандартный конфигурационный файл.
    ```bash
    rm /etc/nginx/sites-enabled/default
    ```
3.  Создайте директорию для сайта.
    ```bash
    mkdir /var/www/html/site
    ```
4.  Перенесите `index.html` в `/var/www/html/site` через SFTP.

5.  Создайте конфигурационный файл для Nginx.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    Вставьте конфигурацию для HTTP-редиректа, **используя второй домен**:
    ```nginx
    server {
        listen 80;
        server_name test2.experced.ru;

        if ($host = test2.experced.ru) {
            return 301 https://$host$request_uri;
        }

        return 404;
    }
    ```
    Сохраните файл (`Ctrl+X`, `Y`, `Enter`).

6.  Активируйте конфигурацию.
    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```

7.  Получите SSL-сертификат для второго домена.
    ```bash
    certbot --nginx -d test2.experced.ru
    ```
    *   **Ожидаемый вывод (сокращённый):**
    ```
    Congratulations! You have successfully enabled HTTPS on https://test2.experced.ru
    ```

8.  Откройте конфигурационный файл `sni.conf` еще раз.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    **Полностью очистите файл** и вставьте финальную конфигурацию:
    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test2.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test2.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test2.experced.ru/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:10m;
        ssl_session_tickets off;

        # Настройки Proxy Protocol
        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
        set_real_ip_from [::1];

        root /var/www/html/site;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
    Сохраните и выйдите.

9.  Проверьте и перезапустите Nginx.
    ```bash
    nginx -t && systemctl restart nginx
    ```
    *   **Ожидаемый вывод:** `...syntax is ok ...test is successful`

**Шаг 8: Установка и настройка 3X-UI на втором сервере**

1.  Установите панель.
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```
2.  Запустите меню управления и сбросьте/установите логин и пароль.
    ```bash
    x-ui
    ```
    Выберите опцию `6`.
3.  Войдите в веб-панель по IP-адресу.
4.  Перейдите в **Panel Settings** -> **Certificates** и укажите пути:
    *   **Public Key Path:** `/etc/letsencrypt/live/test2.experced.ru/fullchain.pem`
    *   **Private Key Path:** `/etc/letsencrypt/live/test2.experced.ru/privkey.pem`
5.  Перейдите во вкладку **Subscription**, включите сервис, укажите **Listen Port:** `8443` и **URI Path:** `/subsbca`.
6.  Нажмите **Save** и **Restart Panel**.
7.  Перезайдите в панель, используя домен: `https://test2.experced.ru:8080`.

**Шаг 9: Настройка входящих и исходящих соединений**

1.  Создайте **Inbound** (входящее соединение) с `reality`, используя SNI `test2.experced.ru`.
2.  Создайте **Outbound**, импортировав JSON с конфигурацией первого сервера.
3.  Создайте **Routing Rule**, чтобы направить трафик с нового Inbound на созданный Outbound.
4.  Сохраните и перезапустите панель.

**Шаг 10: Финальная проверка**

1.  Скопируйте URL подписки со второго сервера и обновите профили в клиенте.
2.  Удалите старый профиль и протестируйте новый.
3.  Проверьте свой IP-адрес. Он должен соответствовать IP-адресу сервера в Германии, подтверждая успешную настройку.
