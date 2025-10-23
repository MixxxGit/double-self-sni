Отлично! Вот пошаговое руководство, созданное на основе предоставленного видео. Оно точно повторяет все действия, включая команды и конфигурации для обоих серверов.

### Введение

В этом руководстве мы настроим два сервера: один в Санкт-Петербурге, другой в Германии. На оба сервера будет установлен Nginx с SSL-сертификатами от Let's Encrypt и панель управления 3X-UI. Затем мы настроим маршрутизацию трафика таким образом, чтобы сервер в Германии (Франкфурт) перенаправлял трафик через сервер в России (Санкт-Петербург).

*   **Сервер 1 (СПБ):** Выполняет роль конечной точки (прокси).
*   **Сервер 2 (Франкфурт):** Выполняет роль входной точки, которая перенаправляет трафик на Сервер 1.

---

### Часть 1. Настройка Сервера 1 (Санкт-Петербург)

#### 1.1. Подключение к серверу и обновление системы

1.  Подключитесь к вашему серверу в Санкт-Петербурге через SSH.
2.  Обновите список пакетов:

    ```bash
    apt update
    ```

    **Ожидаемый вывод (сокращенный):**
    > ...
    > Reading package lists... Done
    > Building dependency tree... Done
    > Reading state information... Done
    > 242 packages can be upgraded. Run 'apt list --upgradable' to see them.

3.  Обновите установленные пакеты:

    ```bash
    apt upgrade
    ```

    Система запросит подтверждение. Введите `y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    > After this operation, 866 MB of additional disk space will be used.
    > Do you want to continue? [Y/n] y
    > ...
    > Processing triggers for man-db (2.12.0-4ubuntu2) ...
    > Processing triggers for install-info (7.1-1build1) ...
    > Processing triggers for libc-bin (2.39-0ubuntu8.1) ...
    > root@vkinggiants:~#

#### 1.2. Проверка IP-адреса и привязка домена

1.  Убедитесь, что ваш домен (в примере `test.experced.ru`) указывает на IP-адрес этого сервера. Сначала проверьте, на какой IP разрешается домен:

    ```bash
    nslookup test.experced.ru
    ```

    **Ожидаемый вывод:**
    > Server:		127.0.0.53
    > Address:	127.0.0.53#53
    > 
    > Non-authoritative answer:
    > Name:	test.experced.ru
    > Address: 103.71.22.37

2.  Проверьте IP-адрес вашего сервера:

    ```bash
    ip a
    ```

    **Ожидаемый вывод (сокращенный):**
    > ...
    > 3: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    >     link/ether 5e:b1:d0:c1:1a:c1 brd ff:ff:ff:ff:ff:ff
    >     inet 103.71.22.37/20 brd 103.71.31.255 scope global ens3
    > ...

3.  В панели управления вашего DNS-провайдера создайте A-запись, указывающую на IP-адрес сервера. В видео используются следующие параметры:
    *   **Тип:** `A`
    *   **Имя:** `test`
    *   **IPv4-адрес:** `103.71.22.37`
    *   **TTL:** `300`

#### 1.3. Установка Nginx и Certbot

1.  Установите Nginx и Certbot с плагином для Nginx:

    ```bash
    sudo apt install nginx certbot python3-certbot-nginx
    ```

    Система запросит подтверждение. Введите `y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    > After this operation, 7,400 kB of additional disk space will be used.
    > Do you want to continue? [Y/n] y
    > ...
    > Setting up python3-certbot-nginx (2.9.0-1) ...
    > Processing triggers for ufw (0.36.2-1ubuntu1) ...
    > Scanning processes...
    > Scanning linux images...
    > [ok]

#### 1.4. Настройка Nginx и получение SSL-сертификата

1.  Удалите стандартную конфигурацию Nginx:

    ```bash
    rm /etc/nginx/sites-enabled/default
    ```

2.  Создайте директорию для вашего сайта:

    ```bash
    mkdir /var/www/html/site
    ```

3.  Создайте файл `index.html` в этой директории. В видео используется SFTP-клиент для загрузки файла. Содержимое файла `index.html`:

    ```html
    <!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Простой Одностраничный Сайт</title>
        <style>
            body { font-family: sans-serif; display: flex; justify-content: center; align-items: center; height: 100vh; margin: 0; background-color: #f0f0f0; }
            .container { text-align: center; padding: 20px; border-radius: 8px; background-color: white; box-shadow: 0 4px 8px rgba(0,0,0,0.1); }
            h1 { color: #333; }
        </style>
    </head>
    <body>
        <div class="container">
            <h1>Hello World</h1>
            <p>Это простой одностраничный HTML-сайт.</p>
        </div>
    </body>
    </html>
    ```
    
    С помощью SFTP-клиента переименуйте локальный файл в `index.html` и загрузите его на сервер в директорию `/var/www/html/site`.

4.  Создайте конфигурационный файл Nginx для вашего домена:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

5.  Вставьте в редактор следующую конфигурацию для перенаправления HTTP на HTTPS. Замените `доменное имя` на ваш домен (`test.experced.ru`).

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

6.  Сохраните файл и выйдите из редактора (Ctrl+X, затем Y, затем Enter).

7.  Создайте символическую ссылку, чтобы активировать сайт:

    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```

8.  Запустите Certbot для получения SSL-сертификата:

    ```bash
    certbot --nginx -d test.experced.ru
    ```

    В процессе выполнения Certbot задаст несколько вопросов:
    *   Введите email-адрес: `testadawd@mail.ru`
    *   Согласитесь с условиями обслуживания: `y`
    *   Согласитесь на рассылку от EFF: `y`

    **Ожидаемый вывод (сокращенный):**
    > Successfully deployed certificate for test.experced.ru to /etc/nginx/sites-enabled/sni.conf
    > Congratulations! You have successfully enabled HTTPS on https://test.experced.ru

#### 1.5. Финальная конфигурация Nginx для проксирования

1.  Откройте конфигурационный файл Nginx снова. Certbot его изменил, но мы заменим его содержимое.

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

2.  Полностью удалите содержимое файла и вставьте следующую конфигурацию. Замените `доменное имя` на ваш домен (`test.experced.ru`).

    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name доменное имя;

        ssl_certificate /etc/letsencrypt/live/доменное имя/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/доменное имя/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;

        # Настройки Proxy Protocol
        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
        set_real_ip_from ::1;

        root /var/www/html/site;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```

3.  Замените все вхождения `доменное имя` на `test.experced.ru`. Сохраните и закройте файл.

4.  Проверьте синтаксис конфигурации Nginx:

    ```bash
    nginx -t
    ```

    **Ожидаемый вывод:**
    > nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    > nginx: configuration file /etc/nginx/nginx.conf test is successful

5.  Перезапустите Nginx, чтобы применить изменения:

    ```bash
    systemctl restart nginx
    ```

#### 1.6. Установка и настройка 3X-UI

1.  Выполните команду для установки панели 3X-UI:

    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
    ```

2.  В процессе установки скрипт задаст вопросы:
    *   `Would you like to customize the Panel Port settings? (If not, a random port will be applied) (y/n):` Введите `y`.
    *   `Please set up the panel port:` Введите `8080`.

    **Ожидаемый вывод (сохращенный):**
    > ...
    > This is a fresh installation, generating random login info for security concerns:
    > Username: W6THI5MX9Y
    > Password: 5M8oUqojgD9
    > 
    > Access URL: http://103.71.22.37:8080/w6thi5mx9y/
    > ...
    > x-ui v2.1.1 Installation Finished, It's running now...

3.  Откройте в браузере предоставленный `Access URL` и войдите в панель, используя сгенерированные имя пользователя и пароль.
4.  Перейдите в раздел **Inbounds** и нажмите `+ Add Inbound`.
5.  Настройте входящее подключение:
    *   **Port:** `443`
    *   **Security:** `reality`
    *   **Client:** Нажмите `+`, в поле `Email` введите `Test`.
    *   **Flow:** `xtls-rprx-vision`
    *   **uTLS:** `firefox`
    *   **Target:** `127.0.0.1:9000`
    *   **SNI:** `test.experced.ru`
    *   Нажмите `Get New Keys`.
    *   Включите **Sniffing**.
    *   Нажмите **Create**.
6.  В списке Inbounds появится новое правило. Нажмите на иконку информации (i), чтобы увидеть детали, и скопируйте **Subscription URL**. Этот URL понадобится нам позже.

#### 1.7. Перезагрузка сервера
Перезагрузите сервер, чтобы применились все обновления ядра.

```bash
reboot
```

---

### Часть 2. Настройка Сервера 2 (Франкфурт)

#### 2.1. Подключение к серверу и обновление системы

1.  Подключитесь к вашему серверу во Франкфурте через SSH.
2.  Обновите список пакетов:

    ```bash
    apt-get update
    ```
    **Ожидаемый вывод (сокращенный):**
    > ...
    > Reading package lists... Done
    > 242 packages can be upgraded. Run 'apt list --upgradable' to see them.

3.  Обновите установленные пакеты:

    ```bash
    apt-get upgrade
    ```

    Система запросит подтверждение. Введите `y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    > After this operation, 386 MB of additional disk space will be used.
    > Do you want to continue? [Y/n] y
    > ...
    > Processing triggers for systemd (255.4-1ubuntu3.1) ...
    > Processing triggers for initramfs-tools (0.142ubuntu1.1) ...
    > root@server-white:~#

#### 2.2. Проверка IP-адреса и привязка домена

1.  Убедитесь, что ваш второй домен (в примере `test2.experced.ru`) указывает на IP-адрес этого сервера:

    ```bash
    nslookup test2.experced.ru
    ```

    **Ожидаемый вывод:**
    > Server:		127.0.0.53
    > Address:	127.0.0.53#53
    > 
    > Non-authoritative answer:
    > Name:	test2.experced.ru
    > Address: 79.137.202.37

2.  Проверьте IP-адрес вашего сервера:

    ```bash
    ip a
    ```

    **Ожидаемый вывод (сокращенный):**
    > ...
    > 2: eth0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
    >     link/ether 00:00:00:00:00:00 brd ff:ff:ff:ff:ff:ff
    >     inet 79.137.202.37/24 scope global eth0
    > ...

#### 2.3. Установка Nginx и Certbot

1.  Установите Nginx и Certbot с плагином для Nginx:

    ```bash
    sudo apt install nginx certbot python3-certbot-nginx
    ```

    Система запросит подтверждение. Введите `y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    > After this operation, 7,400 kB of additional disk space will be used.
    > Do you want to continue? [Y/n] y
    > ...
    > Processing triggers for ufw (0.36.2-1) ...
    > Scanning processes...
    > [ok]

#### 2.4. Настройка Nginx и получение SSL-сертификата

1.  Удалите стандартную конфигурацию Nginx:

    ```bash
    rm /etc/nginx/sites-enabled/default
    ```

2.  Создайте директорию для вашего сайта:

    ```bash
    mkdir /var/www/html/site
    ```

3.  Загрузите файл `index.html` (с тем же содержимым, что и для Сервера 1) в директорию `/var/www/html/site` с помощью SFTP.

4.  Создайте конфигурационный файл Nginx для второго домена:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

5.  Вставьте в редактор конфигурацию для перенаправления HTTP на HTTPS. Замените `доменное имя` на ваш второй домен (`test2.experced.ru`).

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

6.  Сохраните файл и выйдите из редактора (Ctrl+X, затем Y, затем Enter).

7.  Создайте символическую ссылку, чтобы активировать сайт:

    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```

8.  Запустите Certbot для получения SSL-сертификата:

    ```bash
    certbot --nginx -d test2.experced.ru
    ```

    В процессе выполнения Certbot задаст несколько вопросов:
    *   Введите email-адрес: `testadawd@mail.ru`
    *   Согласитесь с условиями обслуживания: `y`
    *   Согласитесь на рассылку от EFF: `y`

    **Ожидаемый вывод (сокращенный):**
    > Successfully deployed certificate for test2.experced.ru to /etc/nginx/sites-enabled/sni.conf
    > Congratulations! You have successfully enabled HTTPS on https://test2.experced.ru

#### 2.5. Финальная конфигурация Nginx для проксирования

1.  Откройте конфигурационный файл Nginx снова:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

2.  Полностью удалите содержимое файла и вставьте следующую конфигурацию, заменив `доменное имя` на `test2.experced.ru`.

    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test2.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test2.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test2.experced.ru/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384;
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;

        # Настройки Proxy Protocol
        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
        set_real_ip_from ::1;

        root /var/www/html/site;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```

3.  Сохраните и закройте файл.

4.  Проверьте синтаксис конфигурации Nginx:

    ```bash
    nginx -t
    ```
    **Ожидаемый вывод:**
    > nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    > nginx: configuration file /etc/nginx/nginx.conf test is successful

5.  Перезапустите Nginx:

    ```bash
    systemctl restart nginx
    ```

#### 2.6. Установка и настройка 3X-UI

1.  Выполните команду для установки панели 3X-UI:
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/mhsanaei/3x-ui/master/install.sh)
    ```

2.  В процессе установки скрипт задаст вопросы:
    *   `Would you like to customize the Panel Port settings? (If not, a random port will be applied) (y/n):` Введите `y`.
    *   `Please set up the panel port:` Введите `8080`.

    **Ожидаемый вывод (сокращенный):**
    > ...
    > Username: fk330PNA2
    > Password: WsJb3KhbW6zL2d
    >
    > Access URL: http://79.137.202.37:8080/fk330pna2/
    > ...
    > x-ui v2.1.1 Installation Finished, It's running now...
    
3.  Откройте в браузере предоставленный `Access URL` и войдите в панель, используя сгенерированные имя пользователя и пароль.

4.  **Настройка Inbounds:**
    *   Перейдите в **Inbounds** и нажмите `+ Add Inbound`.
    *   **Port:** `443`
    *   **Security:** `reality`
    *   **Client:** Нажмите `+`, в поле `Email` введите `Test`.
    *   **Flow:** `xtls-rprx-vision`
    *   **uTLS:** `firefox`
    *   **Target:** `127.0.0.1:9000`
    *   **SNI:** `test2.experced.ru`
    *   Нажмите `Get New Keys`.
    *   Нажмите **Create**.

5.  **Настройка Outbounds:**
    *   Перейдите в **X-ray Configs → Outbounds**.
    *   Нажмите `+ Add Outbound`.
    *   Переключитесь на вкладку **JSON**.
    *   В поле **Link** вставьте **Subscription URL**, который вы скопировали с **Сервера 1 (СПБ)**. Нажмите значок импорта.
    *   Нажмите `Add Outbound`.

6.  **Настройка Routing Rules:**
    *   Перейдите в **X-ray Configs → Routing Rules**.
    *   Нажмите `+ Add Rule`.
    *   **Network:** `TCP,UDP`
    *   **Outbound Tag:** `Test` (или как назвался ваш импортированный outbound).
    *   **Inbound Tags:** `inbound-443`.
    *   Нажмите `Add Rule`.

7.  Сохраните и перезапустите панель, нажав `Save` и `Restart Panel`.

8.  **Сброс учетных данных (опционально, показано в видео):**
    *   Вернитесь в терминал и выполните:
        ```bash
        x-ui
        ```
    *   Выберите опцию `6` (Reset Username & Password).
    *   `Are you sure to reset the username and password of the panel? [default n]:` `y`
    *   `Please set the login username (default is a random username):` `n`
    *   `Please set the login password (default is a random password):` `n`
    *   `Do you want to disable currently configured two-factor authentication? (y/n):` `n`
    *   `Restart the panel, Attention: Restarting the panel will also restart xray [default y]:` `y`

    **Ожидаемый вывод (сокращенный):**
    > Panel login username has been reset to: Jv2NtGfGgcshl3tJk
    > Panel login password has been reset to: jv2ntgfgcshl3tjk
    > Please use the new login username and password to access the X-UI panel. Also remember them!

#### 2.7. Настройка TLS и подписок

1.  Войдите в панель 3X-UI с новыми учетными данными.
2.  Перейдите в **Panel Settings -> Certificates**.
3.  Скопируйте пути к SSL-сертификатам из терминала (из файла `sni.conf` Сервера 2):
    *   **Public Key Path:** `/etc/letsencrypt/live/test2.experced.ru/fullchain.pem`
    *   **Private Key Path:** `/etc/letsencrypt/live/test2.experced.ru/privkey.pem`
4.  Перейдите во вкладку **Subscription**.
    *   **Listen Port:** `8443`
    *   **URI Path:** `/subsbsca/`
    *   **Public Key Path:** `/etc/letsencrypt/live/test2.experced.ru/fullchain.pem`
    *   **Private Key Path:** `/etc/letsencrypt/live/test2.experced.ru/privkey.pem`
5.  Нажмите `Save` и `Restart Panel`.

---

### Часть 3. Тестирование соединения

1.  После перезапуска панели перейдите в раздел **Inbounds**.
2.  Найдите ваше входящее правило (`Client: Test`) и откройте его детали.
3.  В разделе **Subscription URL** скопируйте URL.
4.  Импортируйте этот URL в ваш V2Ray-клиент (в видео используется Nekobox).
5.  После импорта запустите тест задержки (latency test) для добавленного профиля. Тест должен показать успешное соединение и задержку.
6.  Активируйте профиль и проверьте скорость интернет-соединения через сервис `speedtest.net`. Видео демонстрирует высокую скорость загрузки и выгрузки, что подтверждает корректную работу всей цепочки.
