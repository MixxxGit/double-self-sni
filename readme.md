### **Общая цель**

Настроить два сервера с панелью управления прокси-соединениями 3X-UI, защитить их с помощью SSL-сертификатов от Let's Encrypt и настроить проксирование трафика.

---

### **Часть 1: Настройка первого сервера (Санкт-Петербург)**

На этом этапе все действия производятся на первом сервере с тестовым доменом `test.experced.ru`.

**Шаг 1: Подключение к серверу и обновление системы**

1.  В SSH-клиенте Termius выберите сохранённый хост "СПб-Сервер" и подключитесь.

2.  Обновите список пакетов:
    ```bash
    apt update
    ```
    *   **Ожидаемый вывод (сокращённый):** Вы увидите процесс загрузки списков пакетов с репозиториев. Успешное выполнение завершится строкой, сообщающей, сколько пакетов можно обновить.
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
    *   **Ожидаемый вывод (сокращённый):** Команда покажет список обновляемых пакетов, скачает их и установит. Успешное завершение — это возврат к командной строке без сообщений об ошибках.
    ```
    Reading package lists... Done
    Building dependency tree... Done
    [...]
    Get:1 http://archive.ubuntu.com/ubuntu noble-updates/main amd64 linux-firmware all [242 kB]
    [...]
    Unpacking linux-firmware (20240318.git3b246942-0ubuntu2.10) over (20240318.git3b246942-0ubuntu2.8) ...
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
    *   **Ожидаемый вывод (сокращённый):** Процесс установки похож на `apt upgrade`. В конце вы увидите строки о настройке пакетов и возврат к командной строке.
    ```
    Reading package lists... Done
    [...]
    Setting up nginx-common (1.24.0-2ubuntu7.1) ...
    Setting up certbot (2.8.0-1) ...
    [...]
    root@kingsplants:~#
    ```

**Шаг 3: Настройка Nginx и получение SSL-сертификата**

1.  Удалите стандартный конфигурационный файл Nginx.
    ```bash
    rm /etc/nginx/sites-enabled/default
    ```
    *   **Ожидаемый вывод:** Команда не выводит ничего в случае успеха.

2.  Создайте директорию для файлов сайта.
    ```bash
    mkdir /var/www/html/site
    ```
    *   **Ожидаемый вывод:** Команда не выводит ничего в случае успеха.

3.  Создайте и разместите файл `index.html` в директории `/var/www/html/site`. Это можно сделать через SFTP, как в видео.

4.  Создайте новый конфигурационный файл для Nginx.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    Вставьте в редактор `nano` конфигурацию для перенаправления с HTTP на HTTPS (замените `test.experced.ru` на ваш домен):
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

5.  Активируйте конфигурацию, создав символическую ссылку.
    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```
    *   **Ожидаемый вывод:** Команда не выводит ничего в случае успеха.

6.  Запустите Certbot для получения SSL-сертификата.
    ```bash
    certbot --nginx -d test.experced.ru
    ```
    *   **Ожидаемый вывод (сокращённый):** Certbot задаст несколько вопросов (email, согласие с условиями). В конце вы увидите сообщение об успешном получении сертификата.
    ```
    Saving debug log to /var/log/letsencrypt/letsencrypt.log
    Enter email address (used for urgent renewal and security notices)
     (Enter 'c' to cancel): your-email@example.com
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    Please read the Terms of Service at
    https://letsencrypt.org/documents/LE-SA-v1.3-September-21-2022.pdf. You must
    agree in order to register with the ACME server.
    - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
    (Y)es/(N)o: Y
    [...]
    Successfully received certificate.
    Certificate is saved at: /etc/letsencrypt/live/test.experced.ru/fullchain.pem
    Key is saved at:         /etc/letsencrypt/live/test.experced.ru/privkey.pem
    [...]
    Congratulations! You have successfully enabled HTTPS on https://test.experced.ru
    ```

7.  Снова откройте конфигурационный файл для редактирования.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    **Удалите всё содержимое** и вставьте финальную конфигурацию (замените `test.experced.ru` на ваш домен):
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
    Сохраните файл и выйдите.

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
    *   **Ожидаемый вывод:** Команда не выводит ничего в случае успеха.

**Шаг 4: Установка и настройка панели 3X-UI**

1.  Выполните команду для установки панели.
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```
    *   **Ожидаемый вывод (сокращённый):** Скрипт задаст несколько вопросов, на которые нужно ответить `y`. В конце выведет данные для входа.
    ```
    Would you like to customize the Panel Port settings? (If not, a random port will be applied) [y/n]: y
    Please set up the panel port: 8080
    [...]
    Username and password updated successfully
    ---------------------------------------------
    This is a fresh installation, generating random login info for security concerns:
    Username: aBcDeFgH
    Password: xYz12345
    Access URL: http://SERVER_IP:8080/path/
    Start migrating database...
    Migration Done!
    ```
    **Обязательно сохраните имя пользователя и пароль!**

2.  Откройте URL из вывода команды в браузере и войдите в панель.

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

1.  Скопируйте ссылку на подключение из панели 3X-UI.
2.  Импортируйте её в клиент (например, Nekoray) и запустите тест.
3.  Откройте `speedtest.net` и убедитесь, что ваш IP-адрес соответствует серверу в Санкт-Петербурге.

---

### **Часть 2: Настройка второго сервера (Германия)**

Теперь все действия повторяются для второго сервера с доменом `test2.experced.ru`.

**Шаг 6: Подключение и обновление**

1.  Подключитесь ко второму серверу ("Франкфурт сервер").
2.  Обновите систему командами `apt update` и `apt upgrade -y`. Процесс и вывод будут аналогичны первому серверу.

**Шаг 7: Настройка Nginx и SSL**

1.  Установите Nginx и Certbot.
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
2.  Повторите шаги 3.1 - 3.7 из первой части, но **используйте домен `test2.experced.ru`**. Процесс получения сертификата будет идентичным.
    ```bash
    rm /etc/nginx/sites-enabled/default
    mkdir /var/www/html/site
    # (Перенесите index.html)
    nano /etc/nginx/sites-available/sni.conf
    # (Вставьте конфигурацию для HTTP-редиректа с server_name test2.experced.ru)
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    certbot --nginx -d test2.experced.ru
    ```
3.  Отредактируйте `nano /etc/nginx/sites-available/sni.conf`, заменив всё содержимое на финальную конфигурацию для **второго** домена.
4.  Проверьте и перезапустите Nginx: `nginx -t` и `systemctl restart nginx`.

**Шаг 8: Установка и настройка 3X-UI**

1.  Установите панель 3X-UI командой из шага 4.1.
2.  Запустите меню управления панелью:
    ```bash
    x-ui
    ```
    *   **Ожидаемый вывод:** Появится текстовое меню с опциями.
    ```
     3X-UI Panel Management Script
     ---------------------------------------------
     1. Install
     2. Update
     [...]
     6. Reset Username & Password
    ```
3.  Выберите опцию `6`, чтобы сбросить логин и пароль.
4.  Войдите в веб-панель.
5.  Перейдите в **Panel Settings** -> **Certificates** и укажите пути к сертификатам:
    *   **Public Key Path:** `/etc/letsencrypt/live/test2.experced.ru/fullchain.pem`
    *   **Private Key Path:** `/etc/letsencrypt/live/test2.experced.ru/privkey.pem`
6.  Перейдите в **Subscription**, включите сервис, укажите **Listen Port:** `8443` и **URI Path:** `/subsbca`.
7.  Нажмите **Save** и **Restart Panel**.
8.  Перезайдите в панель, используя HTTPS: `https://test2.experced.ru:8080`.

**Шаг 9: Настройка входящих и исходящих соединений**

1.  Создайте **Inbound** (входящее соединение) аналогично первому серверу.
2.  Перейдите в **Outbounds**, нажмите **Add Outbound**, выберите вкладку **JSON** и вставьте JSON-конфигурацию, скопированную с **первого** сервера.
3.  Создайте **Routing Rule** для маршрутизации трафика с вашего входящего соединения на созданное исходящее.
4.  Сохраните и перезапустите панель.

**Шаг 10: Финальная проверка**

1.  Скопируйте URL подписки со второго сервера и обновите профили в клиенте.
2.  Удалите старый профиль и протестируйте новый.
3.  Проверьте свой IP-адрес. Он должен соответствовать IP-адресу сервера в Германии, подтверждая, что прокси-цепочка работает.
