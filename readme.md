### Введение

В этом руководстве мы настроим два сервера: один в Санкт-Петербурге (далее "СПБ Сервер") и второй во Франкфурте, Германия (далее "Франкфурт сервер"). На каждом сервере будет установлен веб-сервер Nginx, получен SSL-сертификат с помощью Certbot и развернута панель управления прокси 3X-UI. Затем мы настроим их для совместной работы.

---

### Часть 1: Настройка сервера в Санкт-Петербурге (СПБ Сервер)

#### Шаг 1.1: Подключение к серверу и обновление системы

1.  Подключитесь к вашему серверу в Санкт-Петербурге через SSH.
2.  Обновите списки пакетов:

    ```bash
    apt update
    ```

    **Ожидаемый вывод (сокращенный):**
    ```
    Get:1 http://security.ubuntu.com/ubuntu noble-security InRelease [120 kB]
    ...
    Get:20 http://archive.ubuntu.com/ubuntu noble-backports/multiverse amd64 c-n-f Metadata [108 B]
    Fetched 40.9 MB in 5s (9,367 kB/s)
    Reading package lists... Done
    Building dependency tree... Done
    Reading state information... Done
    242 packages can be upgraded. Run 'apt list --upgradable' to see them.
    ```

3.  Обновите установленные пакеты:

    ```bash
    apt upgrade
    ```

    Во время выполнения команды система запросит подтверждение. Введите `Y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    The following packages will be upgraded:
      amd64-microcode ...
    ...
    After this operation, 360 MB of additional disk space will be used.
    Do you want to continue? [Y/n] Y
    Get:1 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 md-tool all 0.9.0-6ubuntu0.1 [20.6 kB]
    ...
    Get:242 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 linux-firmware 20240318.git3b12b295-0ubuntu2.10 [110 MB]
    ...
    Processing triggers for systemd (255.4-1ubuntu8.1) ...
    Processing triggers for ufw (0.36.2-4) ...
    ...
    Pending kernel upgrade!
    Running kernel version:
      6.8.0-40-generic
    Diagnostics:
    The currently running kernel version is not the expected kernel version 6.8.0-38-generic.
    Restarting the system to load the new kernel will not be handled automatically, so you should consider rebooting.
    ...
    *** System restart required ***
    ```

4.  Проверьте, что ваш домен (`test.experced.ru`) указывает на IP-адрес сервера:

    ```bash
    nslookup test.experced.ru
    ```

    **Ожидаемый вывод:**
    ```
    Server:		127.0.0.53
    Address:	127.0.0.53#53

    Non-authoritative answer:
    Name:	test.experced.ru
    Address: 103.71.22.37
    ```

5.  Проверьте IP-адрес сервера:

    ```bash
    ip a
    ```

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 52:54:00:01:04:3d brd ff:ff:ff:ff:ff:ff
        inet 103.71.22.37/24 brd 103.71.22.255 scope global ens3
           valid_lft forever preferred_lft forever
    ...
    ```

#### Шаг 1.2: Установка Nginx и Certbot

1.  В окне браузера открыта инструкция по адресу `https://wiki.kasta.pro/h/nastrojka-sni-sajta-dlja-reality`. Пользователь копирует команду для установки Nginx и Certbot.
2.  Выполните команду в терминале:

    ```bash
    sudo apt install nginx certbot python3-certbot-nginx
    ```
    На запрос подтверждения введите `Y` и нажмите Enter.
    
    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    The following NEW packages will be installed:
      nginx-common python3-acme python3-certbot python3-certbot-nginx python3-configargparse python3-josepy python3-parsedatetime python3-rfc3339 python3-tz
    ...
    After this operation, 7,400 kB of additional disk space will be used.
    Do you want to continue? [Y/n] Y
    Get:1 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 nginx-common all 1.26.0-2ubuntu0.1 [41.4 kB]
    ...
    Setting up nginx (1.26.0-2ubuntu0.1) ...
    Setting up python3-certbot-nginx (2.8.0-1_all.deb) ...
    Created symlink /etc/systemd/system/timers.target.wants/certbot.timer → /usr/lib/systemd/system/certbot.timer.
    Setting up certbot (2.8.0-1) ...
    ...
    Processing triggers for ufw (0.36.2-4) ...
    Scanning processes...
    [ OK ]
    ```

#### Шаг 1.3: Настройка Nginx и получение SSL-сертификата

1.  Удалите стандартный конфигурационный файл Nginx:

    ```bash
    rm /etc/nginx/sites-enabled/default
    ```

2.  Создайте директорию для вашего сайта:

    ```bash
    mkdir /var/www/html/site
    ```

3.  В браузере пользователь открывает ChatGPT и запрашивает "Сгенерируй простой HTML сайт".
4.  Создайте файл `index.html` в этой директории. В видео это делается через SFTP-клиент Termius:
    *   В локальной директории `~/Downloads/нека` файл с длинным именем переименовывается в `index.html`.
    *   Этот файл копируется на сервер в директорию `/var/www/html/site`.
    
    Содержимое файла `index.html` должно быть примерно таким:
    ```html
    <!DOCTYPE html>
    <html lang="ru">
    <head>
        <meta charset="UTF-8">
        <title>Простой Одностраничный Сайт</title>
    </head>
    <body>
        <h1>Добро пожаловать на мой сайт!</h1>
        <p>Это простой одностраничный HTML-сайт.</p>
    </body>
    </html>
    ```

5.  Создайте новый конфигурационный файл для Nginx:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

6.  Вставьте в редактор `nano` следующую конфигурацию, заменив `доменное имя` на `test.experced.ru`:

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
    Нажмите `Ctrl+X`, затем `Y` и `Enter`, чтобы сохранить файл и выйти.

7.  Создайте символическую ссылку, чтобы включить сайт:

    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```

8.  Получите SSL-сертификат с помощью Certbot:

    ```bash
    certbot --nginx -d test.experced.ru
    ```
    
    *   Введите email для уведомлений: `testasdasd@mail.ru`
    *   Согласитесь с условиями использования (Terms of Service): `Y`
    *   Согласитесь на рассылку новостей (необязательно): `Y`

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/test.experced.ru/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/test.experced.ru/privkey.pem
    This certificate expires on 2025-01-20.
    ...
    Deploying certificate
    Successfully deployed certificate for test.experced.ru to /etc/nginx/sites-enabled/sni.conf
    Congratulations! You have successfully enabled HTTPS on https://test.experced.ru
    ```

9.  Снова откройте конфигурационный файл для редактирования:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

10. Certbot автоматически изменил файл. Теперь замените всё его содержимое на конфигурацию ниже. В браузере пользователь копирует эту конфигурацию из той же инструкции на `wiki.kasta.pro`. Обязательно замените все вхождения `доменное имя` на ваш домен `test.experced.ru`.

    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test.experced.ru/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
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
    Сохраните файл и выйдите (`Ctrl+X`, `Y`, `Enter`).

11. Проверьте синтаксис конфигурации Nginx:

    ```bash
    nginx -t
    ```

    **Ожидаемый вывод:**
    ```
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

12. Перезапустите Nginx:

    ```bash
    systemctl restart nginx
    ```

#### Шаг 1.4: Установка панели 3X-UI

1.  В браузере пользователь переходит на `https://github.com/MHSanaei/3x-ui` и копирует команду для установки.
2.  Выполните команду в терминале:

    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```

    *   На вопрос `Would you like to customize the Panel Port settings?` ответьте `y`.
    *   На вопрос `Please set up the panel port:` введите `8080`.

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    /usr/local/x-ui/x-ui-linux-amd64.tar.gz          100%[=======================================================================================================>]  11.43M   258KB/s    in 8.2s
    ...
    Username and password updated successfully
    This is a fresh installation, generating random login info for security concerns:
    Username: WETHEMAVSY
    Password: NMdUujgth
    ========================================
    Access URL: http://103.71.22.37:8080/wacmbNbUhUOVGzGO
    ========================================
    ...
    x-ui v2.1.1 Installation finished, it is running now...
    ```

#### Шаг 1.5: Настройка 3X-UI

1.  Откройте в браузере URL-адрес для доступа к панели, указанный в выводе (в данном случае `http://103.71.22.37:8080/wacmbNbUhUOVGzGO`).
2.  Введите `Username` и `Password` из терминала (`WETHEMAVSY` и `NMdUujgth`) и нажмите "Log In".
3.  Перейдите в раздел "Inbounds" (Входящие).
4.  Нажмите кнопку "Add Inbound" (Добавить входящее).
5.  Заполните форму следующим образом:
    *   **Remark (Примечание):** `Test`
    *   **Port (Порт):** `443`
    *   **Security (Безопасность):** `reality`
    *   Нажмите на иконку `+` в секции **Client** (Клиент).
    *   **Email:** `Test`
    *   **Subscription (Подписка):** `Test`
    *   **Flow:** `xtls-rprx-vision`
    *   **Target (Цель):** `127.0.0.1:9000`
    *   **SNI:** `test.experced.ru`
    *   Нажмите "Get New Keys" (Получить новые ключи).
    *   Нажмите "Create" (Создать).

6.  Перейдите в раздел "Outbounds" (Исходящие).
    *   Нажмите "Add Outbound".
    *   Переключитесь на вкладку "JSON".
    *   Вернитесь в "Inbounds", найдите созданное правило, нажмите на меню "..." и выберите "Client". Скопируйте URL-ссылку `vless://...`.
    *   Вставьте скопированную ссылку в поле "Link" на странице "Add Outbound" и нажмите на иконку импорта справа.
    *   Нажмите "Add Outbound".

7.  Перейдите в "Routing Rules" (Правила маршрутизации).
    *   Нажмите "Add Rule".
    *   Заполните форму:
        *   **Network (Сеть):** Выберите `TCP,UDP`.
        *   **Inbound Tags (Теги входящих):** `inbound-443`.
        *   **Outbound Tag (Тег исходящего):** `Test`.
    *   Нажмите "Add Rule".

8.  Сохраните и перезапустите панель.
    *   Вверху страницы появится уведомление `Every change made here needs to be saved`. Нажмите "Save".
    *   Затем нажмите "Restart Panel" и подтвердите действие.

---

### Часть 2: Настройка сервера в Германии (Франкфурт сервер)

Процесс настройки второго сервера почти идентичен первому, но с другими доменным именем и IP-адресом. **Не пропускайте шаги, выполняйте их полностью.**

#### Шаг 2.1: Подключение к серверу и обновление системы

1.  Подключитесь к вашему серверу в Франкфурте через SSH.
2.  Обновите списки пакетов:

    ```bash
    apt update
    ```

    **Ожидаемый вывод (сокращенный):**
    ```
    Get:1 http://security.ubuntu.com/ubuntu noble-security InRelease [120 kB]
    ...
    Fetched 40.9 MB in 5s (9,367 kB/s)
    Reading package lists... Done
    242 packages can be upgraded. Run 'apt list --upgradable' to see them.
    ```

3.  Обновите установленные пакеты:

    ```bash
    apt upgrade
    ```
    На запрос подтверждения введите `Y` и нажмите Enter.

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    The following packages will be upgraded:
      amd64-microcode ...
    ...
    After this operation, 360 MB of additional disk space will be used.
    Do you want to continue? [Y/n] Y
    Get:1 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 md-tool all 0.9.0-6ubuntu0.1 [20.6 kB]
    ...
    Get:242 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 linux-firmware 20240318.git3b12b295-0ubuntu2.10 [110 MB]
    ...
    *** System restart required ***
    ```

4.  Проверьте, что ваш второй домен (`test2.experced.ru`) указывает на IP-адрес этого сервера:

    ```bash
    nslookup test2.experced.ru
    ```

    **Ожидаемый вывод:**
    ```
    Server:		127.0.0.53
    Address:	127.0.0.53#53

    Non-authoritative answer:
    Name:	test2.experced.ru
    Address: 79.137.202.37
    ```

5.  Проверьте IP-адрес сервера:

    ```bash
    ip a
    ```

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    2: ens3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP group default qlen 1000
        link/ether 52:54:00:01:04:3d brd ff:ff:ff:ff:ff:ff
        inet 79.137.202.37/24 brd 79.137.202.255 scope global ens3
           valid_lft forever preferred_lft forever
    ...
    ```

#### Шаг 2.2: Установка Nginx и Certbot

1.  Выполните команду в терминале:

    ```bash
    sudo apt install nginx certbot python3-certbot-nginx
    ```
    На запрос подтверждения введите `Y` и нажмите Enter.
    
    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    The following NEW packages will be installed:
      nginx-common python3-acme python3-certbot python3-certbot-nginx python3-configargparse python3-josepy python3-parsedatetime python3-rfc3339 python3-tz
    ...
    After this operation, 7,400 kB of additional disk space will be used.
    Do you want to continue? [Y/n] Y
    ...
    [ OK ]
    ```

#### Шаг 2.3: Настройка Nginx и получение SSL-сертификата

1.  Удалите стандартный конфигурационный файл Nginx:

    ```bash
    rm /etc/nginx/sites-enabled/default
    ```

2.  Создайте директорию для вашего сайта:

    ```bash
    mkdir /var/www/html/site2
    ```

3.  Скопируйте файл `index.html` на сервер в директорию `/var/www/html/site2` через SFTP.

4.  Создайте новый конфигурационный файл для Nginx:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

5.  Вставьте в редактор `nano` следующую конфигурацию, заменив `доменное имя` на `test2.experced.ru`:

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
    Нажмите `Ctrl+X`, затем `Y` и `Enter`, чтобы сохранить файл и выйти.

6.  Создайте символическую ссылку, чтобы включить сайт:

    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```

7.  Получите SSL-сертификат с помощью Certbot для второго домена:

    ```bash
    certbot --nginx -d test2.experced.ru
    ```
    
    *   Введите email для уведомлений: `testasdasd@mail.ru`
    *   Согласитесь с условиями использования (Terms of Service): `Y`
    *   Согласитесь на рассылку новостей (необязательно): `Y`

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/test2.experced.ru/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/test2.experced.ru/privkey.pem
    ...
    Deploying certificate
    Successfully deployed certificate for test2.experced.ru to /etc/nginx/sites-enabled/sni.conf
    Congratulations! You have successfully enabled HTTPS on https://test2.experced.ru
    ```

8.  Снова откройте конфигурационный файл для редактирования:

    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```

9.  Замените всё его содержимое на конфигурацию ниже. Замените все вхождения `доменное имя` на ваш домен `test2.experced.ru`.

    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test2.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test2.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test2.experced.ru/privkey.pem;

        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_prefer_server_ciphers on;
        ssl_ciphers 'ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384';
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout 1d;
        ssl_session_tickets off;

        # Настройки Proxy Protocol
        real_ip_header proxy_protocol;
        set_real_ip_from 127.0.0.1;
        set_real_ip_from ::1;

        root /var/www/html/site2;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
    Сохраните файл и выйдите (`Ctrl+X`, `Y`, `Enter`).

10. Проверьте синтаксис конфигурации Nginx:

    ```bash
    nginx -t
    ```

    **Ожидаемый вывод:**
    ```
    nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
    nginx: configuration file /etc/nginx/nginx.conf test is successful
    ```

11. Перезапустите Nginx:

    ```bash
    systemctl restart nginx
    ```

#### Шаг 2.4: Установка панели 3X-UI

1.  Выполните команду в терминале:

    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```

    *   На вопрос `Would you like to customize the Panel Port settings?` ответьте `y`.
    *   На вопрос `Please set up the panel port:` введите `8080`.

    **Ожидаемый вывод (сокращенный):**
    ```
    ...
    Username: JvWPPgorsm
    Password: k330BfNAJ
    ========================================
    Access URL: http://79.137.202.37:8080/wacmbNbUhUOVGzGO
    ========================================
    ...
    x-ui v2.1.1 Installation finished, it is running now...
    ```

#### Шаг 2.5: Настройка панели 3X-UI

1.  Откройте в браузере URL-адрес для доступа к панели, указанный в выводе (в данном случае `http://79.137.202.37:8080/wacmbNbUhUOVGzGO`).
2.  Введите `Username` и `Password` из терминала (`JvWPPgorsm` и `k330BfNAJ`) и нажмите "Log In".
3.  Перейдите в раздел "Inbounds" (Входящие).
4.  Нажмите "Add Inbound".
5.  Заполните форму:
    *   **Port (Порт):** `443`
    *   **Security (Безопасность):** `reality`
    *   **Flow:** `xtls-rprx-vision`
    *   **Target (Цель):** `127.0.0.1:9000`
    *   **SNI:** `test2.experced.ru`
    *   Нажмите "Get New Keys".
    *   Нажмите "Create".

6.  Перейдите в "Panel Settings" -> "Xray Configs".
7.  Нажмите "Outbounds" -> "Add Outbound".
8.  Переключитесь на вкладку "JSON", вставьте URL-ссылку от клиента с первого сервера (из шага 1.5) и нажмите иконку импорта.
9.  Нажмите "Add Outbound".

10. Перейдите в "Routing Rules" (Правила маршрутизации).
    *   Нажмите "Add Rule".
    *   Заполните форму:
        *   **Network (Сеть):** Выберите `TCP,UDP`.
        *   **Inbound Tags (Теги входящих):** `inbound-443`.
        *   **Outbound Tag (Тег исходящего):** `Test`.
    *   Нажмите "Add Rule".

11. Сохраните и перезапустите панель: Нажмите "Save", затем "Restart Panel".

#### Шаг 2.6: Настройка SSL для панели 3X-UI

1.  После перезапуска панели, в браузере перейдите в "Panel Settings".
2.  Откройте секцию "Certificates".
3.  В поле "Public Key Path" вставьте путь к вашему сертификату:
    ```
    /etc/letsencrypt/live/test2.experced.ru/fullchain.pem
    ```
4.  В поле "Private Key Path" вставьте путь к приватному ключу:
    ```
    /etc/letsencrypt/live/test2.experced.ru/privkey.pem
    ```
5.  Нажмите "Save", затем "Restart Panel" и подтвердите действие.
6.  Сервер может прервать SSH-соединение. Переподключитесь.

#### Шаг 2.7: Изменение учетных данных панели 3X-UI

1.  В терминале второго сервера выполните команду для доступа к меню управления панелью:

    ```bash
    x-ui
    ```

2.  Выберите опцию `6` (Reset Username & Password).
3.  Система запросит подтверждение. Введите `y`.
4.  На следующие два запроса `Set the login username` и `Set the login password` просто нажмите Enter, чтобы сгенерировать случайные данные.
5.  На вопрос `Do you want to disable currently configured two-factor authentication?` ответьте `n`.
6.  На вопрос `Restart the panel?` ответьте `y`.

    **Ожидаемый вывод:**
    ```
    Panel login username has been reset to: Jv2WzMOrgoe
    Panel login password has been reset to: ZvWPPgorsm5hJk
    Please use the new login username and password to access the X-UI panel. Also remember them!
    ```

#### Шаг 2.8: Финальная проверка

1.  Откройте в браузере URL-адрес панели, но уже с HTTPS и без порта 8080: `https://test2.experced.ru/wacmbNbUhUOVGzGO`
2.  Войдите в систему, используя новые учетные данные из терминала (`Jv2WzMOrgoe` и `ZvWPPgorsm5hJk`).
3.  Перейдите в раздел "Inbounds".
4.  Найдите клиента "Test", нажмите меню "..." -> "More Information".
5.  Скопируйте "Subscription URL".
6.  В клиенте прокси (в видео используется NekoBox) добавьте новую подписку из буфера обмена.
7.  Проведите тест подключения. Он должен быть успешным.
8.  Откройте сайт `speedtest.net` и запустите тест. В видео показаны результаты: **Download 871.60 Mbps** и **Upload 410.27 Mbps**.
