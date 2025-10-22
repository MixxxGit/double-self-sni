Конечно, вот пошаговое руководство, составленное на основе видео. В нём точно воспроизведены все действия и команды, а также указано, к какому из двух серверов (в Санкт-Петербурге или Германии) они относятся.

### Общая цель

Настроить два сервера с панелью управления прокси-соединениями 3X-UI, защитить их с помощью SSL-сертификатов от Let's Encrypt и настроить проксирование трафика.

---

### **Часть 1: Настройка первого сервера (Санкт-Петербург)**

На этом этапе все действия производятся на первом сервере с тестовым доменом `test.experced.ru`.

**Шаг 1: Подключение к серверу и обновление системы**

1.  В SSH-клиенте Termius выберите сохранённый хост (в видео он называется "СПб-Сервер") и подключитесь к нему.
2.  Обновите список пакетов и саму систему, выполнив следующие команды:
    ```bash
    apt update
    ```
    ```bash
    apt upgrade -y
    ```

**Шаг 2: Проверка IP-адреса и установка Nginx и Certbot**

1.  Узнайте IP-адрес сервера. (В видео для этого использовалась команда `nslookup`, но можно использовать и `ip a`).
    ```bash
    nslookup test.experced.ru
    ```
    ```bash
    ip a
    ```
2.  Установите веб-сервер Nginx, Certbot для получения SSL-сертификатов и плагин для Nginx.
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```

**Шаг 3: Настройка Nginx и получение SSL-сертификата**

1.  Удалите стандартный конфигурационный файл Nginx, чтобы он не мешал.
    ```bash
    rm /etc/nginx/sites-enabled/default
    ```
2.  Создайте директорию для файлов вашего сайта.
    ```bash
    mkdir /var/www/html/site
    ```
3.  Создайте и разместите в этой директории файл `index.html`. В видео для этого используется SFTP-клиент в Termius:
    *   В левой панели (локальный компьютер) найдите заранее подготовленный `index.html`.
    *   Переименуйте его, если нужно.
    *   В правой панели (сервер) перейдите в директорию `/var/www/html/site`.
    *   Перетащите файл `index.html` с левой панели на правую.

4.  Создайте новый конфигурационный файл для вашего сайта с помощью текстового редактора `nano`.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
5.  Вставьте в редактор `nano` следующую конфигурацию. Она будет перенаправлять все HTTP-запросы на HTTPS.
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
    *   Нажмите `Ctrl+X`, затем `Y` и `Enter`, чтобы сохранить файл и выйти.

6.  Создайте символическую ссылку, чтобы активировать вашу конфигурацию.
    ```bash
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    ```
7.  Запустите Certbot для получения SSL-сертификата для вашего домена.
    ```bash
    certbot --nginx -d test.experced.ru
    ```
    *   В процессе Certbot запросит ваш email, согласие с условиями использования и предложение подписаться на рассылку. Введите данные и подтвердите действия.

8.  Снова откройте конфигурационный файл Nginx для редактирования.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
9.  Certbot автоматически добавил секцию для HTTPS. Теперь нужно её отредактировать. **Удалите всё содержимое файла** и вставьте вместо него финальную конфигурацию. Эта конфигурация будет обрабатывать SSL и проксировать трафик на локальный порт.
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
    *   Сохраните файл и выйдите (`Ctrl+X`, `Y`, `Enter`).

10. Проверьте синтаксис конфигурации Nginx и перезапустите сервис.
    ```bash
    nginx -t
    ```
    ```bash
    systemctl restart nginx
    ```

**Шаг 4: Установка и настройка панели 3X-UI**

1.  Выполните команду для установки панели 3X-UI.
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```
2.  В процессе установки скрипт задаст несколько вопросов. Ответьте `y` на все. Порт для панели можно оставить по умолчанию или задать свой (в видео используется `8080`).
3.  После установки скрипт выведет имя пользователя, пароль и URL для доступа к панели. **Сохраните эти данные.**
4.  Откройте в браузере IP-адрес сервера с указанным портом (например, `http://193.71.22.37:8080`) и войдите в панель, используя полученные учётные данные.
5.  В панели 3X-UI перейдите в раздел **Inbounds** и нажмите **Add Inbound**.
6.  Настройте входящее соединение:
    *   **Port:** `443`
    *   **Security:** `reality`
    *   **Client:** Создайте нового клиента (email: `Test`, subscription: `Test`)
    *   **Flow:** `xtls-rprx-vision`
    *   **Target:** `127.0.0.1:8443`
    *   **SNI:** `test.experced.ru`
    *   Нажмите **Get New Keys** для генерации ключей.
    *   Нажмите **Create**.
7.  Перейдите в **Routing Rules** и добавьте новое правило:
    *   **Outbound Tag:** `Test`
    *   **Inbound Tags:** `inbound-443` (выберите из списка)
    *   Нажмите **Add Rule**.
8.  Сохраните и перезапустите панель, нажав **Save** и **Restart Panel**.

**Шаг 5: Проверка работы через клиент**

1.  На главной странице панели 3X-UI найдите созданное входящее соединение, нажмите на меню "три точки" и скопируйте ссылку (**Copy Link**).
2.  Импортируйте эту ссылку в ваш клиент (в видео используется Nekoray).
3.  Проверьте подключение, запустив тест в клиенте.
4.  Откройте сайт `speedtest.net` и запустите тест скорости, чтобы убедиться, что трафик идёт через сервер.

---

### **Часть 2: Настройка второго сервера (Германия)**

Теперь все действия повторяются для второго сервера с доменом `test2.experced.ru`.

**Шаг 6: Подключение к серверу и обновление**

1.  Подключитесь ко второму серверу (в видео "Франкфурт сервер").
2.  Обновите систему:
    ```bash
    apt update
    ```
    ```bash
    apt upgrade -y
    ```

**Шаг 7: Настройка Nginx и SSL для второго домена**

1.  Установите Nginx и Certbot, как и на первом сервере.
    ```bash
    sudo apt install nginx certbot python3-certbot-nginx -y
    ```
2.  Повторите шаги 3.1 - 3.7, но используйте домен `test2.experced.ru`.
    ```bash
    rm /etc/nginx/sites-enabled/default
    mkdir /var/www/html/site
    # (Перенесите index.html в /var/www/html/site)
    nano /etc/nginx/sites-available/sni.conf
    # (Вставьте конфигурацию для HTTP-редиректа с server_name test2.experced.ru)
    ln -s /etc/nginx/sites-available/sni.conf /etc/nginx/sites-enabled/
    certbot --nginx -d test2.experced.ru
    ```
3.  Отредактируйте конфигурационный файл Nginx, удалив всё и вставив финальную версию, но уже с путями и доменом для `test2.experced.ru`.
    ```bash
    nano /etc/nginx/sites-available/sni.conf
    ```
    Содержимое файла:
    ```nginx
    server {
        listen 127.0.0.1:8443 ssl http2 proxy_protocol;
        server_name test2.experced.ru;

        ssl_certificate /etc/letsencrypt/live/test2.experced.ru/fullchain.pem;
        ssl_certificate_key /etc/letsencrypt/live/test2.experced.ru/privkey.pem;

        # ... (остальные SSL-параметры, как на первом сервере)

        root /var/www/html/site;
        index index.html;

        location / {
            try_files $uri $uri/ =404;
        }
    }
    ```
4.  Проверьте конфигурацию и перезапустите Nginx.
    ```bash
    nginx -t
    ```
    ```bash
    systemctl restart nginx
    ```

**Шаг 8: Установка и настройка 3X-UI на втором сервере**

1.  Установите панель, как на первом сервере.
    ```bash
    bash <(curl -Ls https://raw.githubusercontent.com/MHSanaei/3x-ui/master/install.sh)
    ```
2.  После установки, зайдите в меню управления панелью, введя команду `x-ui`.
3.  Выберите опцию `6` (**Reset Username & Password**), чтобы сбросить и установить новые логин и пароль.
4.  Войдите в веб-панель.
5.  Перейдите в **Panel Settings** -> **Certificates**.
    *   В поле **Public Key Path** укажите путь к вашему сертификату: `/etc/letsencrypt/live/test2.experced.ru/fullchain.pem`
    *   В поле **Private Key Path** укажите путь к ключу: `/etc/letsencrypt/live/test2.experced.ru/privkey.pem`
6.  Перейдите во вкладку **Subscription**.
    *   Включите **Subscription Service**.
    *   **Listen Port:** `8443`
    *   **URI Path:** `/subsbca` (или любой другой путь)
7.  Нажмите **Save**, а затем **Restart Panel**, чтобы применить изменения.
8.  Перезайдите в панель, используя теперь доменное имя и HTTPS (например, `https://test2.experced.ru:8080`).

**Шаг 9: Настройка входящих и исходящих соединений (второй сервер)**

1.  Настройте входящее соединение (Inbound) аналогично первому серверу, используя порт `443`, `reality` и SNI `test2.experced.ru`.
2.  Создайте исходящее соединение (Outbound), импортировав JSON с конфигурацией первого сервера.
3.  Создайте правило в **Routing Rules**, которое будет направлять трафик с вашего входящего соединения на созданное исходящее (на первый сервер).
4.  Сохраните и перезапустите панель.

**Шаг 10: Финальная проверка**

1.  На втором сервере в разделе **Inbounds** скопируйте URL подписки (**Subscription URL**).
2.  В клиенте Nekoray обновите профили по этой ссылке. У вас появится новое подключение.
3.  Удалите старое подключение (к первому серверу напрямую) и протестируйте новое.
4.  Проверьте свой IP-адрес на сайте `check-host.net`. Он должен соответствовать IP-адресу сервера в Германии, что означает, что трафик успешно проксируется через оба сервера.

На этом настройка завершена.
