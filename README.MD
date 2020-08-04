### 1. Запустите сервер
У нас есть исходный код [сайта](https://github.com/su1tanb3k/systemd_tutorial/blob/master/index.html).

Теперь мы должны сделать так, чтобы в интернете появился наш сайт.

Мы загрузили этот `index.html` на сервер в корневую директорию, но сайт не работает. Дело в том, что сервер не запущен.

1. Запустите сервер с введя в терминале команду `python3 -m http.server 8000`

2. Проверьте, что по ip-адресу вашего сервера на порте 8000 появится сайт

    ![image](git_assets/landing.png)

3. Сайт запущен! Можно выходить из терминала. Просто закройте окошко терминала или введите `exit` и затем нажмите `Enter`.

4. Увидьте, что сайт перестал работать.

Основная причина - это ограниченность ресурсов сервера, он не может держать бесконечное количество активных подключений.

Поэтому он отключает все неактивные подключения.

Мы, конечно, можем сделать так, чтобы время наше соединение было активным "вечно", но это небезопасное решение.

Открытым подключением могут воспользоваться злоумышленники и натворить делов на сервере.

### 2. Создайте демона

К счастью, решение есть!

Если запустить нашу программу в специальном режиме демона, то она продолжит работать даже после закрытия терминала/отключения терминала от сервера.

Для этого есть специальная программа **SystemD**, которая встроена в ОС нашего сервера.

1. Перейдите в эту директорию: `/etc/systemd/system`

2. Создайте специальный файл-инструкцию с названием `my_http_server.service` и с таким содержанием:
 
     ```systemd
    [Service]
    ExecStart=python -m http.server 8000
    ```
    > Если коротко, то там написано: *"При запуске файла надо запустить команду python -m http.server 8000 "*

3. Запустите файл с помощью команды `systemctl start my_http_server`
    - ...и получите ошибку:

    ```bash
    # systemctl start my_http_server
    Failed to start my_http_server.service: Unit my_http_server.service has a bad unit file setting.
    See system logs and 'systemctl status my_http_server.service' for details.
    ```
    
    Вольный перевод: Неудалось запустить ваш файл my_http_server.service: в нём плохие инструкции. Наберите `systemctl status my_http_server.service` , чтобы узнать что случилось.

4. Наберите `systemctl status my_http_server`
    - ...и получите ответ:

    ![image](git_assets/status_bad_setting.png)
    
    Там написано:
    - cтрока `Active: inactive (dead)` - говорит о том, что наш файл, который мы создали неактивен/"мёртв"
    - красная строка `/etc/systemd/system/my_http_server.service:2: Executable "python" not found in path "/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"`
    
    Это значит, что SystemD в соответствии с инструкцией `ExecStart=python -m http.server 8000` искал `python` и не смог его найти.
    Действительно, если посмотреть, то конкретно на нашем сервере по такому адресу нет такой директории, но есть `python3` и `python3.8`.

5. Измените содержание файла на такое:

    ```systemd
    [Service]
    ExecStart=python3 -m http.server 8000
    ```

6. Повторите запуск с помощью команды из пункта №3: `systemctl start my_http_server`.

7. В этот раз терминал ничем не ответил. Проверьте состояние файла-инструкции с помощью команды из пункта №4: `systemctl status my_http_server`
    -ответ терминала:

    ![image](git_assets/status_active_running.png)
    
    В этот раз написано:
    - строка `●my_http_server.service`. Кружок возле названия нашего файла стал зелёным, а в прошлый раз был черный.
    - строка `Active: active (running)`. Слова `active (running)` означают "активный (запущен)". Как и кружок они стали зелеными, а в прошлый раз они были красными.
    
    >*Поняли, что зелёный цвет - это хороший знак.*

8. Пройдите в браузере по адресу `ip-сервера:8000`. Проверьте сайт. Успех!

9. А точно успех? Убедитесь в этом. Закройте терминал или наберите команду `exit` и затем `Enter`.

    ```bash
    # exit
    logout
    Connection to 84.38.180.108 closed.
    ```

10. Пройдите в браузере по адресу `ip-сервера:8000`.
    - Успех! Поздравляю!


### 3. Добавьте в автозапуск
Сервер иногда приходится перезапускать из-за обновления ПО и изменения настроек. Ещё хостинг провайдер периодически обновляет софт, чтобы закрыть свежие уязвимости, и самостоятельно перезапускает сервер.

- Перезагрузите сервер набрав в терминале `reboot now`

- Убедитесь, что сайт действительно не работает:

![image](git_assets/landing_off.png)

Сайт после перезагрузки сервера перестал работать. 

Это произошло потому что мы не дали инструкцию SystemD запускать программу во время включения сервера.

1. Измените файл `my_http_server.service`. Он должен стать таким:
    ```systemd
    [Service]
    ExecStart=/usr/bin/python3 -m http.server 8000
    
    [Install]
    WantedBy=multi-user.target
    ```

2. Введите команду `systemctl enable my_http_server`
    - получите ответ:
    ```systemd
    # systemctl enable my_http_server
    Created symlink /etc/systemd/system/multi-user.target.wants/my_http_server.service → /etc/systemd/system/my_http_server.service.
   ```

3. Перезагрузите сервер с помощью команды `reboot now`

5. Убедитесь, что сайт продолжает работать.

### 4. Автозапуск в случае сбоя

Поздравляю! Сайт продолжает работать даже если вы закроете терминал и прервете соединение с сервером или даже после перезагрузки сервера.

Однако, что если в `http.sever` или в самом `python-интерпретаторе` что-то поломается или просто произойдет какой-то сбой?

- Сымитируйте поломку программы. "Резко" остановите работу файла с помощью команды `systemctl kill my_http_server`
- Проверьте сайт. Убедитесь, что не работает.

Сервер всё еще работает, мы можем "гулять" по директориям внутри сервера, но сайт не работает.

Скажите SystemD, чтобы если вдруг файл перестанет работать, чтобы он перезапустился и так каждый раз.

1. Отредактируйте файл. Теперь он стал таким:

    ```systemd
    [Service]
    ExecStart=/usr/bin/python3 -m http.server 8000
    Restart=always
    
    [Install]
    WantedBy=multi-user.target
    ```

2. Запустите файл с помощью команды `systemctl start my_http_server`
    - прилетел следующий ответ:
    ```bash
    # systemctl start my_http_server
    Warning: The unit file, source configuration file or drop-ins of my_http_server.service changed on disk. Run 'systemctl daemon-reload' to reload units.
    ```
    Автоский перевод: *Внимание: ваш файл-инструкция был изменен. Запустите команду `systemctl daemon-reload`, чтобы перезагрузить ваш файл.*

    В общем, если вы что-то изменили в файле, то всегда запускайте эту команду `systemctl daemon-reload`.

    >Вы спросите почему мы её до этого не запускали? Ведь мы и до этого делали изменения в файле.
    Да, это правда, но тогда, если помните, мы перезагружали сервер (`reboot now`) и поэтому SystemD перезапускался и запускал уже обновленный файл.

3. Запускаем команду `systemctl daemon-reload`

4. Повторите запуск `systemctl start my_http_server`.

5. Резко прервите работу файла с помощью команды `systemctl kill my_http_server`

6. Убедитесь, что сайт работает.

Поздравляю! Теперь сайт работает и после закрытия терминала, и после перезагрузки, и даже если поломается программа в файле-инструкции.

### 5. Посмотреть лог обращений к серверу

Иногда нужно посмотреть, что происходит со всеми процессами на сервере.

- Запустите команду `journalctl`
    - Будет показана вся история с самого начала

![image](git_assets/journalctl.png)

- Запустите `journalctl -b`
    - Будут показаны только последние (самые свежие) записи
    

Иногда нужно посмотреть историю работы только по определенному файлу-инструкции.

- Запустите команду `journalctl -u my_http_server`.

![image](git_assets/journalctl_u_http_server.png)

- Запустите команду `journalctl -f -u my_http_server`.
    - Будете видеть историю работы по определенному файлу в режиме реального времени (на экране будут добавляться новые записи).

### Читать дальше

1. https://tproger.ru/translations/how-to-love-systemd/
2. https://www.digitalocean.com/community/tutorials/understanding-systemd-units-and-unit-files (Аналог на русском https://linux-notes.org/pishem-systemd-unit-fajl/)
3. https://www.digitalocean.com/community/tutorials/how-to-use-systemctl-to-manage-systemd-services-and-units
4. https://pikabu.ru/story/systemd_dlya_samyikh_malenkikh_chast_i_znakomstvo_4285483
5. [Документация SystemD](https://www.freedesktop.org/software/systemd/man/systemd.service.html)